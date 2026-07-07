# Podplane Secrets: Spec and Implementation Plan

## Goals

Podplane should provide a unified developer experience for managing application secrets without requiring developers to have direct access to AWS, GCP, OpenBao, or other infrastructure-layer secret backends.

The target experience is:

```sh
podplane secret create <secret-key> --for <name>
podplane secret create github-private-key --for aok-source-controller
podplane secret update github-private-key --for aok-source-controller
podplane secret list --for aok-source-controller
podplane secret delete github-private-key --for aok-source-controller
podplane secret delete --for aok-source-controller --all
```

Secret values should not be passed as positional shell arguments. By default, create/update should prompt for the value; they should also support reading the value from stdin for automation. Examples:

```
# prompts for value:
podplane secret create github-private-key --for aok-source-controller

# reads value from stdin:
cat github-private-key.pem | podplane secret create github-private-key --for aok-source-controller --stdin
```

## Working example

Use one concrete example throughout this spec:

```text
Workload namespace:         platform-aok
Podplane app name:          aok-source-controller
Workload deployment:        aok-operator
SecretProviderBinding name: aok-source-controller
SecretProviderClass name:   aok-source-controller
Secret key name:            github-private-key
Mounted file:               github-private-key
Cluster ID:                 prod-cluster
Cluster secrets prefix:     prod-cluster
Example provider name:      aws-secrets-manager
Backend path:               /prod-cluster/platform-aok/aok-source-controller/github-private-key
```

The deployment `platform-aok/aok-operator` needs a GitHub private key mounted as a file. A developer writes the value through Podplane:

```sh
cat github-private-key.pem | podplane secret create -n platform-aok github-private-key --for aok-source-controller --provider aws-secrets-manager --stdin
podplane secret list -n platform-aok --for aok-source-controller --provider aws-secrets-manager
```

The create command creates a secret in the specified backend provider under `/prod-cluster/platform-aok/aok-source-controller/github-private-key`.

The workload reads the value through Secrets Store CSI using `SecretProviderClass/aok-source-controller`.

Developers deploy their apps using Podplane templates, which should create a `SecretProviderBinding/aok-source-controller`; the Podplane operator reconciles that binding into the underlying `SecretProviderClass` with provider-specific backend paths under `/prod-cluster/platform-aok/aok-source-controller/`.

Example `SecretProviderClass` for AWS Secrets Manager:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  namespace: platform-aok
  name: aok-source-controller
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "/prod-cluster/platform-aok/aok-source-controller/github-private-key"
        objectType: "secretsmanager"
        objectAlias: "github-private-key"
```

Example workload mount:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: platform-aok
  name: aok-operator
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: aok-operator
  template:
    metadata:
      labels:
        app.kubernetes.io/name: aok-operator
    spec:
      serviceAccountName: aok-source-controller
      containers:
        - name: aok-operator
          image: ghcr.io/example/aok-operator:latest
          volumeMounts:
            - name: aok-source-controller-secrets
              mountPath: /var/run/aok/secrets
              readOnly: true
      volumes:
        - name: aok-source-controller-secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: aok-source-controller
```

The container sees the mounted file at `/var/run/aok/secrets/github-private-key`. No Kubernetes `Secret` object is created unless CSI secret sync is explicitly configured, which is not recommended.

Core requirements:

- Developers authenticate with the Podplane/Kubernetes cluster, not cloud IAM.
- Production secret values are never stored in Kubernetes `Secret` objects.
- Production secret values are never stored in CRDs, ConfigMaps, or other etcd-backed Podplane metadata.
- Production secret values are never logged.
- The Podplane operator is write-only with respect to secret values for developer workflows: it writes values supplied by users but does not read existing secret values back from providers.
- Kubernetes RBAC is the authorization surface for secret operations like put/list/delete.
- Secret delivery to workloads uses Secrets Store CSI Driver, with optional CSI secret sync for users who explicitly need native Kubernetes Secrets or env vars.
- Local clusters use the same API flow and provider adapter behavior as production, rather than the CLI or operator using local-only secret-store special cases.
- The Podplane operator must support the same external secret-store provider families used by the Secrets Store CSI providers Podplane supports: Vault, OpenBao, AWS Secrets Manager and AWS Parameter Store, and GCP Secret Manager.
- The system must be lightweight enough for local/single-VM clusters with 2-4GB RAM.

## Non-goals

- Do not make Kubernetes Secrets the default production storage or delivery model.
- Do not use External Secrets Operator or Sealed Secrets which store plaintext secrets in etcd.
- Do not require application developers to have AWS/GCP/OpenBao credentials.
- Do not build a custom Secrets Store CSI provider.
- Do not support duplicate/copy/export/reveal workflows that require the Podplane operator to read existing secret values from backend providers.
- Do not rely on Kyverno/Gatekeeper as required dependencies to make secrets secure.
- Do not create a separate long-running pod/container per small Podplane policy or API feature.
- Do not add a special local-cluster provider path inside `podplane-operator`; local development should use a normal Vault/OpenBao-compatible endpoint.

## Architecture

Podplane Secrets has three cooperating parts:

1. A Kubernetes aggregated API served by the Podplane operator.
2. A namespaced `SecretProviderBinding` CRD reconciled by the Podplane operator into owned Secrets Store CSI `SecretProviderClass` resources.
3. Backend adapters for Vault, OpenBao, AWS Secrets Manager, AWS Parameter Store, and GCP Secret Manager.

```text
Developer CLI
  podplane secret create --namespace=platform-aok github-private-key --for aok-source-controller
        |
        | Kubernetes API request, authorized by native RBAC
        v
kube-apiserver
        |
        | APIService proxy
        v
platform-podplane-operator/platform-podplane-operator
  aggregated API: secrets-api.podplane.dev
        |
        | writes value directly to configured external backend
        | key example: <cluster-secrets-prefix>/platform-aok/aok-source-controller/github-private-key
        |
        v
AWS Secrets Manager / AWS Parameter Store / GCP Secret Manager / OpenBao / Vault


Workload Pod
        |
        | CSI volume references SecretProviderClass aok-source-controller
        v
SecretProviderClass aok-source-controller
  generated from SecretProviderBinding aok-source-controller
        |
        v
Secrets Store CSI Driver + provider
        |
        | reads Podplane-generated backend paths
        v
same external backend
```

The aggregated API is the secret value write/control path. `SecretProviderBinding` is the declarative read/mount configuration path. Secrets Store CSI Driver is the workload read/mount path.

The CLI should never write directly to a provider backend, including in local clusters. It should always call the Kubernetes API. The operator should then use the configured backend adapter for the cluster.

## Support for Multiple Secrets Providers

Clusters should support multiple configured secrets providers at the same time. This allows a cluster to start with a simple/default backend such as AWS Parameter Store, later adopt AWS Secrets Manager for high-quantity/large-payload/more complex secrets needs, or even migrate selected workloads to OpenBao without changing the developer-facing `podplane secret` workflow.

Cluster configuration should define:

- a map of named secrets providers
- the provider kind for each entry
- non-sensitive provider-specific routing/configuration fields
- one default provider

Example shape:

```jsonc
{
  "cluster": {
    "secrets": {
      "default_provider": "aws-parameter-store",
      "providers": {
        "aws-parameter-store": {
          "kind": "aws",
          "object_type": "ssmparameter",
          "region": "us-east-1"
        },
        "aws-secrets-manager": {
          "kind": "aws",
          "object_type": "secretsmanager",
          "region": "us-east-1"
        },
        "gcp-secret-manager": {
          "kind": "gcp",
          "project_id": "prod-project"
        },
        "openbao": {
          "kind": "openbao",
          "address": "https://bao.example.com",
          "mount_path": "secret"
        }
      }
    }
  }
}
```

Provider map keys are the provider names and should use the same validation rules as the cluster ID: lowercase DNS-label-like (a-z0-9-), max 32 chars, no consecutive hyphens, and no dots. Provider names are cluster-local identifiers, not cloud resource names.

The cluster summary cached by `podplane login` / `podplane local start` should include only a subset of the secrets provider fields:

```jsonc
{
  "secrets": {
    "default_provider": "aws-parameter-store",
    "providers": {
      "aws-parameter-store": { "kind": "aws" },
      "aws-secrets-manager": { "kind": "aws" },
      "gcp-secret-manager": { "kind": "gcp" },
      "openbao": { "kind": "openbao" }
    }
  }
}
```

Only provider names, `kind`, and `default_provider` are needed in the cached summary. Credentials and sensitive backend details must never be stored in cluster config or cluster summary. They should come from operator runtime credential configuration, mounted credential files, Kubernetes workload identity, cloud IAM, or an external secret manager bootstrap mechanism.

Use these terms precisely:

- Provider name: a cluster-local handle chosen by the platform team, e.g. `openbao` or `aws-secrets-manager`.
- Provider kind: the upstream Secrets Store CSI provider slug, e.g. `openbao`, `vault`, `aws`, or `gcp`. This must match the `SecretProviderClass.spec.provider` value to avoid ambiguous Podplane-only naming.
- `SecretProviderClass.spec.provider`: the upstream Secrets Store CSI provider plugin name. Podplane does not choose this string; it must match the installed CSI provider.

Initial provider kind mapping:

| Backend family | Podplane provider kind | `SecretProviderClass.spec.provider` |
| --- | --- | --- |
| AWS Secrets Manager | `aws` | `aws` |
| AWS Parameter Store | `aws` | `aws` |
| GCP Secret Manager | `gcp` | `gcp` |
| HashiCorp Vault | `vault` | `vault` |
| OpenBao | `openbao` | `openbao` |

This is a hard requirement. AWS uses the single CSI provider slug `aws` for both Secrets Manager and Parameter Store, so provider-specific configuration must choose the AWS `objectType` separately (`secretsmanager` or `ssmparameter`). For Vault and OpenBao, Podplane's developer write path is explicitly KV-v2, even though the CSI read path can support broader Vault/OpenBao secret engines. The operator adapter docs should document that write-path contract.

For GCP, Podplane must use the upstream Google Secret Manager Secrets Store CSI provider with `SecretProviderClass.spec.provider: gcp` and the normal Secrets Store CSI Driver `secrets-store.csi.k8s.io`. Podplane is its own Kubernetes distribution, not GKE, so it must not use the GKE managed add-on provider slug `gke` or the GKE-specific CSI driver `secrets-store-gke.csi.k8s.io`.

## Component and dependency wiring

Podplane should manage Secrets Store CSI Driver and provider installation through `podplane/components`, not through ad hoc manifests in the operator or templates. The platform components chart should include component entries for:

