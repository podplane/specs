# Podplane ACME Certificates

## Goal

Podplane uses cert-manager and ACME DNS-01 challenges to issue publicly trusted certificates for supported cluster domains. Initial implementation supports AWS Route53-managed DNS only and builds on the domain model in [DOMAINS.md](./DOMAINS.md).

Domains using manual DNS, or a DNS provider without implemented ACME support, continue using Podplane's self-signed ingress issuer.

## Existing implementation

Podplane already provides:

- cert-manager, approver-policy, platform-certs, and Traefik in the recommended seed;
- an optional ACME `ClusterIssuer` with Route53, Cloudflare, and Cloud DNS solver rendering;
- one cert-manager `Certificate` per domain covering its apex and wildcard;
- Traefik Gateway TLS attachment and normal cert-manager renewal;
- a self-signed ingress issuer; and
- temporary 30-day bootstrap certificates while the configured issuer becomes ready.

Netsyseed converts `cluster.acme` and `cluster.domains` into platform-certs and Traefik values. Solver-rendering plumbing for the `cloudflare` and `google-cloud-dns` providers remains in place for future use, but only `aws-route53` currently enables ACME.

## Configuration

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

The ACME server defaults to the Let's Encrypt production directory shown above, so `acme.server` may be omitted. It remains an advanced override for users who need another ACME server; staging-specific UX is out of scope.

`provider.hosted_zone_id` is an optional explicit Route53 hosted zone ID. Use it to disambiguate or pin the exact hosted zone; otherwise, generated OpenTofu/Terraform looks up the public hosted zone by domain name. OpenTofu/Terraform passes the resolved zone ID into the cluster seed before cert-manager's solver configuration is rendered.

When `cluster.acme` is absent, every domain uses the self-signed ingress issuer. When ACME is configured, supported domains use ACME while manual domains and domains whose provider does not yet support ACME continue using the self-signed issuer. ACME configuration requires a valid account email, an HTTPS server URL when overridden, and at least one domain using a supported ACME DNS provider.

## Route53 identity

Generated OpenTofu/Terraform creates a dedicated cert-manager Route53 role in the cluster AWS account. Initial support assumes each configured hosted zone is in that account. The role:

- permits only the Route53 TXT-record changes and change-status reads needed for DNS-01;
- scopes record changes to the Terraform-resolved hosted zones and restricts them to `UPSERT` and `DELETE` actions;
- can be assumed through kube2iam by the IAM role attached to cluster VMs via Nstance's agent instance profile.

The cert-manager controller receives the role through the `iam.amazonaws.com/role` kube2iam pod annotation. Its namespace restricts allowed kube2iam roles to the generated role through `iam.amazonaws.com/allowed-roles`. Generated Terraform passes the role ARN and resolved hosted-zone IDs through the seed resource's provider-neutral `values_content` overlay without persisting credentials or generated ARNs in cluster config. An optional user-authored `values_file` is applied last. The Route53 solver uses AWS credentials supplied to cert-manager by kube2iam and the resolved zone IDs; the user does not configure a role ARN.

## Issuance behavior

For each domain using the ACME issuer, cert-manager requests one certificate containing:

```text
staging.example.com
*.staging.example.com
```

Traefik continues serving its bootstrap certificate until cert-manager writes a valid ACME certificate to the same TLS Secret. Thereafter cert-manager owns renewal and private-key rotation.

An ACME issuance failure does not silently switch issuer configuration. The bootstrap certificate remains available while cert-manager reports the failure through normal Certificate, Order, Challenge, and ClusterIssuer status.

Certificate approval policy restricts ingress requests to the explicit configured apex and wildcard names rather than allowing arbitrary DNS names from the Traefik namespace.

## Cluster-create wizard

When the user configures a Route53-managed domain, the wizard prompts for the optional ACME account email:

- an email enables ACME using the Let's Encrypt production directory;
- blank input keeps self-signed ingress certificates.

The wizard does not ask about ACME server selection, solver details, IAM roles, or certificate lifetime.

## Implementation plan

1. Default an omitted `cluster.acme.server` to the Let's Encrypt production directory and validate ACME email/domain/provider combinations.
2. Generate least-privilege Route53 IAM roles and kube2iam controller annotations.
3. Restrict the cert-manager namespace to the generated kube2iam role.
4. Pass generated runtime values through the seed resource's provider-neutral values overlay without writing them into cluster config or adding provider-specific seed attributes.
5. Add per-domain issuer selection so unsupported or manual DNS domains remain self-signed while Route53 domains use ACME.
6. Ensure Route53 solvers authenticate using kube2iam-provided AWS credentials and Terraform-resolved hosted-zone IDs.
7. Correct the platform-components example to use `platform.certs.ingress.acme`.
8. Restrict ingress certificate approval policy to configured domain names.
9. Cover config validation, Terraform generation, Netsyseed rendering, Helm rendering, IAM policy restrictions, bootstrap transition, and mixed-domain issuer selection with tests.

## Future work

- Cross-account Route53 role provisioning and provider references; initial support assumes the hosted zone is in the cluster AWS account.
- Enable the retained Cloudflare and Google Cloud DNS solver plumbing and provision their credentials.
- ACME staging-specific wizard UX.
- Other ACME servers and external account binding.
- Explicit certificate lifetime and renewal-window controls.
