# Podplane Workload Certificates

> **STATUS**: In progress

This specification builds on [Kubernetes KEP-4317: Pod Certificates](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/4317-pod-certificates), [KEP-3257: ClusterTrustBundles](https://github.com/kubernetes/enhancements/tree/master/keps/sig-auth/3257-cluster-trust-bundles), the [ClusterTrustBundle documentation](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/#cluster-trust-bundles), and the SPIFFE [ID](https://github.com/spiffe/spiffe/blob/main/standards/SPIFFE-ID.md) and [X.509-SVID](https://github.com/spiffe/spiffe/blob/main/standards/X509-SVID.md) specifications.

## Goal and scope

Podplane will issue automatically rotating workload certificates through Kubernetes `podCertificate` projected volumes. A Pod opts in through its spec; it does not need a ServiceAccount token, sidecar, SPIFFE Workload API, Podplane CRD, or registration object to use this functionality.

The `podplane-operator` is the signer and can issue certificates for two purposes:

- a **SPIFFE identity**, derived from the Pod's namespace and ServiceAccount; and
- a **Service identity**, derived from one same-namespace Service that selects the Pod.

A SPIFFE certificate always contains the SPIFFE URI and may also contain authorized Service DNS SANs. A Service certificate contains only those DNS SANs. SPIFFE certificates are X.509-SVIDs; Service certificates are ordinary X.509 serving certificates.

The `podplane-operator` owns this fixed contract:

| Item | Value |
| --- | --- |
| Signer | `certificates.podplane.dev/workload` |
| SPIFFE ID | `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>` |
| Service-identity annotation | `certificates.podplane.dev/service` |
| Certificate-mode annotation | `certificates.podplane.dev/mode` (`spiffe` or `service`; default `spiffe`) |
| ClusterTrustBundle | `certificates.podplane.dev:workload:roots` |
| Maximum leaf lifetime | 24 hours |

Replicas using the same namespace and ServiceAccount receive distinct keys with the same SPIFFE identity; replicas authorized for the same Service likewise share its DNS identity.

KEP-4317 says “the Pod Certificates mechanism is designed for eventual use by in-tree signers to deliver built-in functionality.” This controller is therefore replaceable. Podplane should remove it when a supported Kubernetes signer provides an equivalent configurable trust domain, namespace/ServiceAccount identity, independently selectable Service identity, lifetime, and trust-bundle contract. Kubernetes has not committed to that: the example `kubernetes.io/kube-apiserver-client-pod` signer is unimplemented, and a future signer may require manifest and trust migration or omit Podplane's Service-certificate behavior. Keep this module isolated so replacement removes code rather than wrapping obsolete behavior.

This design does not provide automatic credential injection, a service mesh, transparent mTLS, the SPIFFE Workload API, arbitrary SANs, Web PKI or intermediate CA issuance, external authorization policy, certificate revocation, federation, automatic CA key rotation, or initial KMS/HSM support. Application-managed Pod-to-Pod mTLS is supported.

Runtime issuance requires the `podplane-operator`, Secrets Store CSI Driver, and selected provider; CA-key provisioning does not.

## Workload contract

To give a Pod a SPIFFE certificate, add a `podCertificate` source to a projected volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  namespace: default
  name: pod-certificates-example
spec:
  restartPolicy: OnFailure
  automountServiceAccountToken: false
  containers:
    - name: main
      image: debian
      command: ["sleep", "infinity"]
      volumeMounts:
        - name: spiffe-credentials
          mountPath: /run/workload-spiffe-credentials
          readOnly: true
  volumes:
    - name: spiffe-credentials
      projected:
        sources:
          - podCertificate:
              signerName: certificates.podplane.dev/workload
              keyType: ED25519
              credentialBundlePath: credential-bundle.pem
```

The Pod blocks until initial issuance. Kubelet atomically writes `credential-bundle.pem` as PKCS#8 private key, leaf, then any future intermediates; the self-signed root is omitted. Applications should read the combined file, watch for replacement, and reload after rotation. Separate `keyPath` and `certificateChainPath` files are allowed but can be read across generations. Do not mount either file through `subPath`.

The signer supports all kubelet key types: `RSA3072`, `RSA4096`, `ECDSAP256`, `ECDSAP384`, `ECDSAP521`, and `ED25519`. Examples use Ed25519. Omitting `maxExpirationSeconds` requests the Kubernetes 24-hour default. If the workload does not use the Kubernetes API, disabling ServiceAccount-token automount avoids giving it an unnecessary bearer credential. This is independent of certificate issuance; when requested, the SPIFFE identity uses `spec.serviceAccountName`, including its `default` value.

### SPIFFE certificates and peer trust

An application that verifies certificates from other Podplane workloads also needs the workload CA roots. Add a `clusterTrustBundle` source to the same projected volume:

```yaml
volumes:
  - name: spiffe-credentials
    projected:
      sources:
        - podCertificate:
            signerName: certificates.podplane.dev/workload
            keyType: ED25519
            credentialBundlePath: credential-bundle.pem
        - clusterTrustBundle:
            signerName: certificates.podplane.dev/workload
            labelSelector: {}
            path: trust-bundle.pem
```

The explicit empty `labelSelector: {}` selects all bundles linked to the signer; omitting the selector selects none. `optional` remains false. Kubelet atomically refreshes `trust-bundle.pem`; applications mount the projection directly, not through `subPath`, reload it, and treat certificates as an unordered set.

SPIFFE certificates have `clientAuth` and `serverAuth`, so they work on both sides of Pod-to-Pod mTLS. Each peer must validate the chain, require exactly one SPIFFE URI in the configured trust domain, and authorize that identity. Trusting the CA alone is not authorization.

Ordinary HTTPS clients and Gateway data-plane proxies verify a server by matching the requested hostname against its DNS SANs. A SPIFFE certificate without a Service identity has no DNS SAN, so it cannot verify a server for a Service hostname; request the Service identity for that purpose.

### Service certificates

The default `spiffe` mode may include a Service identity. Add the Service annotation to place both the SPIFFE URI and authorized Service DNS SANs in the certificate:

```yaml
podCertificate:
  signerName: certificates.podplane.dev/workload
  keyType: ED25519
  credentialBundlePath: serving.pem
  userAnnotations:
    certificates.podplane.dev/service: my-service
```

To omit the SPIFFE URI and request a Service certificate containing only DNS SANs, set the mode to `service`:

```yaml
userAnnotations:
  certificates.podplane.dev/mode: service
  certificates.podplane.dev/service: my-service
```

A workload needing separate SPIFFE and Service keys instead projects one unannotated SPIFFE source and one `mode: service` source at distinct paths; kubelet creates and rotates each independently.

`spec.unverifiedUserAnnotations` is workload-controlled. Kubernetes requires these keys to be domain-prefixed, so shorter `mode` and `service` keys are invalid. The signer accepts:

- omitted mode or `certificates.podplane.dev/mode: spiffe`, with an optional Service annotation; or
- `certificates.podplane.dev/mode: service` with a required Service annotation.

Mode `service` without a Service and all other keys or values are invalid. The Service value must be a Kubernetes DNS-label name, not a namespace, slash, qualified name, wildcard, or IP. Invalid input is denied as `InvalidUnverifiedUserAnnotations`; no supplied value is copied into a certificate.

For an annotated request, the operator GETs the PCR's named Pod and requires its UID to equal `spec.podUID`; GETs the named Service in the PCR namespace; and requires a nonempty selector whose every entry matches current Pod labels. The selector may select multiple replicas. Ordinary ClusterIP and headless Services qualify; selectorless and `ExternalName` Services do not. Readiness and EndpointSlice membership are irrelevant because issuance may precede readiness.

For `my-service` in `my-ns`, the exact DNS SAN set is:

```text
my-service
my-service.my-ns
my-service.my-ns.svc
my-service.my-ns.svc.cluster.local
```

The operator derives these names from the fetched Service, trusted PCR namespace, and Podplane's fixed `cluster.local` domain. Short forms are included because TLS checks the literal reference name even when DNS search expansion is used.

Immediately before status update, refetch the Pod and Service. Restart reconciliation if Pod UID or relevant labels, Service UID, or selector changed; ignore unrelated resource-version churn. Issued certificates remain valid after later selector or label changes for at most 24 hours. Immediate invalidation requires relying-party rejection or distrusting the CA.

## Certificate requirements

`cluster.spiffe.trust_domain` is explicit and immutable. It permits lowercase ASCII letters, digits, `.`, `-`, and `_`, with no scheme, path, port, user information, percent encoding, query, or fragment, and is at most 255 bytes. New clusters persist their initial `cluster.kubernetes.api_hostname` as the default; users may choose another globally unique domain they control before creation. Existing clusters must persist a value before enabling issuance. Changing it in place is unsupported because it changes every identity and authorization domain. The operator receives only the resolved value; the signer name is not configurable.

For trust domain `k8s.staging.example.com`, namespace `payments`, and ServiceAccount `api`, the identity is exactly:

```text
spiffe://k8s.staging.example.com/ns/payments/sa/api
```

Namespace and ServiceAccount names are used directly without escaping, aliases, normalization, or overrides. Certificate contents depend on the requested mode and Service annotation:

- **SPIFFE mode:** exactly one URI SAN, zero or four authorized Service DNS SANs, and both `clientAuth` and `serverAuth` EKUs.
- **Service mode:** no URI SAN, exactly the four authorized Service DNS SANs, and only the `serverAuth` EKU.

Every leaf has:

- no IP or email SANs;
- an empty Subject and therefore a critical SAN extension;
- `BasicConstraintsValid=true`, `CA=false`;
- critical Key Usage containing only `digitalSignature`;
- a random, positive, nonzero 128-bit serial;
- exactly the public key from `spec.stubPKCS10Request`; and
- no workload-controlled extensions.

Lifetime means `notAfter - notBefore`; it is at most 24 hours and is further capped by `spec.maxExpirationSeconds` and CA validity. `notBefore` is backdated two minutes within that limit; `beginRefreshAt` is the validity midpoint. Kubelet adds up to five minutes of jitter, creates a fresh key and PCR, and atomically swaps the bundle. The chain is leaf-first and omits the root. Before status publication, verify the certificate contents, key match, signature, ordering, and validity against the loaded CA.

## Kubernetes and signer contract

The baseline is Kubernetes 1.37 or later: core/v1 `podCertificate`, `certificates.k8s.io/v1` `PodCertificateRequest` and `ClusterTrustBundle`, and `spec.stubPKCS10Request` only. These APIs and projections are stable and enabled by default. All API servers and schedulable kubelets must be at least 1.37; there is no older API or feature-gate path. Move `k8s.io/api`, `k8s.io/apimachinery`, and `k8s.io/client-go` together to `v0.37.x` with compatible controller-runtime.

Podplane retains `NodeRestriction` admission and `Node,RBAC` authorization. Do not enable the signer without NodeRestriction.

### Reconciliation

Use a cluster-wide informer filtered server-side by `spec.signerName=certificates.podplane.dev/workload`, then recheck the name. For each PCR:

1. Fetch the latest object; return when absent, for another signer, or terminal (`Issued=True`, `Denied=True`, or `Failed=True`).
2. Parse the DER PKCS#10 `spec.stubPKCS10Request`. Require an empty subject, no SANs, extensions, or extra attributes. Kube-apiserver verifies proof of possession and key parameters but not empty contents.
3. Parse `mode`, default it to `spiffe`, validate it with the optional Service annotation, and revalidate one of the six supported key configurations.
4. In `spiffe` mode, derive the SPIFFE ID only from configured trust domain, PCR namespace, and `spec.serviceAccountName`. In either mode, when the Service annotation is present, authorize the Pod and Service and derive the four DNS SANs.
5. Sign and locally verify the leaf and chain.
6. Refetch the PCR and terminal state. When the Service annotation is present, also refetch authorization inputs and restart if they changed.
7. Update only `/status`.

Success writes all fields atomically:

```yaml
status:
  conditions:
    - type: Issued
      status: "True"
      reason: Issued
      message: Podplane workload certificate issued
      observedGeneration: <pcr generation>
      lastTransitionTime: <now>
  certificateChain: <leaf-first PEM>
  notBefore: <leaf NotBefore>
  notAfter: <leaf NotAfter>
  beginRefreshAt: <validity midpoint>
```

Kubernetes allows only `Issued`, `Denied`, and `Failed`, exactly one terminal `True`, and immutable terminal status. Denied or failed requests contain no other status fields. Stable denial reasons are:

| Reason | Condition |
| --- | --- |
| `UnsupportedKeyType` | Unsupported key type or parameters |
| `InvalidUnverifiedUserAnnotations` | Unknown key, invalid annotation set, or malformed value |
| `ServiceDoesNotSelectPod` | Selectorless Service or selector mismatch |
| `InvalidRequest` | Nonempty or otherwise invalid CSR |

The API server should reject malformed CSRs, signatures, key parameters, annotations, and lifetimes first; the signer still fails closed. A missing Service, relevant object change, API error, CA outage, signing error, or status conflict is transient: leave pending and retry with rate-limited backoff. A stable selector mismatch is denied. Never set `Failed` for an outage because kubelet treats `Failed` and `Denied` as fatal. After conflict, stop if another reconcile completed the PCR; otherwise signing may repeat because it has no external side effect. Count issuance only after status succeeds.

Kubernetes deletes PCRs after roughly 15–20 minutes, so resolve them well before that window and use durable aggregate metrics rather than PCRs as an audit log.

### Trust boundary and RBAC

For node-created PCRs, NodeRestriction verifies requesting Node and UID, Pod name/namespace/UID and scheduling, ServiceAccount name/UID, and that the Pod requested this signer; kube-apiserver verifies the CSR signature. The unannotated path therefore does not GET Pod, ServiceAccount, or Node or repeat proof verification. Do not grant nodes `request-serviceaccounts-podcertificate-signer`.

NodeRestriction does not compare projection `userAnnotations` with PCR `unverifiedUserAnnotations`; a compromised node may alter them. Service selector verification is the additional Service-identity boundary. A non-node principal with PCR `create` can impersonate trusted PCR fields, so Podplane grants creation only through Kubernetes' node role and never aggregates it to user roles. `cluster-admin` remains trusted.

Operator RBAC is:

```yaml
- apiGroups: ["certificates.k8s.io"]
  resources: ["podcertificaterequests"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["podcertificaterequests/status"]
  verbs: ["update", "patch"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["signers"]
  resourceNames: ["certificates.podplane.dev/workload"]
  verbs: ["sign", "attest"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["clustertrustbundles"]
  verbs: ["get", "list", "watch", "create"]
- apiGroups: ["certificates.k8s.io"]
  resources: ["clustertrustbundles"]
  resourceNames: ["certificates.podplane.dev:workload:roots"]
  verbs: ["update", "patch"]
- apiGroups: [""]
  resources: ["pods", "services"]
  verbs: ["get"]
```

The API separately enforces `sign` on PCR status and `attest` on signer-linked trust-bundle writes. `create clustertrustbundles` cannot be name-scoped, but exact `attest` confines signer linkage. The operator has no delete permission and rejects a pre-existing roots object with absent or different signer. Pod and Service access is named GET only; it needs no ServiceAccount, Node, Kubernetes Secret, `SecretProviderBinding`, or external-provider value-read access.

## Dedicated workload CA and key bootstrap

The workload CA is distinct from the Kubernetes cluster/control-plane CA and shared platform CA; using either would merge unrelated trust and compromise domains. Its initial signer is a dedicated Ed25519 workload CA key in `cluster.secrets.default_provider`, delivered only through Secrets Store CSI. The operator signs directly with Go's X.509 implementation and never creates a cert-manager `CertificateRequest` per Pod. cert-manager owns neither this key nor its CA certificate. An intermediate adds no isolation while root and intermediate remain online together.

Every OpenTofu-managed `recommended`, `minimal`, and `none` cluster using AWS Secrets Manager, AWS SSM Parameter Store, or GCP Secret Manager creates or adopts the key before any cluster-specific seed, whether or not the operator is initially installed. When the infrastructure provider owns authorization, Podplane also creates an exact CSI provider-plugin read grant.

Customers using an externally administered Vault or OpenBao KV-v2 backend must provision the key and equivalent read policy before creating the cluster. Podplane consumes it through the customer's existing Kubernetes/JWT integration and requires no Vault/OpenBao token outside the cluster. Installation and operator startup must never generate the key. Reject any other default provider before apply and never fall back to a Kubernetes Secret.

The object contract is:

```text
provider:       cluster.secrets.default_provider
logical key:    workload-ca-key
AWS/SSM name:   /<key-prefix>/workload-ca-key
GCP secret ID:  <key-prefix>_workload-ca-key
Vault/Bao path: <mount-path>/data/<key-prefix>/workload-ca-key
namespace:      platform-podplane-operator
class:          platform-podplane-operator
mount:          /var/run/podplane/certificates/workload-ca-key.pem
format:         one unencrypted PKCS#8 Ed25519 private key
```

The selected provider's `key_prefix`, which defaults to `cluster.id`, isolates the key by cluster. The backend name is derived, never accepted as a user-supplied raw path, and does not encode the current consuming namespace, ServiceAccount, or component. Provider-native encryption protects storage. Plaintext exists only in the provisioning process, the CSI node/mount path, and operator memory. It must never enter generated HCL, schema attributes, plan, state, output, diagnostics, logs, arguments, reusable seeds, support bundles, or temporary files.

### State-safe provisioning

For AWS and Google Cloud backends, `github.com/podplane/terraform-provider-podplane` provides a dedicated resource, not a normal secret-value resource or `tls_private_key`, whose Create:

1. resolves the canonical object;
2. generates the key in memory and uses create-only write if absent on a new cluster;
3. validates and adopts a value left by an interrupted apply; and
4. records only backend identity, immutable provider version metadata, and public-key SHA-256 fingerprint.

Refresh reads metadata only. Update never rewrites or rotates. OpenTofu/Terraform delete removes state but retains the provider value; final retirement is an explicit out-of-band backend deletion only after relying parties stop trusting it. Concurrent creates adopt the single winner or fail without another current value. Once cluster-specific Netsy state or successful provisioning exists, an absent value or changed provider version/fingerprint is key loss and a hard error, never a replacement plan.

For Vault and OpenBao, the customer creates one unencrypted PKCS#8 Ed25519 private key in KV-v2 field `value` at `<mount-path>/data/<key-prefix>/workload-ca-key`. The operator role has read-only access to that exact path. The customer owns backup, restoration, and retirement of the key.

Follow Podplane's existing Secrets Store CSI access path for operator secrets. The provider role used for `podplane-operator` mounts must be able to read the exact workload CA key; the operator receives the mounted file rather than provider credentials.

The `podplane-operator` Helm chart contains `SecretProviderClass/platform-podplane-operator` in namespace `platform-podplane-operator` and applies it before the Deployment, whether the chart is part of the initial seed or installed later. Do not use a `SecretProviderBinding`: the operator that reconciles it cannot start until this key is mounted. The operator ServiceAccount has the same name, satisfying existing admission policy, and the class mounts only `workload-ca-key`. Do not grant general users create or update access to raw `SecretProviderClass` objects; workload bindings remain limited to generated provider paths, and only this managed class may reference the reserved key. Start the singleton operator with `Recreate` only after CSI driver and provider readiness; missing key, class, grant, or provider blocks volume setup with no fallback.

### CA lifecycle and readiness

Whenever the operator starts or the mounted key file changes, it reads `workload-ca-key.pem`. The file must contain exactly one PEM block encoding an unencrypted PKCS#8 Ed25519 private key, with no certificate or additional key.

On first startup, after validating the key, the operator creates a ten-year self-signed certificate with Common Name `Podplane Workload CA`, valid CA constraints, and certificate-signing Key Usage, and publishes it before issuing leaves. The Common Name is diagnostic only.

After that first publication, every key load requires matching certificates in the ClusterTrustBundle; every match must be self-signed, have valid CA constraints, and permit certificate signing. Select the match with latest `notAfter`, requiring at least 24 hours plus two minutes of remaining validity.

One year before expiry, create a new ten-year self-signed certificate with the same key and Subject, publish old and new together, then switch issuance. Retain the old certificate for at least maximum leaf lifetime plus kubelet, application, and external-cache margin. This renews the public certificate without rotating the key.

Keep the last known-good signer during incomplete or invalid CSI updates. A candidate becomes active only after bundle publication. Publication failure keeps the old signer active, or prevents first issuance. Once a bundle exists, reject a different mounted public key unless deliberate manual rollover authorizes it.

Certificate-signer readiness requires a valid active pair with sufficient lifetime and the exact retained root set published. Candidate-load failure does not make that signer unready; any ClusterTrustBundle reconciliation failure or drift does. Overall operator readiness includes certificate-signer readiness when this module is enabled. A node hosting the operator and a cluster administrator remain able to access mounted key material under the normal Kubernetes trust model.

### Trust-bundle publication and key replacement

The source of truth is:

```yaml
apiVersion: certificates.k8s.io/v1
kind: ClusterTrustBundle
metadata:
  name: certificates.podplane.dev:workload:roots
spec:
  signerName: certificates.podplane.dev/workload
  trustBundle: |
    <current and retained prior root certificates>
```

The name follows the signer-linked `/` to `:` rule; `signerName` is immutable. The API accepts only nonduplicate PEM CA certificates with valid CA constraints. This public object is intentionally readable by all ServiceAccounts. The controller maintains one stable object, while signer-based projection remains compatible with temporary additional bundles; kubelet merges, canonicalizes, and deduplicates them without preserving order.

External administrators export this same bundle. Trust establishes issuer, not SPIFFE or DNS authorization.

Automatic private-key rotation is forbidden because the operator cannot coordinate every relying party. Planned replacement publishes both anchors, updates and confirms relying parties, deliberately switches key and signing generation, waits at least maximum leaf lifetime plus kubelet/application/external-cache margin, then removes the old anchor. Use provider versioning, recovery, backup, and deletion protection for the key and cluster-state backup for the public bundle.

### Platform serving certificates and CA injection

The combined `podplane-operator` process cannot consume a `podCertificate` issued by itself: kubelet waits for the projected certificate before starting the Pod, but the signer cannot issue it until that Pod starts. Until the certificate controller is split into an independently deployable process, the operator directly issues and rotates two fixed Service certificates from its mounted workload CA: one for the aggregated API Service and one for registry authentication when enabled. The operator accepts only each Service's name and namespace and derives its four canonical Kubernetes DNS names; it does not accept arbitrary SANs. Each certificate has only the server-authentication EKU, is written to its own runtime volume, lasts at most 24 hours, and rotates before expiry. Startup must publish the CA and write valid serving keypairs before either HTTPS listener starts. cert-manager is not involved.

The operator also reconciles `certificates.podplane.dev/inject-ca-from: workload` on these cluster-scoped API extension resources:

- `APIService` objects;
- `MutatingWebhookConfiguration` and `ValidatingWebhookConfiguration` objects; and
- CRD webhook conversion configurations.

The annotation value is an exact, closed Podplane CA-source alias, not a Kubernetes signer name or arbitrary object reference. The only supported value is `workload`, selecting the authoritative workload CA bundle. The operator lists the supported resource kinds, processes only annotated objects, requires every endpoint to reference an existing in-cluster Service, copies the authoritative ClusterTrustBundle into every applicable `caBundle`, and overwrites drift. RBAC grants `get`, `list`, and `patch` for the supported API-extension resource kinds and `get` for their Services. Injector failure does not stop certificate issuance, but it makes injector and overall operator readiness false. Adding another CA source requires an explicit code change; adding another integration requires only the annotation. CA injection grants neither identity nor signing authority: Service certificate issuance remains independently authorized against the requesting Pod and referenced Service.

The Cluster API webhook controller projects a `mode: service` Pod certificate authorized for `capi-webhook-service`, mounts it at its existing webhook serving path, and places the workload-CA annotation on its admission webhook configurations and conversion-webhook CRDs. It has no cert-manager Issuer, Certificate, or cert-manager CA-injection annotation.

Nstance Operator projects a Podplane Service certificate for its webhook Service at its existing serving path and places the workload-CA annotation on its validating webhook configuration. It has no cert-manager Issuer, Certificate, Secret volume, or cert-manager CA-injection annotation.

## Gateway backend trust

This optional integration applies only to Gateway-to-backend TLS when the certificate includes a Service identity, whether its mode is `spiffe` or `service`; it does not apply to incoming client mTLS or URI authorization. A Podplane-managed `BackendTLSPolicy` sets `spec.validation.hostname` to an issued Service name, using `<service>.<namespace>.svc.cluster.local` by default, and uses the Podplane workload CA roots as CA source. A SPIFFE certificate without Service DNS SANs will fail hostname verification.

Envoy Gateway 1.9 directly supports a `ClusterTrustBundle` in `BackendTLSPolicy.spec.validation.caCertificateRefs`. The reference uses an empty group, kind `ClusterTrustBundle`, and name `certificates.podplane.dev:workload:roots`, matching Envoy Gateway's tested API contract. Podplane publishes the object through stable `certificates.k8s.io/v1`. Envoy Gateway 1.9 watches the still-served `certificates.k8s.io/v1beta1` representation; Kubernetes 1.37 serves both versions from the same stored object, so no compatibility mirror is required.

## Security and failure behavior

- Kubelet generates workload keys and retains them in node memory and projected tmpfs. They never enter PCRs, etcd, the CA object, or operator. A compromised Pod or node can still extract its key; there is no hardware non-exportability.
- Namespace and ServiceAccount name are the identity; replicas intentionally share it. Pod/Node names and UIDs and ServiceAccount UID are validated context only. Recreating a same-named ServiceAccount recreates the logical identity.
- Anyone able to create a Pod under a ServiceAccount can request its SVID. Anyone able to create a matching Pod or temporarily alter a Service selector can obtain that Service identity for the leaf lifetime. Kubernetes workload RBAC and relying-party policy are the authorization boundaries.
- The operator ignores CSR identity content and denies all workload annotations except the validated mode and Service annotations. The workload CA limits compromise to SPIFFE and Service certificate identities.
- ClusterTrustBundles are public; the CA key is not. Only the CSI plugin may read the exact external value. CA private-key bytes must never enter Kubernetes status, configuration, ConfigMaps, or another public API object; never expose key or certificate payloads through logs or metrics.
- Initial signer/API failure blocks Pod volume setup. Refresh failure retains the previous valid bundle; midpoint refresh normally leaves 12 hours for recovery. Denial is fatal and requires a corrected new Pod.
- Missing Service remains pending; stable selector mismatch is denied. Changes after issuance affect only new/refresh requests. Expiry breaks authentication but does not terminate the container.
- Kubelet restart loses its in-memory keys and may cause an issuance burst. Operator restart reloads key and public certificate; existing workload bundles remain valid.
- Invalid CSI updates retain the last signer. Missing external key blocks bootstrap and requires restoration, never generation. Missing initial trust bundle blocks its projection; update failure retains last projected content.
- A missing or invalid workload ClusterTrustBundle leaves the Gateway backend policy unresolved or preserves its last valid dynamic configuration.

## Operations

Expose bounded-cardinality metrics:

```text
podplane_workload_certificate_pcr_reconciliations_total{result="issued|denied|error",reason="...",key_type="...",mode="spiffe|service"}
podplane_workload_certificate_pcr_pending
podplane_workload_certificate_pcr_terminal_update_conflicts_total
podplane_workload_certificate_ca_not_after_seconds
podplane_workload_certificate_ca_load_errors_total
podplane_workload_certificate_trust_bundle_reconcile_errors_total
```

Derive `mode` only from the validated annotation set. Never label Pod, namespace, Service, ServiceAccount, SPIFFE ID, PCR, serial, DNS name, or certificate. Logs may include namespaced PCR, requested Service, and stable result/reason, but no CSR, certificate, key, mounted value, or sensitive provisioning diagnostic.

Also monitor `apiserver_resource_objects` and kubelet `kubelet_pod_certificate_states`. Alert on signer unready; pending/error for five minutes; overdue/expired projections; denial spikes; CA-load or bundle errors/drift; and less than one year of CA validity.

Export public roots with:

```sh
kubectl get clustertrustbundle certificates.podplane.dev:workload:roots \
  --output 'jsonpath={.spec.trustBundle}'
```

## Implementation order

1. **`github.com/podplane/podplane`**: add and validate immutable `cluster.spiffe.trust_domain`; persist the initial API hostname default; provide safe AWS/GCP/Vault/OpenBao secret-provider defaults and reject unsupported providers; generate the state-safe key resource, canonical identity, provider-specific exact CSI-plugin policy, and seed dependency where Podplane owns provisioning; require externally administered Vault/OpenBao keys to exist before cluster creation; render bootstrap class values when the operator is selected initially or later.
2. **`github.com/podplane/terraform-provider-podplane`**: implement state-safe creation or adoption of the workload CA key in AWS Secrets Manager, AWS SSM Parameter Store, and GCP Secret Manager; metadata-only refresh, retain-on-delete, loss detection, and no schema/data-source/import/debug path exposing the value.
3. **`github.com/podplane/vmconfig`**: establish Kubernetes 1.37 and verify stable PCR and ClusterTrustBundle APIs and projections without feature gates.
4. **`github.com/podplane/operator`**: update Kubernetes libraries; add isolated signer configuration, CSI loading/readiness, CA renewal, filtered PCR and trust-bundle controllers, certificate and Service authorization, fixed operator-serving certificates, annotation-driven workload CA-bundle injection, metrics, and tests; add no provider-value clients.
5. **`github.com/podplane/components`**: render the bootstrap class, read-only mount, signer/CTB and CA-injector RBAC, fixed operator-serving runtime volumes, Cluster API and Nstance projected serving certificates, and direct Envoy Gateway ClusterTrustBundle references; keep provider identity off the operator; order CSI driver/provider and operator before dependent components. Render no workload CA Secret, cert-manager workload CA resources, or trust-manager dependency. Remove cert-manager and trust-manager after migrating their remaining consumers. Envoy Gateway and operator-owned ingress certificates supersede the abandoned `platform-acme` transition; see [ACME.md](./ACME.md).
6. **`github.com/podplane/seedgen`**: generate the non-secret resources and RBAC required when the recommended seed selects the operator; keep key provisioning outside generic seed records.
7. **`github.com/podplane/seeds`**: publish the generated operator resources in the recommended seed and prove reusable seed artifacts contain no CA private key or external secret value.
8. **`github.com/podplane/templates`**: replace cert-manager CSI and Secret-backed workload certificates with Kubernetes 1.37 `podCertificate` and `clusterTrustBundle` projections; use separate Service and SPIFFE identities where both are required, and reference the workload ClusterTrustBundle directly from Envoy Gateway backend policies.

## Verification

Unit, provider, and chart tests must cover:

- signer/trust-domain validation, identity mapping, all six key types, empty CSR enforcement, all valid and invalid mode/Service annotation combinations, exact SAN and EKU requirements for both modes, lifetime, chain, refresh, and status conditions;
- Service Pod/UID lookup, same-namespace selector authorization, replicas, headless/selectorless/`ExternalName`, absent-Service retry, Pod-label and Service-selector races that discard candidates before status publication, fixed DNS names, and named-GET RBAC;
- reconciliation terminal states, conflicts, duplicate events, outages, deletion, filtered watch, `sign`/`attest`, stable denial reasons, and least privilege;
- all CA load, initial publication, same-key renewal, last-known-good, mismatch, expiry, CTB create/update/conflict/retention/deduplication, and readiness paths;
- fixed operator serving-certificate bootstrap and rotation, separate aggregated-API and registry identities, annotation-driven workload CA injection with Service validation and drift repair, injector RBAC/readiness, and Cluster API and Nstance webhook startup without cert-manager;
- unconditional key provisioning for every seed choice, all provider create/adopt races, metadata-only state, retain/loss behavior, exact backend naming and plugin grant, delayed installation, and proof that private bytes never reach forbidden artifacts;
- Envoy Gateway hostname acceptance in either mode with a Service identity, rejection without Service DNS SANs, direct ClusterTrustBundle consumption, and updates/failure/removal; and
- bootstrap class/mount/order, absence of Kubernetes or cert-manager CA objects and operator provider credentials, raw-class prohibition, bounded metrics, and secret-free reusable seeds.

An integrated Kubernetes 1.37 test must prove:

1. `recommended`, `minimal`, and `none` managed clusters all provision the external key and plugin grant without plaintext in OpenTofu artifacts or state; later `podplane install` on `minimal` consumes that original key.
2. Initial Pod blocking with ServiceAccount-token automount disabled, bundle/key match, exact SVID requirements, stable v1 APIs, trust projection including omitted-versus-empty selector behavior, and clear readiness failure when APIs are absent.
3. Pod-to-Pod mTLS authorizing exact URI SANs, external client mTLS, and rejection by the unrelated platform CA.
4. Mode `spiffe` with and without a Service identity, mode `service`, and separately projected SPIFFE and Service certificates; prove `service` mode omits the URI, `spiffe` mode retains it when Service DNS SANs are added, both modes contain exactly four DNS SANs when a Service is requested, separate files have independent matching keypairs, and malformed, additional, cross-namespace, selectorless, mismatched, absent, and racing Service cases behave correctly with bounded validity after authorization changes.
5. Refresh with a new key, atomic application reload, signer restart and API outage, CA certificate overlap, trust reload, and no mismatched generations.
6. Envoy Gateway acceptance in either mode with the annotated Service FQDN, rejection of a SPIFFE certificate without Service DNS SANs, and direct trust of the stable workload ClusterTrustBundle through Envoy Gateway's supported API view.
7. Annotation inability to change URI or inject SANs, forbidden unprivileged PCR creation and raw key class, exact-key CSI success, and inability of the operator identity to read provider values.

Before setting `Implemented`, repeat the integrated path on the supported Kubernetes version, verify every owning repository's relevant tests, and scan generated seeds and artifacts for secret material.
