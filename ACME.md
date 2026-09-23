# Podplane Ingress Certificates and ACME

> **STATUS**: In progress

Related: [CERTS.md](./CERTS.md), [DOMAINS.md](./DOMAINS.md)

## Goal

Podplane provides an apex-and-wildcard serving certificate for every configured cluster domain without persisting private keys in Kubernetes Secrets or etcd. The Podplane operator always provides an immediately usable self-signed fallback and, when `cluster.acme` is configured, replaces it with an ACME DNS-01 certificate for supported DNS providers.

Initial public issuance supports AWS Route53 through Lego. Google Cloud DNS, Azure DNS, Cloudflare, and additional Lego providers are future increments behind an explicit Podplane provider allowlist.

## Ownership

`github.com/podplane/operator` owns key generation, fallback issuance, ACME account and order lifecycle, renewal, validation, external publication, readiness, and metrics.

`github.com/podplane/podplane` owns user configuration, validation, generated infrastructure identity, and rendering operator runtime configuration into the cluster seed.

`github.com/podplane/components` owns the operator chart, Envoy Gateway, and the ingress-certificate delivery adapter. Envoy Gateway 1.9 is selected. The adapter described below is implemented in the current work and awaits end-to-end cluster verification.

## User configuration

`cluster.acme` enables public ACME certificates:

```jsonc
{
  "cluster": {
    "acme": {
      "server": "https://acme-v02.api.letsencrypt.org/directory",
      "email": "ops@example.com"
    },
    "domains": [
      {
        "zone": "staging.example.com",
        "provider": {
          "kind": "aws-route53",
          "hosted_zone_id": "Z123456789"
        }
      }
    ]
  }
}
```

The server defaults to the Let's Encrypt production directory. ACME configuration requires a valid account email, an HTTPS server URL when overridden, and at least one domain with a supported DNS provider. Enabling ACME accepts the selected ACME server's subscriber agreement.

`provider.hosted_zone_id` is optional in user configuration. Generated OpenTofu/Terraform resolves omitted Route53 zone IDs and passes the result and generated role ARN through the seed's provider-neutral `values_content`; generated values do not mutate or persist into the cluster config.

When `cluster.acme` is absent, all domains retain self-signed fallback certificates. When it is present, only supported domains attempt ACME; manual and unsupported domains retain their fallbacks.

## Certificate contract

The operator manages one independently failing certificate pipeline per domain. Each leaf contains exactly:

```text
staging.example.com
*.staging.example.com
```

Each pipeline follows these invariants:

1. Load and cryptographically validate the current externally stored combined PEM bundle.
2. If no valid bundle exists, generate a fresh P-256 key and a 30-day self-signed server certificate and publish it before attempting ACME.
3. If ACME is enabled for the domain, use Lego DNS-01 to obtain a bundled public chain with a fresh P-256 leaf key.
4. Validate private-key correspondence, exact SANs, server-authentication EKU, validity period, and chain signatures before publication.
5. Replace the complete combined certificate-chain-and-private-key value in one provider operation. Never split a generation across backend objects.

The self-signed leaf key is never reused for ACME. Every ACME issuance and renewal uses a fresh leaf key. The stable ACME account key is a separate value and is never reused as a leaf key.

## External state

The configured default Podplane Secrets provider stores operator-owned state under this logical keyspace:

```text
/<key-prefix>/platform-cluster/ingress-certificates/
```

It contains:

- one stable ACME account envelope per ACME directory, containing the P-256 account key and registration resource; and
- one complete combined PEM value per domain, addressed by a deterministic hash of the normalized apex.

The backend's native current-version transition is the publication boundary. AWS Secrets Manager, SSM Parameter Store, Google Secret Manager, and Vault/OpenBao implementations read the current provider version and replace the whole value. Provider versioning, recovery, replication, and deletion protection remain operator responsibilities.

For AWS clusters, generated instance-role policy permits only the operations needed to create, read, and replace values beneath the exact ingress-certificate keyspace. The workload CA key remains covered by its separate read-only statement. No ingress permission grants access to that key or to unrelated Podplane Secrets keyspaces.

Neither account nor leaf private-key bytes may enter Kubernetes configuration, Secrets, status, events, logs, metrics, Terraform state, or the Podplane Secrets API. The Envoy delivery adapter requests only the exact domain bundle object; it does not mount the account object or enumerate the keyspace.

## Route53 identity

Generated OpenTofu/Terraform creates a dedicated ACME Route53 role. The role:

- permits only Route53 TXT changes and change-status reads required by DNS-01;
- scopes changes to the Terraform-resolved hosted zones and permits only `UPSERT` and `DELETE` actions; and
- is assumable by the cluster VM identity.

The generated operator config passes the role ARN, region, and exact hosted-zone ID to Lego's typed Route53 provider. Lego uses ambient AWS credentials only to assume that role. It does not use arbitrary dynamic provider loading or process-global credential mutation. Explicit zone IDs avoid Route53 zone-list permissions.

