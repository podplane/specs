# Source Gateway

> **STATUS**: Ready for implementation

Source Gateway is a new sub-project of Podplane designed to run on any Kubernetes cluster:

- Repository: `github.com/sourcegateway/sourcegateway`
- Website: `https://sourcegateway.dev`

## Goals

- Cache and serve Git clone/fetch, and mediate restricted pushes.
- Provide a small S3-compatible API backed by AWS S3 or GCS.
- Authenticate callers as Kubernetes ServiceAccounts.
- Keep upstream credentials out of calling workloads.
- Run on any Kubernetes cluster.
- Interoperability with Nono, without requiring it.

## Non-goals

- General-purpose Git hosting or S3 compatibility.
- Branch-level Git read confidentiality.
- Human or Pod identity as an initial feature.
- Offline writes, write-back caching, or cross-cluster identity federation.
- Requiring Nono or any particular workload controller.

## Deployment requirements

Secrets Store CSI Driver must mount all static secret material. Source Gateway
requires `<credentialsDirectory>/encryption-key`, containing the standard
base64 encoding of 32 random bytes, and fails startup if it is absent or
invalid. `encryption-key` is reserved and cannot be a source credential name.
The initial release reserves this key for encrypted OAuth token storage so a future
upgrade needs no new secret mount.

## Architecture

The initial deployment is one controller/data-plane replica with one persistent
volume. Git mirrors and object-cache data are reconstructable; upstreams remain
authoritative.

```text
Workload -- ServiceAccount token --> Source Gateway
                                      |-- Git mirror --> GitHub
                                      `-- object cache --> S3/GCS

Optional: tool --> Nono proxy --> Source Gateway
```

The controller validates resources, compiles CEL policies, reconciles mirrors,
and atomically publishes immutable request-handler snapshots.

## Authentication

### Downstream authentication

Every workload request uses:

```http
Authorization: Bearer <projected-service-account-token>
```

The token must have the configured `tokenAudience`, which defaults to
`sourcegateway.dev`. This scopes the credential, not the ServiceAccount: the
same Pod may use its normal Kubernetes API token or other projected tokens for
other audiences.

Source Gateway submits an `authentication.k8s.io/v1 TokenReview` with that exact
audience. It accepts only an authenticated response whose:

- returned audiences contain the configured audience;
- username is `system:serviceaccount:<namespace>:<name>`;
- groups contain the matching ServiceAccount groups; and
- UID is nonempty.

Only the returned namespace, name, and UID become authorization inputs. JWT
claims, HTTP identity headers, and Pod metadata do not.

Positive TokenReview results may be cached by token SHA-256 for at most 30
seconds and never beyond token expiry. Invalid credentials return `401`;
TokenReview failure returns `503`.

### Optional Nono integration

Nono is not required. Users may use its credential proxy to keep the real
ServiceAccount token outside a sandboxed tool:

1. Mount the projected token into the trusted Nono supervisor.
2. Deny the tool access to that file and direct network egress.
3. Give the tool only Nono's phantom credential and local proxy.
4. Have Nono inject the real token only on approved Source Gateway routes.

Nono must reload kubelet-rotated tokens before expiry.

### Upstream authentication

Cluster operators configure credentials; source resources only select one by
name. The runtime config is non-secret JSONC mounted from a ConfigMap and passed
as `sourcegateway --config=/etc/sourcegateway/config.jsonc`:

```jsonc
{
  "tokenAudience": "sourcegateway.dev",
  "objectCacheCapacity": "20Gi",
  "credentialsDirectory": "/var/run/sourcegateway/secrets",
  "githubUserAnnotation": "sourcegateway.dev/github-user",
  "credentials": {
    "gitRepository": {
      "application-github": {
        "kind": "githubApp",
        "namespaces": ["agents"]
      }
    },
    "objectStorageBucket": {
      "artifacts": {
        "kind": "awsStatic",
        "namespaces": ["agents", "builds"]
      },
      "cluster-aws": {
        "kind": "awsAmbient",
        "namespaces": ["agents"]
      }
    }
  }
}
```

Each resource selects a credential from its matching group. `namespaces`
restricts which resource namespaces may select it, including ambient
credentials. `["*"]` explicitly allows all namespaces; an empty list is invalid.
Targets remain solely in the resources, while upstream permissions provide the
final access boundary.

Secret files are mounted read-only through Secrets Store CSI Driver at
`<credentialsDirectory>/<credential-name>`. The default directory is
`/var/run/sourcegateway/secrets`; operators may change it to match an existing
mount convention. Runtime config never lives there. Multiple
`SecretProviderClass` mounts are supported. Ambient credentials require no
files. Source Gateway rejects escaping symlinks and files accessible to group or
others, and reloads files after CSI rotation.

Chart values add a `secretProviderClass` to the encryption key and each
file-backed credential. The chart renders the JSONC ConfigMap and CSI mounts,
but omits these deployment-only fields from runtime config. It rolls the
Deployment when config or mounts change. JSONC is converted with
`github.com/tailscale/hujson`, then decoded strictly with Go's JSON library.

Git upstreams initially support HTTPS GitHub.com and GitHub Enterprise Server:

- `githubApp`: `app-id`, `installation-id`, and `private-key.pem`. Source
  Gateway mints and refreshes installation tokens.
- `githubUser`: `username` and `token`. Fine-grained personal access tokens
  are preferred.

SSH and non-GitHub Git servers are out of scope.

AWS S3 authentication supports:

- `awsAmbient`, using the Pod's AWS credential chain, including kube2iam; or
- `awsStatic`, containing `access-key-id`, `secret-access-key`, and optional
  `session-token` files.

GCS authentication supports:

- `gcsAmbient`, using Application Default Credentials; or
- `gcsServiceAccount`, containing `service-account.json`.

## Kubernetes API

The API group is `sourcegateway.dev/v1alpha1`. Resources are namespaced for
ownership and identity, but their CEL policies may authorize ServiceAccounts
from any namespace.

Use Kubebuilder and `controller-runtime`. `controller-gen` generates structural
CRDs, status subresources, deepcopy code, and least-privilege RBAC from Go types;
the Helm chart packages those files. Do not add admission webhooks for
`v1alpha1`. Use `metav1.Duration` for durations, `resource.Quantity` for
capacities, and merge authorization rules by unique `name`.

Each status has `observedGeneration` and `[]metav1.Condition` entries for
`PolicyReady`, `CredentialsReady`, `UpstreamReady`, and `Ready`. `Ready=True`
only after the controller starts serving the current generation. Write status
only when its value changes; expose refresh times and cache freshness as metrics.

### `GitRepository`

```yaml
apiVersion: sourcegateway.dev/v1alpha1
kind: GitRepository
metadata:
  namespace: agents
  name: application
