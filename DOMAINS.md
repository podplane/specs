# Podplane Cluster Domains

## Goal

Podplane clusters may optionally have one or more domains for application ingress and stable cluster endpoints. Domain configuration is independent of the infrastructure provider: for example, a cluster may run on AWS while Cloudflare manages its DNS.

Domains are optional. A cluster running only workers may omit them and may omit Traefik and other ingress components.

## Domain model

`cluster.domains[].zone` is the exact ingress apex. Podplane does not prepend the cluster ID.

For example, this domain:

```jsonc
{
  "cluster": {
    "domains": [
      {
        "zone": "staging.example.com",
        "provider": {
          "kind": "cloudflare"
        }
      }
    ]
  }
}
```

configures these ingress names:

```text
staging.example.com
*.staging.example.com
```

The first domain is the default. Advanced users may add further domains by editing the cluster config; array order selects the default.

Each domain may use a different DNS provider. The DNS provider is independent of `cluster.providers`, which describes cluster infrastructure. Omitting `provider` means DNS is managed manually.

Podplane assumes the user already owns the domain and has configured its authoritative nameservers. Users may delegate a dedicated zone such as `staging.example.com` to limit the DNS access granted to cluster administrators.

## DNS records

The domain apex and wildcard point to the cluster ingress load balancer:

```text
staging.example.com   -> cluster ingress load balancer
*.staging.example.com -> cluster ingress load balancer
```

When a DNS provider is configured, generated Terraform creates suitable provider-native records. It should use CNAME records where valid and alias, flattening, A, or AAAA records where required for a zone apex.

When `provider` is omitted, generated Terraform does not manage records. Its outputs provide the target and actionable record guidance. Managed domains need only concise outputs identifying the configured names; they do not need manual setup instructions.

## Kubernetes API

When at least one domain is configured, `cluster.kubernetes.api_hostname` is required. The cluster-create wizard writes it explicitly as `k8s.<default-domain>` so administrators can see and change the endpoint:

```jsonc
{
  "cluster": {
    "kubernetes": {
      "api_hostname": "k8s.staging.example.com",
      "api_port": 6443
    }
  }
}
```

The Kubernetes API is exposed publicly by default through the existing provider load-balancer configuration. The wizard does not ask about exposure. Users needing private access can customize the generated Terraform with a private load balancer, security groups, VPN, tunnel, or jump host.

Without a cluster domain, the public API uses the provider-assigned load-balancer hostname. Podplane's kubeconfig uses the cluster CA to trust its certificate.

Changing `api_hostname` is a mutable cluster update: it propagates through `mutable.env`, regenerates the kube-apiserver certificate with the new DNS SAN, and restarts kube-apiserver. Service-account issuer behavior is outside this specification.

## Registry

When a domain is configured, the cluster-create wizard writes `cluster.registry.hostname` as `registry.<default-domain>`:

```jsonc
{
  "cluster": {
    "registry": {
      "hostname": "registry.staging.example.com",
      "ingress": {
        "enabled": false
      }
    }
  }
}
```

Registry ingress is disabled by default. `podplane push` continues to use Kubernetes port-forwarding and does not require Traefik or public registry ingress. Node-local Zot resolves the configured registry hostname locally for containerd pulls.

If registry ingress is later enabled, the existing hostname works through the shared ingress load balancer without changing deployed image references. If admins desire a separate registry load balancer, this is outside scope but still achievable by adding custom .tf config to the generated-tf module.

Domainless clusters use the existing internal registry hostname `<cluster-id>-registry.local`.

## TLS

The Kubernetes API uses its Podplane cluster CA, which the CLI places in kubeconfig.

Initial domain support configures DNS and may use bootstrap self-signed ingress certificates. Public ACME certificates, wildcard issuance, cert-manager DNS-01 credentials, and renewal behavior are later work. They should build on the DNS provider model in this specification without coupling DNS to the infrastructure provider.

## Cluster-create wizard

The wizard asks only:

1. Whether the cluster has a domain.
2. The single domain, if present.
3. Its DNS provider, which may be left unset for manual DNS.

It writes the domain, `k8s.<domain>`, and `registry.<domain>` explicitly. Multiple domains and provider-specific advanced settings remain cluster-config edits.