## Envoy Gateway SDS delivery

Each Gateway listener references a core Secret in the Gateway namespace. That Secret is a pointer marked with `gateway.envoyproxy.io/sds`; its `url` and `secretName` data fields contain only SDS routing metadata, never certificate or private-key bytes. Envoy Gateway 1.9's upstream SDS Secret-reference API is extension-gated, so Podplane explicitly enables that extension and treats failure to recognize the reference as a listener-readiness failure rather than falling back to an ordinary TLS Secret.

The private combined PEM bundle remains in the configured external Secrets backend. Secrets Store CSI mounts only the exact per-domain bundle into each Envoy data-plane Pod. A hardened Podplane SDS sidecar in that Pod reads the mount and serves the certificate to the co-located Envoy process over a Pod-local Unix domain socket. The socket is carried on a shared in-memory volume and is inaccessible over the Pod network. The sidecar runs as non-root with a read-only root filesystem, no privilege escalation, all Linux capabilities dropped, and read-only access to the certificate mount; Envoy receives neither external-provider credentials nor direct access to that mount.

The sidecar validates key correspondence, exact apex-and-wildcard SANs, chain signatures, validity, and server-authentication usage before publishing an SDS generation. It watches CSI's atomic mount updates, sends a complete validated generation over SDS, and retains the last-known-good generation across malformed, partial, or temporarily unavailable updates. Envoy hot-reloads successful SDS updates without a data-plane Pod restart. Initial invalid or absent material prevents the sidecar from starting and leaves the listener unready; later failures preserve the prior valid listener and emit bounded diagnostics without certificate or key material.

The Envoy Gateway control plane sees only the pointer Secret metadata and SDS configuration. Ingress private-key bytes must never enter etcd, xDS resources emitted by the Envoy Gateway control plane, control-plane memory, logs, events, or status. They exist only in the external backend, CSI/provider and node mount path, SDS sidecar memory, the UDS exchange, and Envoy data-plane memory.

## Renewal and failure behavior

- The operator checks certificates hourly and attempts ACME replacement or renewal 30 days before public-certificate expiry.
- A fallback rotates seven days before its 30-day expiry.
- A transient ACME, DNS, account, validation, or publication failure preserves the current valid bundle.
- If a public certificate expires before renewal succeeds, the operator atomically publishes a fresh self-signed fallback and reports the ACME failure.
- Failure for one domain does not block reconciliation of another domain.
- A missing or unreadable external backend blocks initial fallback publication and operator readiness. A valid fallback keeps readiness true during ACME failures.
- Logs and metrics identify only the configured domain and failure stage; they never contain account, challenge, certificate, or private-key payloads.

## Cluster-create wizard

When the user configures a Route53-managed domain, the wizard prompts for the optional ACME account email:

- an email enables ACME using the Let's Encrypt production directory; and
- blank input keeps self-signed ingress certificates.

The wizard does not ask about ACME server selection, solver details, IAM roles, certificate lifetime, renewal windows, account keys, or leaf keys.

## Implementation state and order

1. **Implemented in the current work:** validate `cluster.acme`, default the server, resolve Route53 zones, and generate a least-privilege provider-independent ACME role.
2. **Implemented in the current work:** render proxy-neutral ingress certificate configuration into the Podplane operator chart.
3. **Implemented in the current work:** persist and read operator-owned state from supported Podplane Secrets backends without exposing reads through the aggregated Secrets API.
4. **Implemented in the current work:** publish and rotate validated self-signed fallback bundles per domain.
5. **Implemented in the current work:** use pinned Lego Route53 DNS-01 for ACME account registration, issuance, key rotation, renewal, and failure retention.
6. **Implemented in the current work:** configure Envoy Gateway 1.9's extension-gated upstream SDS Secret-reference API and metadata-only pointer, per-Pod CSI mount, hardened SDS sidecar, and UDS boundary specified above.
7. **Implemented in the current work:** validate initial material and hot rotations, retain the last-known-good generation, and leave listeners unready when no valid initial generation exists.
8. **Implemented in the current work:** remove the transitional `platform-acme`/cert-manager ingress path, Kubernetes TLS Secrets, bootstrap hooks, and issuer/certificate resources.
9. Update user documentation and advance this spec only after the public ingress endpoint serves and renews the externally stored certificate and integrated checks prove private material never enters etcd or the Envoy Gateway control plane.

## Future work

- Add typed Lego adapters and least-privilege identities for Google Cloud DNS, Azure DNS, Cloudflare, and additional providers.
- Support cross-account Route53 zones and explicit provider references.
- Add external account binding where required by a configured ACME server.
- Add staging-specific wizard UX.
- Evaluate configurable lifetime and renewal windows after operational evidence warrants them.