- `secrets-store-csi-driver-crds`
- `secrets-store-csi-driver`
- `secrets-store-csi-driver-provider-aws`
- `secrets-store-csi-driver-provider-gcp`
- `secrets-store-csi-driver-provider-vault`
- `secrets-store-csi-driver-provider-openbao`

The Secrets Store CSI Driver and its CRDs should be installed whenever any Podplane secrets provider is configured or any app template renders a `SecretProviderBinding`. Provider components should be enabled from cluster config provider kinds:

- `aws` enables the AWS provider once, regardless of whether configured providers use AWS Secrets Manager, AWS Parameter Store, or both.
- `gcp` enables the GCP provider.
- `vault` enables the Vault provider.
- `openbao` enables the OpenBao provider.
- Local clusters should always enable the OpenBao provider because the local path uses the normal Vault/OpenBao-compatible adapter against fakevault.

`podplane cluster create` and `podplane local start` should pass this through the normal seed path. The `netsyseed` platform-components values generation should inspect `cluster.secrets.providers[*].kind`, enable the Secrets Store CSI Driver/CRDs, enable the matching provider app components, and always enable OpenBao provider support for local clusters.

`podplane deps download` should also prefetch provider artifacts using sane defaults. When a user asks for an infrastructure/provider family such as `aws`, Podplane should download both the normal provider dependencies and the Secrets Store CSI AWS provider artifacts. The same applies to `gcp`. The OpenBao provider should always be downloaded because it is required for local clusters and fakevault-backed development, even when the production cluster config does not use OpenBao.

## CLI & Secrets Providers

The CLI should use the default provider unless `--provider` is set:

```sh
podplane secret create github-private-key --for aok-source-controller
podplane secret create github-private-key --for aok-source-controller --provider aws-secrets-manager
podplane secret list --for aok-source-controller
podplane secret list --for aok-source-controller --provider openbao
```

The aggregated API should treat `SecretProviderKeyspace` as the provider-specific logical keyspace aligned to a generated `SecretProviderClass`/`SecretProviderBinding` boundary. Encode the provider name in the Kubernetes resource name:

```text
<provider-name>.<secret-provider-binding-name>
```

For example, provider `aws-secrets-manager` and `SecretProviderClass/aok-source-controller` use `SecretProviderKeyspace/aws-secrets-manager.aok-source-controller`. Provider names must not contain dots, so the delimiter is unambiguous and remains valid as a Kubernetes DNS-subdomain object name. This keeps provider scope visible to Kubernetes RBAC and audit logs. If the user omits `--provider`, the CLI should resolve the cluster default provider before making the API request.

Deploy/template integration should also allow provider selection through template values. Templates render `SecretProviderBinding` resources, and the selected provider determines which Secrets Store CSI provider syntax the operator renders into the generated `SecretProviderClass`.

```sh
podplane deploy worker --name aok-source-controller --secret github-private-key
```

Deploy should resolve the cluster default secrets provider before rendering the default secrets item created by `--secret`. Template charts should expose provider selection as a stable template value so advanced users can select a non-default provider with `--set` or values files. The rendered `SecretProviderBinding.spec.providerName` must reference the same provider selected for `podplane secret create/update`; otherwise the generated `SecretProviderClass` will mount from a different backend than the one developers wrote to.

App templates that can render a `SecretProviderBinding`, such as `web`, should declare a static dependency on `podplane-operator` and `secrets-store-csi-driver` so deploy can enable the Podplane CRD/controller, CSI CRDs, and CSI driver before Helm renders the template. Templates should not statically declare a dependency on every concrete provider, because provider selection is cluster-config-driven and may be supplied through the default provider or values. The happy path is that `podplane cluster create`, `podplane local start`, or `podplane install` has already enabled the provider component that matches the selected/default provider.

The `podplane secret` implementation does not support automatically migrating values between providers. However, users can move workloads explicitly:

1. Configure the new provider in cluster config.
2. Re-create or copy required keys into the new provider, e.g. `podplane secret create ... --provider openbao`.
3. Redeploy or update the workload's `SecretProviderBinding` using template values such as `--set secrets[0].providerName=openbao`.
4. After workloads are confirmed on the new provider, delete old-provider keys using `--provider <old-provider>`.

The operator should not automatically move secret values between providers.

Deploy should support one `SecretProviderBinding` through simple CLI flags in the initial UX. For the simple CLI path, deploy should create one binding named exactly the deploy `--name`; the operator then generates a same-namespace `SecretProviderClass` with that same name. If the user does not select a provider through values, deploy should set `spec.providerName` to the resolved `cluster.secrets.default_provider`. Secret keys should be passed with a repeatable flag, matching existing deploy conventions for `--env` and `--set`:

```sh
podplane deploy web \
  --name aok-source-controller \
  --secret github-private-key \
  --secret webhook-secret
```

Do not add a dedicated mount-path flag initially. Templates should expose mount path as a normal template value, overridden with `--set` when needed:

```sh
podplane deploy web \
  --name aok-source-controller \
  --secret github-private-key \
  --set secrets[0].mountPath=/var/run/aok/secrets
```

Template values should use an array so values files can express multiple SecretProviderBindings even though the simple CLI path only creates one default item:

```yaml
secrets:
  - bindingName: aok-source-controller
    providerName: aws-secrets-manager
    mountPath: /var/run/podplane/secrets
    items:
      - key: github-private-key
      - key: webhook-secret
    syncToKubernetesSecrets: []
```

The repeatable `--secret` flag appends items to the CLI-generated default `secrets[0]` item:

```yaml
secrets:
  - bindingName: <deploy --name>
    providerName: <cluster.secrets.default_provider>
    mountPath: /var/run/podplane/secrets
    items:
      - key: <first --secret value>
      - key: <second --secret value>
```

For values-file-defined items, `providerName` may default to `cluster.secrets.default_provider`; `bindingName` and `mountPath` should be required because multiple items are expected to be intentionally distinct. Users can set array values with Helm-style `--set` index syntax, for example:

```sh
podplane deploy web \
  --name aok-source-controller \
  --set secrets[0].providerName=openbao \
  --set secrets[0].bindingName=aok-source-controller \
  --set secrets[0].mountPath=/var/run/podplane/secrets \
  --set secrets[0].items[0].key=github-private-key
```

Each `secrets[].items[]` item has a required `key` and optional `path`. `key` is the Podplane logical key and backend identifier/path segment, so it must keep the strict portable key-name rules. `path` is the mounted relative path inside `mountPath`, defaults to `key`, and should be validated with Kubernetes/Secrets Store CSI-safe path rules: non-empty, relative, clean, no `..` path elements, not starting with `..`, and within sane path/component length limits.

Templates may configure `syncToKubernetesSecrets` as an advanced values-only opt-in. Podplane should not add simple deploy flags for this because syncing to Kubernetes Secrets stores secret values in etcd (unencrypted) base64-encoding, and is not the recommended path. This aligns with the Secrets Store CSI Driver model: sync-as-Kubernetes-Secret exists for compatibility when an application absolutely requires native Kubernetes Secrets or env vars, not as the default consumption path. Example:

```yaml
secrets:
  - bindingName: aok-source-controller
    providerName: aws-secrets-manager
    mountPath: /var/run/podplane/secrets
    items:
      - key: github-private-key
        path: id_rsa
    syncToKubernetesSecrets:
      - secretName: aok-source-controller-secrets
        type: Opaque
        data:
          - fromKey: github-private-key
            key: GITHUB_PRIVATE_KEY
```

This lets advanced users map a Podplane key such as `github-private-key` to a Kubernetes Secret key such as `GITHUB_PRIVATE_KEY` e.g. ideal for env-var use. The operator translates `fromKey` to the generated Secrets Store CSI `secretObjects[].data[].objectName`, using the matching mounted content path for that Podplane key: `path` when set, otherwise `key`. In upstream Secrets Store CSI, `secretObjects[].data[].objectName` refers to the mounted content name/path, not the backend provider object name. Treat this as an escape hatch for compatibility when a Kubernetes `Secret` is required, such as for setting them as env vars for legacy applications. The values schema and docs must warn clearly that enabling `syncToKubernetesSecrets` creates native Kubernetes Secrets containing plaintext secret values in etcd (anyone with sufficient kube-apiserver or etcd access can read those secret values).

For example, this Podplane SecertProviderBinding excerpt:

```yaml
items:
  - key: github-private-key
    path: id_rsa
syncToKubernetesSecrets:
  - secretName: aok-source-controller-secrets
    type: Opaque
    data:
      - fromKey: github-private-key
        key: GITHUB_PRIVATE_KEY
```

should render AWS Secrets Store CSI fields like:

```yaml
parameters:
  objects: |
    - objectName: "/prod-cluster/platform-aok/aok-source-controller/github-private-key" # backend object
      objectType: "secretsmanager"
      objectAlias: "id_rsa" # mounted content path/name
secretObjects:
  - secretName: aok-source-controller-secrets
    type: Opaque
    data:
      - objectName: id_rsa # mounted content path/name, not backend object
        key: GITHUB_PRIVATE_KEY # Kubernetes Secret data key
```

The template should render a `SecretProviderBinding`, not a `SecretProviderClass`:

```yaml
apiVersion: secrets.podplane.dev/v1beta1
kind: SecretProviderBinding
metadata:
  namespace: platform-aok
  name: aok-source-controller
spec:
  providerName: aws-secrets-manager
  mountPath: /var/run/podplane/secrets
  items:
    - key: github-private-key
      path: id_rsa
    - key: webhook-secret
  syncToKubernetesSecrets: []
```

`spec.providerName` is the cluster-local provider name from Podplane cluster config, not the upstream Secrets Store CSI provider slug. The operator resolves it to a provider kind such as `aws`, `gcp`, `vault`, or `openbao`.

`SecretProviderBinding` status should expose the resolved provider and generated implementation resource:

```yaml
status:
  provider:
    name: aws-secrets-manager
    kind: aws
  secretProviderClass:
    name: aok-source-controller
  items:
    - key: github-private-key
      path: id_rsa
  conditions: []
```

The operator should create or update only the `SecretProviderClass` owned by the binding. The generated `SecretProviderClass` must carry a same-namespace controller `ownerReference` pointing to the `SecretProviderBinding`, plus labels/annotations for auditability. If a `SecretProviderClass` with the target name already exists without a matching Podplane owner reference, or is owned by a different binding UID, reconciliation must fail with a conflict status rather than adopting or overwriting it implicitly.

The generated `SecretProviderClass` should include labels/annotations linking it back to the Podplane binding, for example:

```yaml
metadata:
  labels:
    app.kubernetes.io/managed-by: podplane
    secrets.podplane.dev/secret-provider-binding: aok-source-controller
```

Provider identity should live on the owning `SecretProviderBinding` spec/status. If platform teams update the provider configuration behind a provider name, the operator should reconcile any provider-specific SecretProviderClass changes from the binding.

Example AWS Secrets Manager `SecretProviderClass` rendering:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  namespace: platform-aok
  name: aok-source-controller
  labels:
    app.kubernetes.io/managed-by: podplane
    secrets.podplane.dev/secret-provider-binding: aok-source-controller
spec:
  provider: aws
  parameters:
    objects: |
      - objectName: "/prod-cluster/platform-aok/aok-source-controller/github-private-key"
        objectType: "secretsmanager"
        objectAlias: "github-private-key"
      - objectName: "/prod-cluster/platform-aok/aok-source-controller/webhook-secret"
        objectType: "secretsmanager"
        objectAlias: "webhook-secret"
```

Example GCP Secret Manager `SecretProviderClass` rendering using the upstream provider:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  namespace: platform-aok
  name: aok-source-controller
  labels:
    app.kubernetes.io/managed-by: podplane
    secrets.podplane.dev/secret-provider-binding: aok-source-controller
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/prod-project/secrets/prod-cluster_platform-aok_aok-source-controller_github-private-key/versions/latest"
        path: "github-private-key"
      - resourceName: "projects/prod-project/secrets/prod-cluster_platform-aok_aok-source-controller_webhook-secret/versions/latest"
        path: "webhook-secret"
```

The GCP `path` field is the mounted filename and must be derived from the Podplane key name. The operator must generate the `resourceName` identity from the selected provider config and Podplane key, not from user-supplied raw GCP fields. For v1, generated resources should render `versions/latest` only. Regional GCP Secret Manager resources may be supported by adding a non-sensitive provider field such as `location` and rendering `projects/<project-id>/locations/<location>/secrets/<secret-id>/versions/latest`; when `location` is unset, use the global `projects/<project-id>/secrets/<secret-id>/versions/latest` form.

Example OpenBao/Vault KV-v2 rendering shape:

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  namespace: platform-aok
  name: aok-source-controller
  labels:
    app.kubernetes.io/managed-by: podplane
    secrets.podplane.dev/secret-provider-binding: aok-source-controller
spec:
  provider: openbao
  parameters:
    baoAddress: "https://bao.example.com"
    roleName: "aok-source-controller"
    objects: |
      - objectName: "github-private-key"
        secretPath: "secret/data/prod-cluster/platform-aok/aok-source-controller/github-private-key"
        secretKey: "value"
      - objectName: "webhook-secret"
        secretPath: "secret/data/prod-cluster/platform-aok/aok-source-controller/webhook-secret"
        secretKey: "value"
```

Vault rendering is the same shape but uses `spec.provider: vault` and Vault-prefixed parameters such as `vaultAddress`. OpenBao rendering uses `spec.provider: openbao` and Bao-prefixed parameters such as `baoAddress`. For Vault/OpenBao, the generated CSI `roleName` should default to the `SecretProviderBinding` name, matching Podplane's `secretProviderClass == serviceAccountName` convention. The operator contract should remain: reconcile each `SecretProviderBinding`, loop over `spec.items`, map each `key` to the provider-specific backend path, and expose each secret at its `path`, defaulting `path` to `key`.

Multiple SecretProviderBindings per deploy, per-secret provider selection, and per-secret mount paths are out of scope for the simple deploy flags. Advanced users can use `--set`, values files, or custom templates when they need those shapes.

## Deployment naming

Initial implementation should live in a new repository:

```text
github.com/podplane/operator
```

The chart should initially live in `podplane/components` to avoid multi-repo chart rollout choreography. It can be promoted into `github.com/podplane/operator` later if the operator becomes generally useful outside Podplane.

Recommended Kubernetes names:

```text
Namespace:    platform-podplane
Deployment:   platform-podplane-operator
Service:      platform-podplane-operator
Binary:       podplane-operator
Repo:         github.com/podplane/operator
Chart:        platform-podplane
Feature:      Podplane Secrets
Aggregated API group: secrets-api.podplane.dev
CRD API group:        secrets.podplane.dev
```

The `APIService` should point at:

```yaml
service:
  namespace: platform-podplane
  name: platform-podplane-operator
```

TLS serving certificates should be managed by cert-manager resources in the `platform-podplane` chart. The chart should create a `Certificate`, mount the resulting serving cert into the operator pod, and use cert-manager CA injection for the `APIService` `caBundle`. The operator should not implement custom self-signed certificate bootstrap in v1. Podplane Secrets is part of the platform/recommended components path and is not required for the minimal seed.

The `platform-podplane-operator` Deployment must run exactly `1` replica by default and should not expose a normal autoscaling path. This is acceptable for Podplane Secrets traffic: secret writes are human/automation control-plane operations, and `SecretProviderBinding` reconciliation should be lightweight. A single correctly vertically scaled operator is simpler and safer than horizontally scaling the aggregated API while it uses an in-memory encryption key for client-side encrypting create/update request values.

## Chart RBAC defaults

The `platform-podplane` chart should create the operator service account and RBAC by default, with explicit enable flags:

```yaml
serviceAccount:
  create: true

rbac:
  create: true
  aggregateDefaultRoles: true
```

When `rbac.create` is true, the chart should create the operator RBAC and helper ClusterRoles. When `rbac.aggregateDefaultRoles` is also true, the relevant helper ClusterRoles should carry Kubernetes aggregation labels. Following the cert-manager pattern, helper ClusterRoles can also augment built-in default roles when their rules exactly match the desired default-role behavior:

- `podplane-secrets-admin`: full named `secretproviderkeyspaces` management plus `publickeys` get; label with `aggregate-to-admin: "true"`.
- `podplane-secrets-view`: metadata-only named `secretproviderkeyspaces` get plus `publickeys` get; label with `aggregate-to-edit: "true"`.
- `podplane-secrets-publickey-read`: `publickeys/latest` get only; label with `aggregate-to-view: "true"`, `aggregate-to-edit: "true"`, and `aggregate-to-admin: "true"`.

Do not create an `aggregate-to-view` role for `secretproviderkeyspaces` by default. The public key reader is safe to aggregate to `view` because `publickeys/latest` contains only (public key) encryption metadata required by clients before they can submit encrypted write requests; it does not reveal secret values or backend key metadata.

This mirrors Kubernetes' conservative treatment of native `Secret` access: broad view should not reveal secret metadata unless an admin explicitly opts in.

The helper roles should not create broad user bindings by default. App/provider-specific write/delete access should normally be granted with explicit names such as:

```yaml
resourceNames:
  - aws-secrets-manager.aok-source-controller
```

## Aggregated API and SecretProviderBinding CRD

Use Kubernetes API aggregation for the operator-served secrets API: `publickeys` exposes the current encryption public key, and `secretproviderkeyspaces` proxies metadata-only list plus encrypted write/delete/restore operations to the configured external provider - these resources are virtual and are not persisted in etcd. Use a normal namespaced Kubernetes CRD for declarative workload read/mount configuration: `SecretProviderBinding`.

Register an `APIService` such as:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.secrets-api.podplane.dev
spec:
  group: secrets-api.podplane.dev
  version: v1beta1
  service:
    namespace: platform-podplane
    name: platform-podplane-operator
  groupPriorityMinimum: 1000
  versionPriority: 15
```

The aggregated API must be implemented as a proper Kubernetes extension API server, not as a generic HTTP handler that trusts inbound headers. It must:

- authenticate the kube-apiserver aggregation proxy client certificate using the requestheader client CA and allowed names published through `kube-system/extension-apiserver-authentication`;
- trust original-user headers such as `X-Remote-User` and `X-Remote-Group` only after the aggregation proxy client is authenticated;
- authorize the original user with delegated Kubernetes authorization, including `SubjectAccessReview` checks for normal requests and the custom verbs enforced by the operator;
- bind the operator service account to Kubernetes' `extension-apiserver-authentication-reader` Role and `system:auth-delegator` ClusterRole;
- reject direct in-cluster calls that do not come through the authenticated kube-apiserver aggregation proxy.

This is security-critical. A caller must not be able to bypass kube-apiserver RBAC by calling the operator Service directly or by spoofing requestheader identity headers.

The operator must also serve complete Kubernetes discovery for the aggregated API, including current aggregated-discovery media types used by supported Kubernetes versions. The APIService should only become `Available=True` when TLS, delegated auth, discovery, and handler readiness are all functional. Broken aggregated API discovery can affect unrelated cluster operations such as namespace deletion, so upgrade and uninstall paths must avoid leaving a stale or unavailable `v1beta1.secrets-api.podplane.dev` APIService behind.

The Podplane operator serves virtual API resources backed directly by the configured secret provider backend. No secret metadata or values need to be persisted in Kubernetes.

The aggregated API write path and the Secrets Store CSI read path are separate Kubernetes objects coordinated by convention:

- `SecretProviderBinding/<binding-name>` is the Podplane read/mount intent for workloads.
- `SecretProviderClass/<secret-provider-class-name>` is the generated CSI read/mount implementation for workloads. In the Podplane-managed path, this name is the same as the owning binding name.
- `secrets-api.podplane.dev` `secretproviderkeyspaces/<provider-name>.<secret-provider-binding-name>` is the Podplane write/list/delete API for backend keys under that binding name in that provider.
- The shared `<secret-provider-binding-name>` is the Podplane backend keyspace boundary. The operator encodes that binding name into the provider-specific object paths/resource names it writes into the generated `SecretProviderClass`, so the backend provider secret __prefix__ is `/<cluster-secrets-prefix>/<namespace>/<secret-provider-binding-name>/`.

Conceptually, a SecretProviderKeyspace resource represents the logical space of keys under that provider-specific backend prefix.

In the working example, `SecretProviderKeyspace/aws-secrets-manager.aok-source-controller` in Kubernetes namespace `platform-aok` writes keys under the same backend prefix that `SecretProviderClass/aok-source-controller` is allowed to read.

- The command `podplane secret create github-private-key --namespace platform-aok --for aok-source-controller --provider aws-secrets-manager` maps to `/<cluster-secrets-prefix>/platform-aok/aok-source-controller/github-private-key` in the `aws-secrets-manager` provider.
- Deleting `SecretProviderKeyspace/aws-secrets-manager.aok-source-controller` deletes the backend keys managed under that prefix in that provider.

