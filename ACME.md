# Podplane ACME Certificates

## Goal

Podplane uses cert-manager and ACME DNS-01 challenges to issue publicly trusted certificates for each configured cluster domain. Initial implementation supports Route53-managed DNS only and builds on the domain model in [DOMAINS.md](./DOMAINS.md).

Domains using manual DNS, or a DNS provider without implemented ACME support, continue using Podplane's self-signed ingress issuer.

## Existing implementation

Podplane already provides:

- cert-manager, approver-policy, platform-certs, and Traefik in the recommended seed;
- an optional ACME `ClusterIssuer` with Route53, Cloudflare, and Cloud DNS solver rendering;
- one cert-manager `Certificate` per domain covering its apex and wildcard;
- Traefik Gateway TLS attachment and normal cert-manager renewal;
- a self-signed ingress issuer; and
- temporary 30-day bootstrap certificates while the configured issuer becomes ready.

Netsyseed already converts `cluster.acme` and `cluster.domains` into platform-certs and Traefik values. The main missing work is Route53 identity provisioning, per-domain issuer selection, and cluster-create orchestration.

## Configuration

`cluster.acme` enables public ACME certificates:

```jsonc
{
  "cluster": {
    "acme": {
      "email": "ops@example.com"
    },
    "domains": [
      {
        "zone": "staging.example.com",
        "provider": {
          "kind": "aws",
          "hosted_zone_id": "Z123456789"
        }
      }
    ]
  }
}
```

The ACME server defaults to the Let's Encrypt production directory. `acme.server` remains an advanced override for users who need another ACME server; staging-specific UX is out of scope.

When `cluster.acme` is absent, or a domain has no supported DNS provider, that domain uses the self-signed ingress issuer. ACME configuration must not leave a domain without a usable certificate path.

## Route53 identity

Generated Terraform creates a dedicated cert-manager DNS role for each required AWS account. The role:

- permits only the Route53 lookup and TXT-record changes needed for DNS-01;
- scopes record changes to the configured hosted zones;
- can be assumed through kube2iam by the IAM role attached to cluster VMs via Nstance's agent instance profile.

The cert-manager controller receives the role through the existing kube2iam pod annotation. Its namespace restricts allowed kube2iam roles to the generated DNS role. Generated Terraform passes the role ARN and namespace restriction into seeded platform-components values without persisting credentials or generated ARNs in cluster config. The Route53 solver uses those ambient credentials and the configured hosted-zone ID.

## Issuance behavior

For each ACME-enabled domain, cert-manager requests one certificate containing:

```text
staging.example.com
*.staging.example.com
```

Traefik continues serving its bootstrap certificate until cert-manager writes a valid ACME certificate to the same TLS Secret. Thereafter cert-manager owns renewal and private-key rotation.

An ACME issuance failure does not silently switch issuer configuration. The bootstrap certificate remains available while cert-manager reports the failure through normal Certificate, Order, Challenge, and ClusterIssuer status.

Certificate approval policy should restrict ingress requests to the configured apex and wildcard names rather than allowing arbitrary DNS names from the Traefik namespace.

## Cluster-create wizard

When the user configures a Route53-managed domain, the wizard prompts for the optional ACME account email:

- an email enables ACME using the Let's Encrypt production directory;
- blank input keeps self-signed ingress certificates.

The wizard does not ask about ACME server selection, solver details, IAM roles, or certificate lifetime.

## Implementation plan

1. Default an omitted `cluster.acme.server` to the Let's Encrypt production directory and validate ACME email/domain/provider combinations.
2. Generate least-privilege Route53 IAM roles and kube2iam controller annotations.
3. Restrict the cert-manager namespace to the generated kube2iam role.
4. Pass generated role ARNs into Netsyseed/platform-components values without writing them into cluster config.
5. Add per-domain issuer selection so unsupported or manual DNS domains remain self-signed while Route53 domains use ACME.
6. Ensure Route53 solvers use ambient kube2iam credentials and configured hosted-zone IDs.
7. Correct the platform-components example to use `platform.certs.ingress.acme`.
8. Restrict ingress certificate approval policy to configured domain names.
9. Add config, tfgen, Netsyseed, Helm-rendering, IAM-policy, bootstrap-transition, mixed-domain, issuance, and renewal tests.

## Future work

- Cross-account Route53 role provisioning and provider references; initial support assumes the hosted zone is in the cluster AWS account.
- Cloudflare and Google Cloud DNS solvers and credentials.
- ACME staging-specific wizard UX.
- Other ACME servers and external account binding.
- Explicit certificate lifetime and renewal-window controls.
