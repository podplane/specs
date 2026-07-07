# Podplane Registry Push

## Goal

Podplane provides a cluster-hosted OCI registry for application images.

```sh
podplane push example-api:latest
# pushed: <registry-hostname>/apps/example-api:latest

podplane deploy web --name example-api --image <registry-hostname>/apps/example-api:latest
```

The registry bucket remains the single backing store, with two I/O paths:
- Read path: On each Kubernetes cluster Node, vmconfig configures a Zot Registry service, which only reads from the bucket.
- Write path: All writes to the bucket go through an in-cluster Zot Registry component with a Service and a Deployment.

## Existing contract

- vmconfig configures host-level Zot from `REGISTRY_*` user-data env vars and serves it only on loopback to containerd.
- containerd is configured with only `pull` and `resolve` capabilities.
- host-level Zot access control grants only `read` to `containerd.client`.
- AWS `cluster create` already creates two registry roles: `registry_read_only` for host-level Zot and `registry_read_write` for the in-cluster registry component.
- `deps download` already writes Zot-compatible object layout for mirrored dependency images under the registry bucket root.
- local clusters expose a fake S3 `registry` bucket backed by the local registry cache.

Do not add write credentials to vmconfig nodes.

## User experience

```sh
podplane push <local-image> [<remote-image>]
```

Rules:

- The source is a local Docker/containerd/OCI image reference resolvable by the CLI image library.
- If `<remote-image>` is omitted, push to `<registry-hostname>/apps/<source-repo-basename>:<source-tag>`.
- If `<remote-image>` omits a registry hostname, prefix the cluster registry hostname.
- User-pushed images must live under `apps/**`; reserve `mirror/**` for Podplane-managed cached upstream images.
- If the source tag is omitted, use `latest`.
- The command resolves the current kube context the same way `podplane deploy` does.
- The command prints exactly the image reference that workloads should use.
- If `zot-registry` is not installed or has no ready Service endpoints, fail before port-forwarding and suggest installing the component.

## Write path

Use Kubernetes port-forward for the first implementation.

```text
podplane push
  -> get/refresh the current Podplane OIDC ID token
  -> Kubernetes API port-forward
  -> Service/zot-registry in namespace platform-zot-registry
  -> push with direct Authorization: Bearer <id_token>
  -> Zot validates OIDC issuer/audience/groups and applies ACLs
  -> registry object-storage bucket
  -> node-local vmconfig Zot reads the same objects
  -> containerd pulls <registry-hostname>/<repo>:<tag>
```

Implementation requirement: use a registry library path that sends the token as a direct bearer/access token, not Docker's `IdentityToken` token-exchange flow.

Reasons:

- It removes the need for a dependency on an ingress controller, useful for example because:
  - developers may want a cluster only for background worker payloads, and do not wish to run/install Traefik
  - for those use cases, they must still be able to push images to the registry
- Kubernetes authorizes the port-forward, but registry authorization is Zot OIDC ACLs against the user's Podplane OIDC token user/group claims.
- It avoids exposing push on public ingress.
- It works for local, AWS, and future GCP clusters with the same CLI path.
- It keeps registry writes inside Kubernetes, where the write-capable bucket identity belongs.

Do not require public Traefik/Gateway ingress for `podplane push`.

## Optional Docker-compatible ingress

Standard `docker push <registry-hostname>/apps/<repo>:<tag>` must be supported if the registry ingress is explicitly enabled. Ingress is disabled by default.

When enabled, `zot-registry` owns custom routing instead of using the upstream Zot chart ingress:

```text
https://<registry-hostname>/v2/...  -> Zot
https://<registry-hostname>/token   -> registry token service
```

Docker flow:

```text
docker push <registry-hostname>/apps/example-api:latest
  -> Zot challenges with WWW-Authenticate: Bearer realm="https://<registry-hostname>/token"
  -> Docker credential helper supplies the Podplane OIDC ID token as IdentityToken
  -> Docker POSTs grant_type=refresh_token&refresh_token=<id_token>&scope=... to /token
  -> token service validates the OIDC token and returns {"access_token":"<id_token>"}
  -> Docker retries Zot with Authorization: Bearer <id_token>
  -> Zot validates OIDC and applies ACLs
```

The token service may be implemented by the Podplane operator or a smaller dedicated service. It is not in the blob data path and needs no bucket credentials.

`podplane login` may configure Docker automatically when Docker is installed, the registry hostname is known, and ingress/token service is enabled. Docker requires an executable named `docker-credential-podplane`; support this by letting the installed `podplane` binary detect `argv[0] == "docker-credential-podplane"` and dispatch to the same implementation as `podplane hooks docker-credentials`, so installation can create a symlink instead of shipping a second binary. Do not configure Docker for port-forward-only clusters; stock Docker cannot use Zot OIDC bearer auth without the token-service adapter.

## OIDC and ACLs

Configure Zot OIDC bearer auth against the same issuer and audience Podplane already uses for Kubernetes API clients:

- issuer: `cluster.oidc.issuer_url`;
- audience: `cluster.oidc.client_id`, defaulting to `cluster.id`;
- username claim: `cluster.oidc.username_claim`, defaulting to `email`;
- groups claim: `cluster.oidc.groups_claim`, defaulting to `groups`.

Default ACL groups:

```text
podplane:registry:admin -> ** read/create/update/delete
podplane:registry:edit  -> ** read; apps/** create
podplane:registry:view  -> ** read
podplane:admins         -> ** read/create/update/delete
podplane:editors        -> ** read; apps/** create
podplane:operators      -> ** read; apps/** create
podplane:viewers        -> ** read
system:masters          -> ** read/create/update/delete
```

Only admin groups can write `mirror/**`. Edit groups can read all repositories and create user images under `apps/**`, but cannot update existing tags or write mirrored dependency repositories.

`system:masters` is the Kubernetes default super-user group. `podplane:admins`, `podplane:editors`, `podplane:operators`, and `podplane:viewers` match the platform RBAC groups. Zot cannot infer arbitrary Kubernetes `ClusterRoleBinding`s from a raw OIDC token; if that becomes required, the token service must mint scoped registry tokens after Kubernetes authorization checks.

Expose ACLs in `zot-registry` chart values so operators can replace or extend them through `platform-components` values.

## Registry config shape

Move the registry hostname to cluster-level config. Existing configs with `components.registry.mirror.hostname` should be migrated by hoisting that value to `cluster.registry.hostname`.

```json
{
  "cluster": {
    "registry": {
      "hostname": "registry.example.com",
      "ingress": { "enabled": false }
    }
  }
}
```

Use `cluster.registry.hostname` for vmconfig `REGISTRY_HOSTNAME`, `podplane push`, and optional Docker ingress. Keep `components.registry.mirror.enabled` as the dependency image rewrite switch. `components.registry.mirror.hostname` remains as an advanced shared-mirror override; when omitted, it defaults to `cluster.registry.hostname`. `components.registry.mirror.prefix` is an advanced path-prefix override; it defaults to `mirror`.

When passing values to the `platform-components` chart for an individual component, put them only under `platform.components.values.<component-name>`. For example, `zot-registry` chart values belong under `platform.components.values.zot-registry`.

Do not set values directly on child component charts from the cluster config, and do not put component-specific chart values on the `platform.components.apps.<component-name>` entry. App entries are only install metadata such as `enabled`, `namespace`, `dependsOn`, `core`, and `manageNamespace`.

Mirror prefix rules:

- Accept `mirror`, `/mirror/`, `my/sub/directory`, and `/my/sub/directory/`.
- Normalize by trimming leading/trailing `/`.
- Empty string or `/` means no prefix.
- Render mirrored refs as `<mirror-hostname>/<prefix>/<upstream-registry>/<repo>:<tag>`, or `<mirror-hostname>/<upstream-registry>/<repo>:<tag>` when the normalized prefix is empty.
- `podplane deps download` uses the default local layout `mirror/<upstream-registry>/<repo>` regardless of shared-mirror overrides.

## Registry component

Add a `zot-registry` component in `podplane/components`:

- Helm-only chart, reconciled by `platform-components` like other apps.
- Use the upstream Zot Helm chart as the implementation; `zot-registry` is only a thin Podplane wrapper for values, config, identity, and naming.
- Do not fork upstream Zot chart templates or hand-write equivalent Deployment/Service templates unless the upstream chart cannot express a required setting.
- Chart/app key: `zot-registry`; Flux release name: `platform-zot-registry`.
- Namespace: `platform-zot-registry`.
- Add `zot-registry` to `platform.components.apps` and to the bootstrap chart's addon list so `bootstrap.install: recommended` and `all` enable it. Adding only the app entry is insufficient because bootstrap now renders recommended/all overrides from `bootstrap.addons.apps`.
- Workload: Zot configured for push/pull against the existing registry bucket.
- Service: ClusterIP named `zot-registry`, selected by `podplane push` for port-forward.
- Dependencies: `platform-certs` for TLS, plus provider identity plumbing for object-store writes.
- Do not add ingress, HTTPRoute, ServiceMonitor, persistence PVCs, or CRDs for the initial version.
- No CRD chart unless a Zot operator is introduced; do not introduce one for the initial version.
- Later Docker-compatible ingress support may add optional custom routing for `/v2/...` and `/token`; keep it disabled by default and do not use the upstream Zot chart ingress for that routing.

The component receives:

- registry bucket name,
- provider region/endpoint/path-style settings,
- write-capable cloud identity (`registry_read_write` on AWS; the equivalent GCP service account when GCP cluster creation exists),
- registry hostname used by vmconfig/containerd,
- OIDC issuer/audience/claim settings,
- ACL values,
- optional ingress/token-service settings.

Object-storage credentials must not be stored in Helm values. Use cloud workload identity or mounted provider credentials. Static local test credentials are allowed only for `podplane local start` fake S3.

## Local clusters

`podplane local start --components recommended` (which is the same as `podplane local start`, since `recommended` is the default) installs `zot-registry`.