The Kubernetes RBAC resource name is the provider plus SecretProviderClass boundary, not the individual secret key e.g. `aws-secrets-manager.aok-source-controller`. The key name, such as `github-private-key`, is carried in the encrypted create/update request body and becomes the final backend path segment. This lets RBAC grant access to all keys under one provider/SPC boundary without enumerating every key.

The operator should be configured with a cluster-wide secrets prefix via startup flag or environment variable. The default should be the Podplane cluster ID. This keeps multiple clusters sharing the same external backend isolated by default.

The operator's provider config should use the same map shape and non-secret provider fields as `cluster.secrets.providers`; the platform chart/netsyseed path can render that map directly. Provider credentials are configured separately by the chart as mounted files or workload identity. Static token values and token file paths must not be embedded in Podplane cluster config, cluster summaries, or operator config. If a Vault/OpenBao token is required, the chart/admin must mount it at `/var/run/podplane/providers/<provider-name>/token`. AWS Secrets Manager recovery-window overrides are an advanced operator/chart setting and are not part of cluster config or cluster summary.

The operator should generate an in-memory X25519 public/private keypair and expose the current public key through the aggregated API. The operator keypair should be stable across many writes and rotate on a coarse interval, initially every 6 hours (but configurable). `podplane secret create/update` should fetch the current public key, encrypt the secret value client-side, and send only ciphertext through the Kubernetes API. If the operator rejects a write because the referenced operator `keyID` is stale, the CLI should refetch the public key and retry once.

The in-memory operator keypair requires the operator to run as a singleton (`replicas: 1`). Multiple replicas would cause clients to fetch a public key from one operator pod and submit ciphertext to another pod that cannot decrypt it. Persisting or sharing the private key is intentionally out of scope for v1 because it would add bootstrap-secret handling and increase the blast radius if Kubernetes storage or backups are compromised. Encrypted dry-run manifests are therefore short-lived operational artifacts and may become unusable after an operator key rotation, restart, or rollout; clients should refetch `publickeys/latest` and regenerate them when needed.

Secret value encryption should use only Go standard-library cryptography in v1:

```text
algorithm: x25519-hkdf-sha256-aes-256-gcm
```

The operator public key is a base64-encoded raw X25519 public key. The CLI should accept standard padded base64 and unpadded base64url encodings; the operator should emit one canonical encoding consistently in `publickeys/latest`. For each create/update encryption, the CLI should generate a fresh ephemeral X25519 sender keypair, derive a shared secret with the operator public key, derive a 32-byte content-encryption key using HKDF-SHA-256 with the ephemeral sender public key as salt and info string `podplane secrets v1`, then encrypt the value with AES-256-GCM using a fresh random nonce. The ephemeral sender public key and nonce are included in the ciphertext envelope. This per-encryption ephemeral key is not the operator key and does not affect the 6-hour (but configurable) operator key rotation window.

Bind the ciphertext to its destination by using AES-GCM associated data that the operator can reconstruct from the request and cluster identity. The associated data should include the canonical `encryptedValue.algorithm` value plus the Podplane cluster ID, Kubernetes namespace, `SecretProviderKeyspace` resource name, and entry key. This prevents a ciphertext captured from one Podplane secret destination from being replayed into another destination while the same operator key is still active. Do not include fields that clients cannot know during `--dry-run=client` generation, such as the cluster-wide secrets prefix.

The canonical AES-GCM associated-data byte string is UTF-8 text made by joining these fields in this exact order with a single NUL byte (`0x00`) between fields and no leading/trailing separator:

```text
encryptedValue.algorithm
clusterID
metadata.namespace
metadata.name
spec.entries[i].key
```

For example:

```text
x25519-hkdf-sha256-aes-256-gcm\0prod-cluster\0platform-aok\0aws-secrets-manager.aok-source-controller\0github-private-key
```

The operator must reconstruct the same byte string before AES-GCM decryption. Do not include `encryptedValue.keyID`, the envelope version, the operator public key, the cluster-wide secrets prefix, backend path, provider kind, operation, or any field unavailable to `--dry-run=client`.

The ciphertext string should be a versioned Podplane envelope:

```text
podplane-secrets-v1.<base64url-json-envelope>
```

The decoded JSON envelope contains:

```json
{
  "version": "podplane-secrets-v1",
  "ephemeralPublicKey": "base64url-raw-x25519-public-key",
  "nonce": "base64url-aes-gcm-nonce",
  "ciphertext": "base64url-aes-gcm-ciphertext-and-tag"
}
```

Use `encryptedValue.algorithm` as the single source of truth for the encryption algorithm. The envelope `version` field and `podplane-secrets-v1.` prefix identify the envelope format; the envelope should not carry a second algorithm field.

Initial aggregated API resource model:

```text
API group: secrets-api.podplane.dev
Version:   v1beta1

Namespaced resource:
  secretproviderkeyspaces

Cluster-scoped resource:
  publickeys
```

Initial CRD resource model:

```text
API group: secrets.podplane.dev
Version:   v1beta1

Namespaced resource:
  secretproviderbindings
```

Example API paths:

In these examples, `platform-aok` is the Kubernetes namespace, `aws-secrets-manager` is the provider name, `aok-source-controller` is the `SecretProviderClass` name, `aws-secrets-manager.aok-source-controller` is the Podplane `SecretProviderKeyspace` resource name, and `github-private-key` is a key carried in the encrypted create/update body.

```text
GET    /apis/secrets-api.podplane.dev/v1beta1/publickeys/latest

GET    /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
PUT    /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
DELETE /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

The `PUT secretproviderkeyspaces/<provider-name>.<secret-provider-binding-name>` request body identifies one or more entries, the requested operation for each entry, and encrypted value payloads for write operations. Production should not expose a value-read operation - if truly required, secret value reading functionality of the secrets backend provider can be used to recover the value.

### Aggregated API schema

Use Kubernetes-style API objects for normal responses so `kubectl get --raw` and client-go discovery behave predictably. Values below are the v1 minimum shape; fields can be extended later with normal Kubernetes-style optional fields.

Public key:

```http
GET /apis/secrets-api.podplane.dev/v1beta1/publickeys/latest
```

```json
{
  "apiVersion": "secrets-api.podplane.dev/v1beta1",
  "kind": "PublicKey",
  "metadata": { "name": "latest" },
  "spec": {
    "keyID": "0198f2b0-6c9a-7c51-9f9a-0c5f0c8f71a2",
    "createdAt": "2026-06-17T10:00:00Z",
    "algorithm": "x25519-hkdf-sha256-aes-256-gcm",
    "publicKey": "base64-raw-x25519-public-key"
  }
}
```

`keyID` should be an opaque UUID string. UUIDv7 is preferred for sortable IDs if already available; otherwise a normal random UUID is fine. Clients must not parse semantics from it.

Create one key:

```http
PUT /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

Update one key:

```http
PUT /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

```json
{
  "apiVersion": "secrets-api.podplane.dev/v1beta1",
  "kind": "SecretProviderKeyspace",
  "metadata": {
    "namespace": "platform-aok",
    "name": "aws-secrets-manager.aok-source-controller"
  },
  "spec": {
    "entries": [
      {
        "key": "github-private-key",
        "operation": "create",
        "encryptedValue": {
          "keyID": "0198f2b0-6c9a-7c51-9f9a-0c5f0c8f71a2",
          "algorithm": "x25519-hkdf-sha256-aes-256-gcm",
          "ciphertext": "podplane-secrets-v1.eyJ2ZXJzaW9uIjoiLi4uIn0"
        }
      }
    ]
  }
}
```

The provider is selected by the `SecretProviderKeyspace` resource name prefix. `PUT` with `operation: create` creates a new key and must fail if the key already exists or is archived. `PUT` with `operation: update` updates an existing active key and must fail if the key is missing or archived. Updating an existing value is more sensitive than creating a new key, so the operator must perform an explicit Kubernetes `SubjectAccessReview` for custom verb `overwrite` before executing `operation: update`. The `encryptedValue` field is required for each `spec.entries[]` item in create/update requests and omitted from normal read/list responses. For v1, the CLI sends exactly one entry; the API shape allows bulk writes later.

Successful create/update response:

```json
{
  "apiVersion": "secrets-api.podplane.dev/v1beta1",
  "kind": "SecretProviderKeyspace",
  "metadata": {
    "namespace": "platform-aok",
    "name": "aws-secrets-manager.aok-source-controller"
  },
  "status": {
    "provider": "aws-secrets-manager",
    "entries": [
      {
        "key": "github-private-key",
        "status": "active",
        "backendPath": "/prod-cluster/platform-aok/aok-source-controller/github-private-key"
      }
    ]
  }
}
```

If the write references a stale public key, return HTTP `409 Conflict` with a Kubernetes `Status` object using reason `Conflict` and message `stale public key`. The CLI should refetch `publickeys/latest` and retry once.

List keys for one provider and SecretProviderClass boundary:

```http
GET /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

```json
{
  "apiVersion": "secrets-api.podplane.dev/v1beta1",
  "kind": "SecretProviderKeyspace",
  "metadata": {
    "namespace": "platform-aok",
    "name": "aws-secrets-manager.aok-source-controller"
  },
  "status": {
    "provider": "aws-secrets-manager",
    "entries": [
      {
        "key": "github-private-key",
        "status": "active",
        "backendPath": "/prod-cluster/platform-aok/aok-source-controller/github-private-key"
      }
    ]
  }
}
```

This is the normal Kubernetes named-resource shape. The backend-observed entries are returned in `status.entries`, not `spec`, because they are derived from the external backend rather than desired state stored in Kubernetes. `status.entries[].status` is `active`, `archived`, or `unknown`; archived entries may include `restoreUntil` when the provider exposes a recovery deadline. If the provider list succeeds but per-key status lookup fails or times out, return the listed key with `status: unknown` rather than failing the whole list response. `podplane secret list --for aok-source-controller` should call this named-resource endpoint.

Listing is intentionally limited to keys under one provider/SPC boundary so every request remains provider-scoped and RBAC can use exact `resourceNames`.

Delete/archive one key:

```http
DELETE /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller?key=github-private-key
```

The key query parameter is required for single-key deletes. Delete means recoverable archive. If the selected provider does not support archived delete, this request must fail with a clear error telling the user to use destroy instead.

Destroy one key permanently:

```http
DELETE /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller?key=github-private-key&destroy=true
```

Destroy permanently removes archived value/history where the provider supports it, or performs the provider's irreversible delete when no archive mode exists. The operator must perform an explicit Kubernetes `SubjectAccessReview` for custom verb `destroy` before executing a destroy request.

Delete/archive all keys under one SecretProviderClass boundary:

```http
DELETE /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

Destroy all keys under one SecretProviderClass boundary:

```http
DELETE /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller?destroy=true
```

Delete responses should be Kubernetes `Status` objects. The CLI should require `--all` before calling either boundary form. Boundary deletes are provider-scoped by the `SecretProviderKeyspace` resource name. Plain boundary delete means archive all keys and must fail if the provider cannot archive every affected key. Boundary destroy means permanently destroy all keys and requires custom `destroy` RBAC permission.

Restore one archived key:

```http
PUT /apis/secrets-api.podplane.dev/v1beta1/namespaces/platform-aok/secretproviderkeyspaces/aws-secrets-manager.aok-source-controller
```

```json
{
  "apiVersion": "secrets-api.podplane.dev/v1beta1",
  "kind": "SecretProviderKeyspace",
  "metadata": {
    "namespace": "platform-aok",
    "name": "aws-secrets-manager.aok-source-controller"
  },
  "spec": {
    "entries": [
      {
        "key": "github-private-key",
        "operation": "restore"
      }
    ]
  }
}
```

Restore reuses the named `PUT SecretProviderKeyspace` endpoint, so kube-apiserver authorizes the request as normal `update` on `secretproviderkeyspaces` scoped to the exact `SecretProviderKeyspace` resource name. The operator must then perform an explicit Kubernetes `SubjectAccessReview` for the same user, namespace, resource name, API group, and `secretproviderkeyspaces` resource with custom verb `restore`. Restore is allowed only when both checks pass.

### Delete and restore semantics

`podplane secret delete` should mean "make this key unavailable to workloads now" rather than "physically destroy every trace immediately". After a successful delete:

- Secrets Store CSI/provider reads of that key should fail or return no current value.
- `podplane secret list --for ... --provider ...` should show the key as `archived` if the provider supports restore.
- `podplane secret list --for ... --provider ... --hide-archived` should hide archived keys.
- Permanently deleted keys should not be listed.
- Existing mounted files may remain until the CSI driver/provider refreshes or the pod is restarted; delete only controls backend availability.

Provider adapters should use the safest native deletion mode that makes the current value unreadable immediately:

| Provider | Delete behaviour | Restore | Destroy behaviour | Podplane list behaviour | Prevents new Secrets Store CSI mounts |
| --- | --- | --- | --- | --- |
| AWS Secrets Manager | Schedule deletion with recovery window | `RestoreSecret` | `ForceDeleteWithoutRecovery`; recreate may need retry due eventual consistency | `archived` during recovery window | Yes; scheduled-for-deletion secrets are not directly accessible |
| AWS Parameter Store | Not supported; archive delete should fail | Not supported | Delete parameter | Hidden after destroy | Yes |
| GCP Secret Manager | Disable the active/latest version | Re-enable version | Destroy disabled archived version(s) | `archived` while disabled | Yes; `latest` does not fall back to an older enabled version |
| Vault/OpenBao KV-v2 | Soft-delete latest version | Undelete latest version | Delete metadata for the logical key | `archived` while soft-deleted | Yes |

If a provider uses a recoverable/archive-style delete, Podplane should expose restore for that provider/key within the provider-defined recovery window. Providers without native recovery should reject archive delete and require explicit destroy.

For Vault/OpenBao KV-v2, Podplane operator does not expose per-version operations. A Podplane key maps to one logical KV-v2 metadata path. `delete` should soft-delete the latest/current version so normal CSI reads of the key fail. `restore` should undelete that soft-deleted latest/current version. `destroy` should delete the KV-v2 metadata path for the logical key, permanently removing all versions and metadata so the key disappears from `podplane secret list`, cannot be restored, and can be recreated as a clean logical key. `destroy --all` should delete metadata for every Podplane-managed key under the selected SecretProviderClass prefix.

`podplane secret create` and `podplane secret update` should fail when the key is archived. The error should explain both recovery paths: use `podplane secret restore KEY --for ...` to restore the previous value, or use `podplane secret destroy KEY --for ...` followed by `podplane secret create KEY --for ...` to permanently discard archived history and write a new value.

Podplane-generated `SecretProviderClass` resources should not include explicit historical version references where providers support them, such as AWS object version selectors, GCP explicit old versions, or Vault/OpenBao explicit version queries. Workloads using `SecretProviderBinding` will mount the current/latest active version of a key only. Raw `SecretProviderClass` usage remains an operator-controlled escape hatch enable by configuring standard RBAC roles/bindings.

## Authorization model

Native Kubernetes RBAC is the authorization model.

Example role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: platform-aok
  name: aok-source-controller-secret-writer
rules:
  - apiGroups: ["secrets-api.podplane.dev"]
    resources: ["secretproviderkeyspaces"]
    resourceNames: ["aws-secrets-manager.aok-source-controller"]
    verbs: ["get", "update", "delete", "overwrite", "restore", "destroy"]

  - apiGroups: ["secrets-api.podplane.dev"]
    resources: ["publickeys"]
    resourceNames: ["latest"]
    verbs: ["get"]
```

Developers only need Kubernetes/Podplane access. The operator/backend adapter holds the backend identity through workload identity, IAM role, OpenBao token, or local configuration.

RBAC is intentionally scoped at the `namespace + provider name + SecretProviderBinding name` boundary. For example, `resourceNames: ["aws-secrets-manager.aok-source-controller"]` grants access to keys under `/prod-cluster/platform-aok/aok-source-controller/*` in the `aws-secrets-manager` provider; it does not require every key such as `github-private-key` or `webhook-secret` to be listed in RBAC.

Create and update use the same named `PUT SecretProviderKeyspace` transport but require different authorization. Kubernetes cannot restrict top-level `create` requests by `resourceNames`, so kube-apiserver authorizes both requests as normal `update` on the exact `SecretProviderKeyspace` `resourceName`. The operator then enforces `operation: create` vs `operation: update` semantics from the request body. `operation: create` only requires the normal named `update` permission and must fail if the key already exists or is archived. `operation: update` requires the normal named `update` permission plus a successful explicit `SubjectAccessReview` for custom verb `overwrite` on the same named `secretproviderkeyspaces` resource. This preserves a simple Kubernetes-style named resource API while letting platform teams grant create-new-key access without granting overwrite access.

Restore requires both `update` and custom verb `restore` on the same named `secretproviderkeyspaces` resource. Destroy requires normal `delete` authorization for the incoming request plus custom verb `destroy` on the named `secretproviderkeyspaces` resource. Kubernetes accepts custom RBAC verbs, and the operator enforces `overwrite`/`restore`/`destroy` with explicit `SubjectAccessReview` checks after the normal kube-apiserver authorization succeeds. This lets platform teams allow create/delete without allowing overwrite of active credentials, restore of archived credentials, or permanent destruction of archived history.

Namespace-wide `list` of `secretproviderkeyspaces` is intentionally not part of the initial API. Kubernetes RBAC cannot restrict `list` to `resourceNames` or provider prefixes such as `aws-secrets-manager.*`, so supporting broad list would weaken the provider-scoped authorization model.

## Backend path invariant

Use the tuple below as the identity boundary:

```text
namespace + SecretProviderBinding name
```

For a cluster with secrets prefix `prod-cluster`, and a `SecretProviderBinding` named `aok-source-controller` in namespace `platform-aok`, all Podplane-managed backend secrets for that binding must live under:

```text
/prod-cluster/platform-aok/aok-source-controller/
```

The cluster secrets prefix should follow the exact same validation rules as `cluster.id`:

```text
^[a-z0-9]([a-z0-9-]*[a-z0-9])?$
maximum length: 32
no consecutive hyphens
reserved values: local, k8s, oidc
```

The default cluster secrets prefix should be `cluster.id`, which already satisfies these rules. For v1, keep every backend path segment slash-free and DNS-label-like:

- namespace: Kubernetes namespace name
- SecretProviderBinding name: max 63 chars, lowercase alphanumeric plus hyphen, no leading/trailing hyphen
- key name: max 63 chars, lowercase alphanumeric plus hyphen, no leading/trailing hyphen

This keeps provider mappings portable and avoids GCP Secret Manager length/character issues.

Allowed:

```text
/prod-cluster/platform-aok/aok-source-controller/github-private-key
/prod-cluster/platform-aok/aok-source-controller/webhook-secret
```

Rejected:

```text
/prod-cluster/platform-aok/shared/github-private-key
/prod-cluster/orders/aok-source-controller/github-private-key
/other-cluster/platform-aok/aok-source-controller/github-private-key
```

The CLI should map:

```sh
podplane secret create github-private-key --for aok-source-controller
```

in namespace `platform-aok` to the logical backend path:

```text
/prod-cluster/platform-aok/aok-source-controller/github-private-key
```

For v1, each logical key (e.g. `github-private-key`) maps to one physical backend secret/parameter object. A single `SecretProviderClass` may still reference many backend objects under its allowed prefix. For example, `aok-source-controller` may mount `github-private-key`, `webhook-secret`, and `github-app-id` as separate objects. Packed JSON/multi-field backend objects are out of scope for v1.

Provider-specific mappings:

```text
AWS Secrets Manager: /prod-cluster/platform-aok/aok-source-controller/github-private-key
AWS Parameter Store: /prod-cluster/platform-aok/aok-source-controller/github-private-key
GCP Secret Manager: prod-cluster_platform-aok_aok-source-controller_github-private-key
OpenBao KV-v2:       secret/data/prod-cluster/platform-aok/aok-source-controller/github-private-key
Vault KV-v2:         secret/data/prod-cluster/platform-aok/aok-source-controller/github-private-key
```

Provider constraints that shape the v1 path rules:

- AWS Secrets Manager supports `/` in secret names and allows up to 512 chars.
- AWS Parameter Store supports `/` hierarchy, but has a 15-level hierarchy limit and name-length limits; the proposed four-segment path is safe.
- GCP Secret Manager secret IDs do not support `/`, only letters, numbers, hyphen, and underscore, with a 255-char limit. Encoding the four segments with `_` is safe because Podplane-controlled segments do not allow underscores. Worst case is `32 + 1 + 63 + 1 + 63 + 1 + 63 = 224` chars.
- Vault/OpenBao KV-v2 support slash-separated paths; keep segments URL-safe.

The external backend is the source of truth for listing secrets. The aggregated API should derive list metadata from backend list APIs and provider tags/labels where available. Provider-specific detail calls may be used to derive archived state; failures in those detail calls should degrade the affected entries to `status: unknown` after the key list itself has been retrieved.

## SecretProviderBinding controller