spec:
  upstream:
    url: https://github.com/acme/application.git
    credential: application-github
  mirror:
    refreshInterval: 1m
    maxStaleness: 10m
  authorization:
    rules:
      - name: read
        operations: [git.read]
        allow: sa.namespace == "agents"
```

Status includes conditions, `policyRevision`, and `path`:

```text
/git/<namespace>/<name>.git
```

### `ObjectStorageBucket`

One resource maps one logical bucket to one backend bucket:

```yaml
apiVersion: sourcegateway.dev/v1alpha1
kind: ObjectStorageBucket
metadata:
  namespace: agents
  name: artifacts
spec:
  backend:
    provider: s3
    endpoint: https://s3.us-east-1.amazonaws.com
    region: us-east-1
    bucket: acme-agent-artifacts
    credential: artifacts
  cache:
    capacity: 20Gi
    ttl: 15m
  authorization:
    rules:
      - name: own-prefix
        operations: [object.list, object.read, object.write, object.delete]
        allow: request.key.startsWith("workspaces/" + sa.name + "/")
```

S3 `region` is required and passed only to the upstream client.
`endpoint` is optional and defaults to the provider endpoint; explicit endpoints
must use HTTPS except on loopback. GCS omits `region`.

Status includes conditions, `policyRevision`, and `bucketName`, generated as:

```text
<first 24 characters of namespace>-<first 24 characters of name>-<hash>
```

Periods become hyphens before truncation. `hash` is the first 12 characters of
the lowercase, unpadded Base32 SHA-256 of `<namespace>/<name>`. The result is
stable, at most 62 characters, and S3-compatible. Consumers use it with the
deployment's Service or Ingress URL and path-style addressing. `ListBuckets`
checks `object.list-buckets` on each ready resource and returns its authorized
`bucketName`. `CopyObject` is limited to one logical bucket.

The controller recomputes `bucketName` and maps it to the resource in memory;
status is output only. A collision makes all affected resources unready.

Invalid policy immediately removes a source from serving. Both kinds use the
`sourcegateway.dev/runtime-cleanup` finalizer. On deletion it removes routes,
drains requests, and deletes local data, but never upstream data.

## CEL authorization

Rules are additive. Any matching rule returning `true` allows; absence of a
match or any compilation, evaluation, timeout, or cost failure denies.

Operations are:

```text
git.read                 object.list-buckets
git.push                 object.head-bucket
                         object.list
                         object.read
                         object.write
                         object.delete