The in-cluster registry writes to the local fake S3 `registry` bucket. Node-local vmconfig Zot keeps reading that same bucket through `/s3/cache/`, so pushed local images are immediately pullable by pods as `<local-registry-hostname>/apps/<repo>:<tag>`.

Minimal local clusters do not install the component; `podplane push` reports that `zot-registry` is missing, suggests how to install it with the `podplane install` command, and exits early.

## Production clusters

AWS `cluster create` wires:

- vmconfig/node Zot: `REGISTRY_ASSUME_ROLE = registry_read_only`.
- `zot-registry`: `registry_read_write`.

GCP must follow the same split when `cluster create` supports GCP: node-local Zot gets read-only bucket access; `zot-registry` gets write access.

## Image layout

Pushed images use normal registry repositories under the same bucket root:

```text
<registry-hostname>/apps/example-api:latest
<registry-hostname>/apps/acme/example-api:latest
```

Dependency mirrors use an explicit `mirror/` prefix:

```text
<registry-hostname>/mirror/ghcr.io/podplane/hello:latest
<registry-hostname>/mirror/registry.k8s.io/pause:3.10.2
```

Do not make Zot a transparent pull-through cache.

## Implementation order

1. Update component image mirror plumbing so bootstrap-installed core charts and Flux-managed app charts render the same default `mirror/<upstream-registry>/<repo>` layout.
2. Add the `zot-registry` component, `platform-components` app entry, and bootstrap `recommended`/`all` addon entry.
3. Wire cluster/local `zot-registry` values through `platform.components.values.zot-registry` from cluster config and local start.
4. Implement `podplane push` with port-forward plus direct bearer registry copy.
5. In `podplane push`, verify that the `zot-registry` Service has ready endpoints before opening the port-forward.
6. Add optional ingress routing and token service for stock Docker compatibility.
7. Have `podplane login` configure Docker only when ingress/token-service support is enabled.
8. Document both flows: `podplane push` always; `docker push` only with registry ingress enabled.

## Implementation plan

### Repository: `github.com/podplane/components`

1. Add the `zot-registry` app entry and wrapper chart around upstream Zot.
2. Add `zot-registry` to the bootstrap chart `bootstrap.addons.apps` for `recommended` and `all`, so `podplane local start --components recommended` installs it.
3. Render Zot object-storage, OIDC, and ACL config from `platform.components.values.zot-registry`.
4. Extend `platform.components.registry.mirror` with `prefix`, defaulting to `mirror`.
5. Update `platform-components` mirror value rendering and comments so mirrored component images use `<mirror-hostname>/<mirror-prefix>/<upstream-registry>/<repo>:<tag>` with the default prefix `mirror`.
6. Update bootstrap chart/apply wiring so initially-installed core images (`cilium`, `coredns`, `fluxcd`) use the same mirror prefix layout as Flux-managed components.
7. Keep ingress/HTTPRoute, token service, ServiceMonitor, PVCs, and CRDs out of the initial chart.
8. Later add optional custom routing for `/v2/...` to Zot and `/token` to the token service.

### Repository: `github.com/podplane/podplane`

1. Add `cluster.registry.hostname` and `cluster.registry.ingress.enabled` to cluster config, schema, validation, summaries, local config, and Terraform generation.
2. Change registry cache population so `deps download` writes mirrored component/template/runtime images under `mirror/<upstream-registry>/<repo>`.
3. Change `MirroredImageRef` and deploy mirror overrides to emit `<mirror-hostname>/<mirror-prefix>/<upstream-registry>/<repo>:<tag>`, defaulting hostname from `cluster.registry.hostname` and prefix to `mirror`.
4. Keep `components.registry.mirror` as image-rewrite config, including advanced `hostname` and `prefix` overrides.
5. Update local-start example output, tests, and docs that currently show unprefixed mirror refs.
6. Implement `podplane push <local-image> [<remote-image>]` using port-forward and direct bearer-token registry auth.
7. Add `podplane hooks docker-credentials` and `argv[0] == "docker-credential-podplane"` dispatch.
8. Configure Docker during `podplane login` only when registry ingress/token service is enabled.
9. Update registry/vmconfig docs so mirror examples use `<registry-hostname>/mirror/...` and app examples use `<registry-hostname>/apps/...`.

### Repository: `github.com/podplane/operator`

1. Add the optional registry token endpoint for Docker ingress mode. Keep the endpoint stateless, out of the blob data path, and without bucket credentials.
2. Accept Docker refresh-token POSTs, validate the Podplane OIDC ID token, and return matching `token` and `access_token` plus `expires_in`.

### Repository: `github.com/podplane/vmconfig`

Keep node-local Zot read-only. No initial changes unless the new cluster registry config exposes a currently missing `REGISTRY_*` value.

## Security invariants

- Nodes never receive write-capable registry bucket credentials.
- Public ingress is disabled by default.
- `podplane push` does not require ingress.
- Stock Docker support requires the token-service adapter; do not claim Docker compatibility without it.
- Zot authorization is based on OIDC claims and ACLs, with operator-overridable policy.
- The only bucket writer is the in-cluster registry component.
- Existing mirrored dependency image layout remains compatible with vmconfig Zot.