Raw `SecretProviderClass` should remain the cluster-operator escape hatch for advanced CSI/provider features, controlled by normal Kubernetes RBAC. Developers should normally create `SecretProviderBinding` resources instead. Podplane operator does not serve any `ValidatingAdmissionWebhook`.

Initial `SecretProviderBinding` policy:

- Treat `metadata.namespace + metadata.name` as the SecretProviderClass/backend prefix boundary.
- Require `spec.providerName` to reference a configured cluster provider name.
- Require every `spec.items[].key` item to satisfy the Podplane key-name rules and every `spec.items[].path` value to satisfy CSI-safe relative path rules.
- Generate one owned `SecretProviderClass` with the same namespace/name as the binding.
- For every generated backend secret, use the configured cluster-wide secrets prefix:

```text
/<cluster-secrets-prefix>/<metadata.namespace>/<metadata.name>/<key>
```

For AWS, the controller must generate `spec.parameters.objects` from `spec.items[]` items and provider config. In v1, Podplane-managed bindings should not use AWS full ARN references, `jmesPath`, `failoverObject`, or historical object versions. Those remain raw `SecretProviderClass` escape-hatch features for users with direct upstream RBAC.

For GCP, the controller must generate upstream `spec.provider: gcp` resources from provider config:

- reject configured provider kind `gke`; Podplane uses the upstream `gcp` provider and `secrets-store.csi.k8s.io` driver
- generate global resource names of the form `projects/<project-id>/secrets/<podplane-secret-id>/versions/latest`, or regional resource names of the form `projects/<project-id>/locations/<location>/secrets/<podplane-secret-id>/versions/latest` when the provider config declares `location`
- generate `<podplane-secret-id>` from the GCP-encoded backend path for `<cluster-secrets-prefix>/<metadata.namespace>/<metadata.name>/<key>`
- generate `path` from the Podplane key as the mounted filename

The controller should report invalid bindings, provider resolution failures, and generated `SecretProviderClass` collisions through `status.conditions` and Kubernetes events. It should not mutate or audit raw, non-owned `SecretProviderClass` objects.

## Pod-side SecretProviderClass admission policy

Podplane operator chart should install a native Kubernetes CEL `ValidatingAdmissionPolicy` by default to ensure workloads can only mount their own `SecretProviderClass` generated by their own `SecretProviderBinding`. This closes the gap between `SecretProviderBinding`/`SecretProviderClass` authorization and pod creation: RBAC can restrict who creates or edits bindings and raw `SecretProviderClass` objects, but a pod that can be created in the same namespace can otherwise reference any existing `SecretProviderClass` by name.

This policy should be Kubernetes-native rather than served by the Podplane operator. Keeping the pod admission check in kube-apiserver is safer and more reliable: if `podplane-operator` is restarting or unavailable, users should still be able to deploy pods that satisfy the policy.

Default convention:

```text
secretProviderClass == pod.spec.serviceAccountName
```

This yields a simple model:

```text
Namespace:              platform-aok
ServiceAccount:         aok-source-controller
SecretProviderClass:    aok-source-controller
Allowed backend prefix: /platform-aok/aok-source-controller/
```

Initial policy:

- Enable by default in the `podplane-operator` chart.
- Validate pod `CREATE` and updates that change CSI volumes or `serviceAccountName`.
- For every `secrets-store.csi.k8s.io` CSI volume, require `volumeAttributes.secretProviderClass` to equal `pod.spec.serviceAccountName`.
- Require `pod.spec.serviceAccountName` to be explicitly set and non-empty for any pod that mounts a Secrets Store CSI volume. Do not treat an omitted `serviceAccountName` as Kubernetes' `default` service account for this policy, because that would let workloads mount a `SecretProviderClass` named `default` without making the identity boundary explicit.
- Provide an explicit platform-admin-controlled bypass only when matched by a tightly scoped namespace selector controlled by cluster administrators.

## CLI UX

Use a top-level command group:

```sh
podplane secret
```

The top-level help entry should use the singular command name and a user-facing description such as `secret      Manage application secrets`.

Initial commands:

```sh
podplane secret create KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
podplane secret create KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --stdin
podplane secret update KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
podplane secret update KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --stdin
podplane secret list --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
podplane secret list --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --hide-archived
podplane secret delete KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
podplane secret delete KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --auto-approve
podplane secret delete KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --destroy
podplane secret delete --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --all
podplane secret delete --for SECRET_PROVIDER_CLASS [--provider PROVIDER] --all --destroy
podplane secret restore KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
podplane secret destroy KEY --for SECRET_PROVIDER_CLASS [--provider PROVIDER]
```

`KEY` is positional because it is the object being written or deleted. `--for` is a required scope flag because it selects the workload secret boundary, analogous to `--namespace` selecting the Kubernetes namespace. This avoids two adjacent positional names with different meanings.

`--for` maps to the binding-name portion of the Podplane `SecretProviderKeyspace/<provider-name>.<secret-provider-binding-name>` resource and normally matches `SecretProviderBinding/<secret-provider-binding-name>`. In the Podplane-managed path, the generated `SecretProviderClass` uses the same name. The binding name often matches the Podplane deploy app/release name, but `podplane deploy --name` does not need to change.

Examples:

```sh
podplane secret create github-private-key --for aok-source-controller
podplane secret update github-private-key --for aok-source-controller --provider aws-secrets-manager
cat github-private-key.pem | podplane secret create github-private-key --for aok-source-controller --stdin
cat github-private-key.pem | podplane secret create github-private-key --for aok-source-controller --stdin --dry-run=client -o yaml
podplane secret list --for aok-source-controller --provider aws-secrets-manager
podplane secret list --for aok-source-controller --provider aws-secrets-manager --hide-archived
podplane secret delete github-private-key --for aok-source-controller
podplane secret delete github-private-key --for aok-source-controller --auto-approve
podplane secret delete github-private-key --for aok-source-controller --destroy
podplane secret restore github-private-key --for aok-source-controller
podplane secret destroy github-private-key --for aok-source-controller
podplane secret delete --for aok-source-controller --all
podplane secret delete --for aok-source-controller --all --destroy
```

Behavior:

- `create` prompts securely by default and fails if the key already exists.
- `update` prompts securely by default, requires custom `overwrite` RBAC permission, and fails if the key does not already exist.
- `--stdin` supports automation without shell-history exposure.
- `--dry-run=client -o yaml` encrypts the value with `publickeys/latest` and prints a `SecretProviderKeyspace` manifest without writing it. This only requires `get` on `publickeys/latest`, so a user without write permission can securely pass the encrypted manifest to someone with the matching named `secretproviderkeyspaces` update permission to submit. Submitting an update/overwrite manifest also requires custom verb `overwrite`.
- `--provider` selects a configured backend provider. The CLI must have a cached Podplane cluster summary for the selected Kubernetes context, must verify that an explicit `--provider` is one of the configured provider names in that summary, and must use the summary's cluster ID when constructing encryption associated data. If `--provider` is omitted, the CLI resolves the cluster default provider from the cached summary and sends it explicitly to the API.
- `create` and `update` encrypt the value with the operator's current public key before sending it through the Kubernetes API.
- Successful `create` and `update` should print concise success output by default. Full Kubernetes object output is reserved for explicit `-o yaml` or `-o json`, and encrypted dry-run manifests still default to YAML.
- `create` and `update` fail if the key is archived. Users must either restore the old value or destroy the archived key before creating a new value with the same key.
- If `create` or `update` receives a stale-key response, it refetches the public key and retries once.
- `list` is metadata-only and does not return secret values. It shows archived/restorable keys by default so accidental deletes are discoverable.
- `--hide-archived` filters archived/restorable keys from list output.
- `delete KEY --for SECRET_PROVIDER_CLASS` deletes one backend key from the selected/default provider.
- `delete` prompts before non-restorable deletes. `-y, --auto-approve` skips confirmation prompts, matching existing Podplane commands.
- `delete --destroy` permanently destroys one key where the provider supports archived history; it requires custom `destroy` RBAC permission.
- `delete --for SECRET_PROVIDER_CLASS --all` archives all managed backend keys under that SecretProviderClass boundary and fails if the provider does not support archive delete.
- `delete --for SECRET_PROVIDER_CLASS --all --destroy` permanently destroys all managed backend keys under that SecretProviderClass boundary and requires custom `destroy` RBAC permission.
- `restore KEY --for SECRET_PROVIDER_CLASS` restores an archived key when the selected provider supports restore and the user has both `update` and `restore` RBAC permission on the named `SecretProviderKeyspace`.
- `destroy KEY --for SECRET_PROVIDER_CLASS` is an explicit alias for `delete KEY --destroy`.
- No production `get`/`reveal` in v1.
- Local and production both call the aggregated API.

The CLI should resolve namespace/context like `podplane deploy` and then call the Kubernetes aggregated API using the user's normal kubeconfig credentials.

## Local clusters

Local clusters should use the same API path as production:

```text
podplane CLI -> kube-apiserver -> podplane operator aggregated API -> configured provider adapter
```

Local clusters should configure the operator with the normal Vault/OpenBao adapter pointed at Podplane's local fakevault endpoint. The fakevault endpoint may be backed by host keychain storage, but it should present enough Vault/OpenBao-compatible API surface for the operator's Vault/OpenBao adapter and the Secrets Store CSI provider to use the same provider semantics as production. This preserves local/production parity and avoids local-only branches in `podplane-operator`.

Local fakevault requirements:

- Expose a Vault/OpenBao-compatible HTTPS API under the Podplane local server, scoped by cluster ID, for example `/vault/<cluster-id>/v1/...`.
- Store local secret values in the host OS keychain/keyring, scoped by cluster ID and backend path.
- Support the minimal KV-v2 and auth API surface needed by the OpenBao/Vault CSI provider, the `bao` CLI compatibility test, and the future operator Vault/OpenBao adapter.
- Implement KV-v2-lite soft delete semantics rather than a full Vault/OpenBao version-history engine. This keeps production operator/provider logic clean while giving local clusters the same user-facing `delete`, `restore`, `destroy`, and `list` behavior.
- Authenticate login requests with Kubernetes service-account JWTs from the target local cluster.
- Validate service-account JWTs against the local kube-apiserver JWKS endpoint. If the JWKS endpoint is not anonymous-readable, fakevault may use the submitted service-account JWT as the bearer token to fetch JWKS, then verify that same JWT locally.
- Do not require any special local-only backend adapter logic in `podplane-operator`.

KV-v2-lite fakevault behavior should be intentionally narrow:

- Track one current value map per path, a monotonically increasing version number, and an archived/deleted flag.
- `GET <mount>/data/<path>` returns the current value only when the path is active; archived paths return not found/permission-equivalent behavior so CSI mounts fail the same way they would after production delete.
- Create/update writes a new current value and clears any non-destroyed active state rules according to the operator adapter's create/update contract.
- Soft delete marks the path archived instead of removing the keyring value.
- Undelete clears the archived flag and makes the current value readable again.
- Destroy deletes the stored value and index/metadata entry for the path.
- List includes archived paths with enough metadata for the operator to report `status: archived`; destroyed paths are not listed.
- Do not implement multiple historical versions, CAS, per-version deletion arrays, retention windows, or complete Vault metadata responses unless the operator or CSI provider needs a specific field.

Current status: the local fakevault server portion has been implemented in `github.com/podplane/podplane`. It is keyring-backed, mounted by the local server at `/vault/<cluster-id>/v1/...`, supports the minimal KV-v2/auth API exercised by `bao`, and validates login with real local Kubernetes service-account JWTs. The future work is wiring `podplane secret`, `podplane-operator`, and Secrets Store CSI provider end-to-end flows to use it.

## Security considerations

Secret values are not persisted to Kubernetes. `podplane secret create/update` encrypts values before sending them through the Kubernetes API server, so kube-apiserver and APIService proxy request bodies only contain ciphertext.

Required mitigations:

- Serve the aggregated API over TLS.
- Run the podplane operator as a singleton (`replicas: 1`) while the public/private encryption keypair is in-memory only.
- Keep the operator private key in memory only.
- Configure provider credentials without secret-value read permissions where the provider supports that separation.
- Reject encrypted value writes that reference a stale public key, causing the CLI to refetch and retry once.
- Avoid logging decrypted values in the Podplane operator.
- Avoid logging request bodies even though value-write request bodies contain ciphertext.
- Do not return secret values in normal responses.
- Keep backend IAM policies scoped to Podplane-managed prefixes where possible.

Defense in depth:

- Backend IAM/OpenBao policies should also restrict reads/writes to expected prefixes.
- Backend IAM/OpenBao policies should avoid value-read actions such as AWS `GetSecretValue`, GCP `AccessSecretVersion`, and Vault/OpenBao KV data read for the operator identity where practical.
- `SecretProviderBinding` reconciliation prevents Kubernetes-side read path escape for Podplane-managed mounts.
- Native pod-side CEL admission policy prevents workloads from mounting another app's `SecretProviderClass` in the same namespace.
- Native RBAC controls who may write/delete values.

Duplicate/copy/export/reveal workflows are intentionally out of scope because they require provider value-read permission. If developers no longer have the original value, duplication or cross-provider migration should be handled by a platform team with direct backend access under that organisation's operational controls.

## Vault/OpenBao authentication preference

For Vault/OpenBao-backed clusters, prefer the JWT/OIDC auth method over Kubernetes TokenReview auth when practical.

Preferred order:

1. JWT/OIDC with OIDC discovery if OpenBao can reach the Kubernetes service-account issuer discovery endpoint.
2. JWT with `jwks_url` if only the JWKS endpoint is reachable.
3. JWT with `jwt_validation_pubkeys` if neither endpoint is reachable and automation can configure the service-account public key material.
4. Kubernetes TokenReview auth as an acceptable fallback when direct JWT verification is not practical.

This preference avoids extra Kubernetes API-server load and avoids requiring external OpenBao deployments to reach the full Kubernetes API. Roles should bind projected service-account JWTs using a dedicated audience such as `openbao` and subjects like `system:serviceaccount:<namespace>:<serviceaccount>`.

For local fakevault, the same principle applies in simplified form: validate projected Kubernetes service-account JWTs directly against the local kube-apiserver JWKS endpoint rather than using TokenReview.

## Operator implementation shape

Repository:

```text
github.com/podplane/operator
```

Suggested layout:

```text
cmd/podplane-operator/
internal/secretsapi/
internal/secretsbackend/
internal/secretspolicy/
internal/controllers/
internal/server/
```

Keep the secrets module self-contained so it can be split into a future `github.com/podplane/secrets` repository if needed.

The operator should be a single lightweight Go process that can host:

- aggregated APIs
- `SecretProviderBinding` reconciliation
- health/readiness endpoints

Avoid broad informer caches. Watch only the resources required by enabled modules.

## Split-out path

Start with one operator for resource efficiency:

```text
platform-podplane-operator/platform-podplane-operator
```

Keep `SecretProviderBinding` reconciliation in the same operator for v1. If Podplane Secrets later needs to split into a standalone service, move the self-contained secrets module as a unit rather than introducing an early multi-service design.

If Podplane Secrets later becomes standalone, split by:

1. Moving the self-contained secrets module to a new repo/binary.
2. Deploying a separate `podplane-secrets` service disabled or inactive initially.
3. Disabling the secrets module in `podplane-operator`.
4. Repointing the `APIService` to the new service.
5. Enabling the standalone service.

Only one component should own the `secrets-api.podplane.dev` aggregated API resources and `secrets.podplane.dev` CRD resources at a time.

## Implementation plan

Organize execution by repository so one agent can own each repo's changes concurrently. Within each repository, implementation should come before testing; cross-repo and end-to-end verification is collected at the end of this plan so individual implementation work is not blocked on early test wiring.

### Repository: `github.com/podplane/operator`

This is a new repository for the `podplane-operator` service and the main owner of the aggregated API, backend adapter, and `SecretProviderBinding` controller implementation.

#### Operator scaffolding and runtime

- Create `github.com/podplane/operator` (do not create the actual repository on github, a human will. just work in the already-created directory in `~/Workspace/podplane/operator`), using Kubebuilder/controller-runtime to scaffold the repository, binary, manager wiring, health/readiness endpoints, controller structure, and test harness.
- Build the initial `podplane-operator` Go binary from that scaffold.
- Look at our other podplane repo's like podplane, templates, components, etc. `Makefiles` and try to follow conventions in those for things like help, code formatting, linting, testing, pre-commit hooks, setup, etc.
- Add configuration for the cluster-wide secrets backend prefix, defaulting to the Podplane cluster ID.
- Add TLS serving support from mounted cert files suitable for APIService use.
- Keep the secrets module self-contained using the suggested layout from this spec so it can be split into a future `github.com/podplane/secrets` repository if needed.

#### Aggregated API MVP

- Implement the aggregated API with Kubernetes `k8s.io/apiserver` generic-apiserver machinery, following `kubernetes/sample-apiserver` patterns, not as Kubebuilder webhook routes or a hand-written HTTP handler.
- Include delegated authentication/authorization, authenticated requestheader handling, direct-Service-call rejection, and the required `extension-apiserver-authentication-reader` / `system:auth-delegator` integration.
- Serve complete Kubernetes discovery, including aggregated-discovery responses for supported Kubernetes versions, for cluster-scoped `publickeys` and namespaced, named-only `secretproviderkeyspaces`.
- Implement in-memory X25519 operator keypair generation, 6-hour operator key rotation (6 hours should be default, but configurable), and public key retrieval.
- Implement Kubernetes-style API objects and responses for `publickeys/latest` and named `GET`/`PUT`/`DELETE SecretProviderKeyspace` requests.
- Implement encrypted create/update payload handling: clients fetch `publickeys/latest`, encrypt values with the `x25519-hkdf-sha256-aes-256-gcm` Podplane envelope, send `spec.entries[].operation` in named `PUT SecretProviderKeyspace` requests, and retry once when the server returns a stale-key `409 Conflict`.
- Implement create vs update semantics from `spec.entries[].operation`: `create` creates only missing active keys, `update` overwrites only existing active keys, and both fail when the key is archived.
- Implement archive delete, restore, destroy, and boundary delete/destroy API behavior against the backend adapter interface.
- Implement operator-enforced `SubjectAccessReview` checks for custom verbs: `overwrite` for update, `restore` for restore, and `destroy` for permanent destroy.
- Implement provider-aware metadata-only list through the configured backend adapter, including active/archived status where it can be determined and `unknown` when per-key status lookup fails after the provider list succeeds.
- Define the backend adapter interface around create, update, list, archive delete, restore, destroy, and boundary delete/destroy operations.

#### First backend and local fakevault parity

- Implement the first concrete backend adapter: OpenBao KV-v2 semantics sufficient for the existing local fakevault endpoint.
- Ensure the OpenBao adapter can be configured to point at the local fakevault endpoint without local-only branches in `podplane-operator`.

#### SecretProviderBinding controller

- Add the namespaced `SecretProviderBinding` CRD in API group `secrets.podplane.dev/v1beta1`.
- Implement reconciliation from `SecretProviderBinding` to an owned same-namespace `SecretProviderClass` with the same name.
- Select provider configuration from `spec.providerName` and resolve provider status as `status.provider.name` plus `status.provider.kind`.
- Generate provider-specific `SecretProviderClass` syntax for AWS, OpenBao, Vault, and GCP from `spec.items[]` items and safe provider config.
- Implement `syncToKubernetesSecrets` by translating `data[].fromKey` to upstream `secretObjects[].data[].objectName` using the matching resolved mounted relative path: `spec.items[].path` when set, otherwise `spec.items[].key`.
- Set a controller `ownerReference` from generated `SecretProviderClass` to the owning `SecretProviderBinding`, plus labels/annotations for the binding name.
- Reject collisions by setting a conflict condition when the target `SecretProviderClass` exists without a matching Podplane owner reference or is owned by a different binding UID.
- Do not mutate or validate raw, non-owned `SecretProviderClass` objects; leave them as an RBAC-controlled escape hatch for cluster operators.

#### AWS production backends

- Implement AWS Secrets Manager backend adapter.
- Implement AWS Parameter Store backend adapter.
- Define exact name/tag conventions for `/<cluster-secrets-prefix>/namespace/spc/key` paths.
- Support create, update, metadata-only list, archived-key detection, archive delete, restore, and destroy according to each AWS backend's native semantics.
- For AWS Secrets Manager, implement archive delete as scheduled deletion, restore via `RestoreSecret`, and destroy via force delete without recovery.
- For AWS Parameter Store, make archive delete and restore unsupported with clear errors, and support explicit destroy as parameter deletion.
- Scope operator IAM to Podplane-managed prefixes where possible.

#### Vault, OpenBao, and GCP production backends

- Harden the OpenBao KV-v2 adapter beyond the local fakevault path for production OpenBao deployments.
- Implement Vault KV-v2 adapter.
- Implement GCP Secret Manager adapter.
- For Vault/OpenBao, support create, update, metadata-only list, soft delete of the latest/current version, undelete/restore, and destroy as KV-v2 metadata delete for the logical key; do not expose per-version operations.
- For GCP Secret Manager, support create, update, metadata-only list, delete as disabling the active/latest version, restore as re-enabling that version, and destroy as destroying disabled archived version(s).
- Extend `SecretProviderBinding` rendering for Vault/OpenBao and upstream GCP SecretProviderClass formats; generated resources must avoid GKE managed add-on provider/driver shapes and provider-specific historical version references.