```

The versioned, strongly typed environment contains only:

```text
sa.namespace         string
sa.name              string
sa.uid               string
request.operation    string
request.branch       string
request.create       bool
request.delete       bool
request.force        bool
request.bucket       string
request.key          string
request.size         int
```

Each resource allows at most 32 rules. Each expression allows 4 KiB of text,
500 AST nodes, 1 KiB of regex text, and 10,000 estimated or runtime CEL cost
units. Exceeding a limit sets `PolicyReady=False`. Permission to change source
resources is security-sensitive because it changes policy and upstream access.

## Git protocol

Source Gateway implements smart HTTP discovery, `git-upload-pack`, and
`git-receive-pack`. Dumb HTTP, cookies, and SSH are unsupported.

Discovery and upload-pack each authorize `git.read`. Reads are repository-wide.
A mirror older than `maxStaleness` returns `503`; refreshes never expose partial
state.

Pushes accept only `refs/heads/*`. Each ref command authorizes `git.push`; one
denial rejects the whole push before upstream access.

For each command, CEL receives the branch name and whether it creates, deletes,
or force-updates the branch. Non-commit targets, malformed commands, missing
objects, and stale old object IDs reject the push.

Objects are unpacked into quarantine. Source Gateway validates the complete
push before sending it to GitHub. Serving refs change only after GitHub accepts.
If local update then fails, the client still receives success, reads stop, and
an immediate refresh repairs the mirror.

## Future: GitHub user impersonation

The initial release uses the configured upstream credential. A future optional
mode may instead use a GitHub App user access token selected for each request.

The JSONC setting `githubUserAnnotation` names a qualified Pod annotation and
defaults to `sourcegateway.dev/github-user`. The Pod annotation value is the
requested GitHub login.

An OAuth-enabled GitHub App credential adds CSI-mounted `client-id` and
`client-secret` files for the web authorization and refresh flows.

This mode requires a Pod-bound projected token. TokenReview supplies the Pod
name and UID; Source Gateway verifies that Pod and reads the configured
annotation. OAuth associates the GitHub App credential, ServiceAccount UID, and
stable GitHub user ID. A valid association is reusable by later Pods using that
ServiceAccount, while another ServiceAccount requires OAuth authorization. A
request without a valid association receives an authorization challenge.

Anyone allowed to launch a Pod as that ServiceAccount and set the annotation
may select its associated users. Source Gateway audits the ServiceAccount, Pod
UID, requested and resolved GitHub user, repository, and operation. Kubernetes
audit logs may separately identify who created a workload or SandboxClaim.

OAuth adds `github.id` and `github.login` as CEL inputs so each `GitRepository`
can limit which resolved users may act. The GitHub numeric ID is authoritative;
the login is display and selection data.

Source Gateway stores each OAuth grant in its namespace as a separate Kubernetes
Secret containing only an AES-256-GCM ciphertext envelope. The CSI-mounted
encryption key protects access and refresh tokens; each write uses a random nonce
and binds the ciphertext to the Secret, GitHub App credential, and GitHub user
ID. Refresh-token rotation updates the Secret atomically. RBAC permits only the
Source Gateway ServiceAccount to access these Secrets.

## S3-compatible protocol

Clients use anonymous credentials, path-style addressing, and the downstream
bearer header. SigV4 and presigned URLs are unsupported.

| Operation | Authorization |
| --- | --- |
| `ListBuckets` | `object.list-buckets` |
| `HeadBucket` | `object.head-bucket` |
| `ListObjectsV2` | `object.list` on the prefix |
| `HeadObject`, `GetObject` | `object.read` |
| `PutObject` | `object.write` |
| `DeleteObject`, `DeleteObjects` | `object.delete` per key |
| `CopyObject` | source read and destination write |

Initial limits:

- 1,000 list or multi-delete entries;
- one byte range;
- 15 MiB writes and copies;
- required `Content-Length` and precomputed checksums;
- no chunking, trailers, or `Expect: 100-continue`; and
- ASCII keys of 1–1,024 bytes matching `[A-Za-z0-9][A-Za-z0-9._/-]*`, excluding
  empty, `.` and `..` segments.

Multipart, multiple ranges, ACLs, versioning, Object Lock, browser uploads,
S3 Select, notifications, virtual-host buckets, and bucket mutation are out of
scope.

ETags, checksums, and generations are opaque. Preconditions use provider-native
atomic conditions or fail without writing.

Authorization precedes cache access. Data becomes visible only after a complete
validated read or committed write. Failed transfers leave no partial entry.
Writes acknowledge only after backend commit. Lists always query the provider.

Deployment setting `objectCacheCapacity` sets the global object-cache limit.
`cache.capacity` is a per-resource maximum, not reserved space. Evict the least
recently used objects to enforce both limits and keep 10% of the volume free.
Return `507` before changing upstream data when space is unavailable.

## Security and operations

Allow 64 active and 128 waiting requests. Extra requests return `503` with
`Retry-After`; there is no separate rate limit. Timeouts are 10 seconds for
headers, 30 seconds for provider API calls, 2 minutes for object transfers, and
10 minutes for Git transfers. Return `504` only if nothing committed.

- TLS protects every non-loopback hop.
- The deployment mounts its serving certificate and key through CSI;
  cert-manager is not required.
- The chart provides an ingress NetworkPolicy. Operators manage provider egress
  because standard NetworkPolicy cannot match DNS names.
- Upstream TLS verification is mandatory; cross-origin redirects are rejected.
- Policy denial returns `403`; unavailable providers return `503` only when no
  operation committed.
- Credentials, provider topology, URLs, branches, keys, and content are redacted
  from status, errors, logs, metrics, and audit events.
- Audits record ServiceAccount, source, operation, policy revision, decision,
  reason, and matched rules.
- Startup validates mirrors and discards incomplete cache entries. Volume loss
  rebuilds local state without replaying writes.

Readiness requires Kubernetes API access, serving TLS, writable storage, and a
published runtime snapshot. Individual source failures do not disable healthy
sources.

## Rollout

Deploy read-only canaries first, then object writes, then Git pushes on
disposable branches. Rollback points clients directly upstream after confirming
all acknowledged writes exist there.

## Implementation and verification

`github.com/sourcegateway/sourcegateway` owns the binary, API, Helm chart,
provider adapters, documentation, and tests.

Pin tools and dependencies in the repository, and document tested versions with
each release.

Implementation order:

1. Kubebuilder scaffold, CRDs, controller, TokenReview, CEL, and audit.
2. CSI credential loading and GitHub authentication.
3. Git mirrors, smart HTTP, quarantine, and pushes.
4. S3 HTTP, cache, S3/GCS adapters, and provider authentication.
5. Helm deployment, TLS, RBAC, NetworkPolicies, metrics, and recovery.

### Offline development environment

`Procfile.dev` is the single process definition for interactive development and
end-to-end tests. Overmind runs:

- an `envtest` kube-apiserver and etcd with the CRDs installed;
- `weed mini` as the S3 backend;
- a local HTTPS server wrapping Git's `git http-backend`; and
- Source Gateway.

After pinned dependencies are installed, the stack requires neither Docker nor
external network access. `make dev` runs it interactively. `make e2e` creates
temporary state, certificates, and an isolated Overmind socket; starts Overmind
in daemon mode; waits for readiness; runs checks; and always stops Overmind,
printing logs on failure.

The end-to-end test obtains a ServiceAccount token through `TokenRequest`, then
verifies Git clone, fetch, allowed push, and denied push, plus S3 put, get, list,
delete, and CEL denial against SeaweedFS using rclone and an AWS SDK. It excludes
live GitHub authentication, GCS, CSI, PVC, and NetworkPolicy behavior.

Other tests cover CRDs and status, finalizers, credentials, tokens, CEL limits,
redaction, cache recovery, and provider failures. Completion requires all tests,
the offline end-to-end test, and Helm lint and template validation to pass.

Direct downstream authentication must work without Nono. Separate optional-Nono
tests verify confinement, proxy-only access, and token rotation.