### Repository: `github.com/podplane/components`

This repository owns the platform component chart and component metadata needed to deploy the operator, API aggregation, `SecretProviderBinding` CRD/controller, native pod-side admission policy, Secrets Store CSI Driver, and provider components.

#### `podplane-operator` chart and operator deployment

- Add the `podplane-operator` chart, including the `platform-podplane-operator` Deployment and Service by default.
- Configure the chart to run the operator with exactly `replicas: 1` by default and do not expose autoscaling as a normal v1 path.
- Register `v1beta1.secrets-api.podplane.dev` via `APIService`.
- Ensure APIService availability is gated on serving certificate validity, delegated auth readiness, discovery readiness, and handler readiness; upgrade/uninstall must not leave a stale unavailable APIService behind.
- Add cert-manager `Certificate` and CA injection wiring for the APIService in the chart.
- Add chart RBAC defaults, helper ClusterRoles, and aggregation labels matching the spec; aggregate only `publickeys/latest` read access to `view`, and do not create broad user bindings or a default `aggregate-to-view` role for `secretproviderkeyspaces`.
- Add native RBAC manifests for admin and developer roles, including custom verbs `overwrite`, `restore`, and `destroy`, while relying on the operator to enforce those custom verbs with SAR checks.

#### Components and policy metadata

- Add component metadata for Secrets Store CSI Driver, provider charts, and `podplane-operator`, initially optional/not recommended until functional.
- Add default-on native CEL `ValidatingAdmissionPolicy` enforcing explicit non-empty `serviceAccountName` and `secretProviderClass == serviceAccountName` before or alongside any app template support for mounting Secrets Store CSI volumes. Do not add an operator webhook fallback; for unsupported Kubernetes versions, require an explicit compatibility decision or warning rather than silently weakening the default policy.
- Promote `podplane-operator` and secrets components from optional/not recommended to the platform/recommended components path once the end-to-end feature is functional.

### Repository: `github.com/podplane/templates`

This repository owns Podplane app template Helm charts and template manifest metadata consumed by `podplane deploy`.

#### App template secret mounts

- Add template chart support for mounting secrets via Secrets Store CSI, including rendered `SecretProviderBinding` resources where a template opts into secrets.
- Add deploy/template rendering for AWS Parameter Store vs AWS Secrets Manager based on the selected provider's config, including `kind: aws` and `object_type`, not on the provider name alone.
- Add deploy/template rendering for Vault/OpenBao/GCP providers, including `providerName` on rendered `SecretProviderBinding` resources.
- Add static `podplane-operator` and `secrets-store-csi-driver` dependency metadata to app templates that can render `SecretProviderBinding` resources, without statically depending on every provider-specific CSI plugin.
- Keep CSI Kubernetes Secret sync opt-in only.
- Update template manifests so secret-capable templates expose the required component dependencies and image metadata.

### Repository: `github.com/podplane/seedgen`

This repository owns the seed generator rules used to decide which platform component records belong in minimal and recommended Podplane seeds.

#### Seed generation rules

- Update seed include/exclude rules and expected-record checks as needed when `podplane-operator` and secrets components move into the platform/recommended components path.
- Preserve the minimal seed's lightweight profile unless the secrets platform components are explicitly required there.

### Repository: `github.com/podplane/seeds`

This repository owns the generated Netsy seed snapshots and seed manifests downloaded by Podplane.

Do not do these steps, but check that a human has:

- Regenerate the recommended seed snapshot after the components chart includes `podplane-operator` and the secrets components in the recommended path.
- Regenerate the minimal seed snapshot only if the implementation intentionally moves any secrets-related component into the minimal path.
- Update seed manifests for any regenerated snapshots.

### Repository: `github.com/podplane/podplane`

This repository owns the Podplane CLI, cluster config, local-cluster behavior, dependency download selection logic, deploy UX, and local fakevault integration.

#### Cluster config, summaries, and dependencies

- Add cluster config schema for named secrets providers and a default provider. Provider kind must equal the upstream Secrets Store CSI provider slug: `aws`, `gcp`, `vault`, or `openbao`.
- Add provider name validation and cluster summary fields cached by `podplane login` / `podplane local start`. Cluster config and cached summaries should include only safe fields needed for provider selection and template rendering, such as provider name, kind, AWS `object_type`, and GCP `project_id`/`location`; credentials, tokens, private keys, and other sensitive backend access details must come from operator runtime configuration, workload identity/IAM, or bootstrap mechanisms instead.
- Add `netsyseed` wiring so cluster config provider kinds enable Secrets Store CSI Driver/CRDs and the matching provider components; local clusters always enable the OpenBao provider.
- Update `podplane deps download` provider defaults so `aws`/`gcp`/`vault` include their matching Secrets Store CSI provider artifacts and OpenBao is always downloaded for local clusters.

#### `podplane secret` CLI

- Add `--provider` to `podplane secret` commands, defaulting to the cluster default provider.
- Add `podplane secret` CLI commands that call the aggregated API, including create, update, list, delete, restore, destroy, `--all`, `--destroy`, `--hide-archived`, `--stdin`, and encrypted `--dry-run=client -o yaml` behavior.
- Ensure create/update fetch `publickeys/latest`, encrypt values before sending them through Kubernetes using the `x25519-hkdf-sha256-aes-256-gcm` Podplane envelope, send `spec.entries[].operation` in named `PUT SecretProviderKeyspace` requests, and retry once when the server returns a stale-key `409 Conflict`.
- Ensure list is metadata-only and implements `--hide-archived` filtering client-side or via the API without returning secret values.
- Ensure local `podplane secret create/update` calls the aggregated API, not direct keychain methods.

#### Local fakevault and local-cluster configuration

- Configure local clusters with a default provider that uses the operator's normal OpenBao adapter pointed at the local fakevault endpoint.
- Fill only the fakevault gaps required by the OpenBao adapter: create/update, KV-v2-lite soft delete, undelete/restore, metadata destroy, list, and any SDK response fields required by the adapter or CSI provider.
- Preserve fakevault login with real local Kubernetes service-account JWTs through the already implemented local fakevault auth path.
- Ensure the local Secrets Store CSI OpenBao provider can read the same values through fakevault-compatible API behavior.

#### Deploy and template integration

- Extend Podplane app templates/deploy UX to mount secrets via Secrets Store CSI, defaulting the selected provider from the cluster summary and allowing values-file/`--set` provider selection.
- Ensure `podplane deploy` passes provider selection and safe provider template fields to templates without exposing credentials or sensitive backend access details.

### End-of-work testing and verification

Run repository-local checks after implementation in each repo, then run cross-repo verification once the operator, components, and CLI/deploy work can be composed.

#### `github.com/podplane/operator`

- Add tests for Kubernetes-style API object responses, discovery, named `SecretProviderKeyspace` request handling, stale-key `409 Conflict`, create/update/archive/restore/destroy semantics, boundary delete/destroy behavior, metadata-only listing, and custom-verb `SubjectAccessReview` enforcement.
- Add tests for `SecretProviderBinding` reconciliation, generated `SecretProviderClass` ownership, collision handling, status provider fields, and provider-specific path generation.
- Add AWS-specific rendering tests for generated `objectName`, `objectType`, object aliases, `syncToKubernetesSecrets` translation, and absence of full ARNs, `jmesPath`, `failoverObject`, and historical versions from generated resources.
- Add Vault/OpenBao/GCP rendering tests for supported upstream formats and absence of historical versions and GKE managed add-on shapes from generated resources.
- Validate AWS backend behavior with Secrets Store CSI AWS provider.
- Validate Vault, OpenBao, and GCP backend behavior with their matching Secrets Store CSI providers where practical.

#### `github.com/podplane/components`

- Validate chart rendering for the singleton `platform-podplane-operator` Deployment, Service, APIService, cert-manager `Certificate`, CA injection annotations, `SecretProviderBinding` CRD, RBAC helper roles, aggregation labels, CEL pod policy, and component metadata.
- Verify no chart default creates broad user bindings or an `aggregate-to-view` secrets role.

#### `github.com/podplane/templates`

- Validate template rendering for AWS Parameter Store, AWS Secrets Manager, Vault, OpenBao, and GCP `SecretProviderBinding` resources and CSI volume mounts.
- Verify secret-capable templates depend on `podplane-operator` and `secrets-store-csi-driver` but not every provider-specific CSI plugin.
- Verify CSI Kubernetes Secret sync remains opt-in.

#### `github.com/podplane/seedgen`

- Add or update tests for seed include/exclude rules and expected-record checks covering any recommended-path secrets components.

#### `github.com/podplane/seeds`

- Validate regenerated seed snapshots and manifests.

#### `github.com/podplane/deps`

- Run the dependency mirror build after the source manifests are released or available locally, and verify it publishes updated components, templates, and seeds manifests.

#### `github.com/podplane/podplane`

- Add CLI tests for `--provider` defaulting, create, update, list, delete, restore, destroy, `--all`, `--destroy`, `--hide-archived`, `--stdin`, encrypted `--dry-run=client -o yaml`, and stale-key retry behavior.
- Add config/summary tests ensuring only safe provider fields are persisted and sensitive backend access details are excluded.
- Add dependency and `netsyseed` tests ensuring provider kinds enable Secrets Store CSI Driver/CRDs and the matching provider components, and local clusters always enable/download OpenBao provider support.
- Add focused tests for the local end-to-end path.
- Verify fakevault login with real local Kubernetes service-account JWTs through the already implemented local fakevault auth path.

#### Cross-repo end-to-end

- Stand up a local cluster with the components chart, `podplane-operator`, Secrets Store CSI Driver, and OpenBao provider enabled.
- Verify local `podplane secret create/update/list/delete/restore/destroy` flows go through `podplane CLI -> kube-apiserver -> podplane operator aggregated API -> configured provider adapter`.
- Verify the local Secrets Store CSI OpenBao provider can mount values written by `podplane secret` through fakevault-compatible API behavior.
- Verify `SecretProviderBinding` reconciliation generates backend paths under the expected prefix and avoids historical version references.
- Verify the native CEL pod-side policy prevents a workload from mounting a different service account's `SecretProviderClass`, rejects Secrets Store CSI pods with omitted or empty `serviceAccountName`, and allows the explicit `secretProviderClass == serviceAccountName` convention.
