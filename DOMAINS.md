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
          "kind": "aws-route53"
        },
        "load_balancer": "main"
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

## Load-balancer selection

Load balancers are a named map under each infrastructure provider. Domain and Kubernetes API configuration reference those stable keys:

```jsonc
{
  "cluster": {
    "providers": [{
      "kind": "aws",
      "load_balancers": {
        "main": {
          "public": true,
          "subnets": "public",
          "listeners": [
            { "port": 443, "pool": "ingress" },
            { "port": 6443, "pool": "control-plane" }
          ]
        },
        "kubernetes-api": {
          "public": true,
          "subnets": "public",
          "listeners": [
            { "port": 443, "target_port": 6443, "pool": "control-plane" }
          ]
        }
      }
    }]
  }
}
```

`domains[].load_balancer` selects the target for that domain's apex and wildcard records and defaults to `main`. `kubernetes.api_load_balancer` independently selects the target for the exact API hostname. It has no implicit default: omitting it disables managed API load-balancer and DNS wiring. This permits the API and application ingress to share a load balancer, use separate load balancers, or leave API connectivity to custom infrastructure.

Each load balancer declares its listeners explicitly. A listener's `port` is the external port, `target_port` is the port on its target VMs and defaults to `port`, and `pool` selects the VMs registered with that listener's target group. The load balancer's `subnets` field independently selects its placement subnet role.

Podplane validates the listeners required by higher-level configuration:

- A domain requires `443 -> 443` on its selected load balancer.
- The Kubernetes API requires `api_port -> 6443` targeting the `control-plane` pool on its selected load balancer.
- Listener ports must be unique within a load balancer.

When a domain is configured, the cluster-create wizard creates `main` with explicit ingress and Kubernetes API listeners and selects it for the Kubernetes API. Advanced config may define more load balancers and arbitrary listeners.

## DNS records

The domain apex and wildcard point to the domain's selected load balancer:

```text
staging.example.com   -> cluster ingress load balancer
*.staging.example.com -> cluster ingress load balancer
```

When a DNS provider is configured, generated Terraform creates suitable provider-native records. It should use CNAME records where valid and alias, flattening, A, or AAAA records where required for a zone apex.

When `provider` is omitted, generated Terraform does not manage records. Its outputs provide the target and actionable record guidance. Managed domains need only concise outputs identifying the configured names; they do not need manual setup instructions.

## Kubernetes API

`cluster.kubernetes.api_hostname` is always required. When a domain is configured, the cluster-create wizard writes it explicitly as `k8s.<default-domain>` so administrators can see and change the endpoint:

```jsonc
{
  "cluster": {
    "kubernetes": {
      "api_hostname": "k8s.staging.example.com",
      "api_port": 6443,
      "api_load_balancer": "main"
    }
  }
}
```

When `api_load_balancer` is set, the API hostname gets an exact DNS record targeting it; this record takes precedence over a domain wildcard that targets another load balancer. It uses the default domain's DNS provider, whose configured authoritative zone must cover the API hostname. If it does not, the API record must be managed manually. When `api_load_balancer` is omitted, Podplane creates neither an API load-balancer attachment nor an API DNS record.

For domain-configured clusters, the wizard exposes the Kubernetes API publicly by selecting the public `main` load balancer. The wizard does not ask about exposure. Advanced users may select a private load balancer or omit `api_load_balancer` and provide connectivity through custom security groups, VPN, tunnel, jump host, or Terraform.

The default API port is 6443. Setting it to 443 requires an explicit load-balancer listener from port 443 to kube-apiserver port 6443.

For Cloudflare, `domains[].provider.proxied` controls whether that domain's records are proxied and defaults to false. The exact Kubernetes API record is proxied if and only if the default domain sets `proxied` to true and `api_port` is 443; otherwise it is DNS-only.

The design must support the first case initially and leave room for the second as future work:

1. An AWS-managed domain and API both select `main`; `k8s.example.com:6443` reaches the same load balancer as `example.com` and `*.example.com`.
2. A Cloudflare-managed `staging.example.com` sets `proxied` to true and selects `main`, while `k8s.example.com:443` selects `kubernetes-api`, is also proxied, and forwards to kube-apiserver on port 6443.

Without a cluster domain, the user must supply `api_hostname` and arrange its connectivity and DNS outside Podplane's managed generation. Podplane does not infer a provider load-balancer hostname or create API DNS for domainless clusters. The configured hostname is used for the kube-apiserver certificate SAN and kubeconfig; Podplane's kubeconfig uses the cluster CA to trust its certificate.

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

If registry ingress is later enabled, the existing hostname works through the default domain's selected load balancer without changing deployed image references. If admins desire a separate registry load balancer, this is outside scope but still achievable by adding custom .tf config to the generated-tf module.

Domainless clusters use the existing internal registry hostname `<cluster-id>-registry.local`.

## TLS

The Kubernetes API uses its Podplane cluster CA, which the CLI places in kubeconfig.

Ingress starts with temporary bootstrap certificates. Domains using a supported ACME DNS provider can use publicly trusted apex and wildcard certificates; manual domains and unsupported DNS providers continue using the self-signed ingress issuer. See [ACME.md](./ACME.md) for configuration, credentials, issuance, and renewal behavior.

## Cluster-create wizard

The wizard asks only:

1. The cluster domain, which may be left blank to skip domain configuration.
2. Its DNS provider when a domain is entered; the provider may be left unset for manual DNS.
3. The optional ACME account email when the selected DNS provider supports ACME.
4. The Kubernetes API hostname when the domain is left blank.

With a domain, it writes the domain, `k8s.<domain>`, `registry.<domain>`, `kubernetes.api_load_balancer: "main"`, and a public `main` load balancer with explicit ingress and Kubernetes API listeners. A supplied ACME email also writes `cluster.acme`; leaving it blank keeps self-signed ingress certificates. Without a domain, it writes the supplied API hostname and leaves load-balancer and DNS wiring to the user. The domain may omit its load-balancer reference because it defaults to `main`. Multiple domains, load balancers, and provider-specific advanced settings remain cluster-config edits.

## Implementation plan

### 1. Finalize the config contract

- Make `domains[].provider` optional in the Go model and JSON Schema.
- Validate domain names, reject duplicates, and always require `kubernetes.api_hostname`.
- Keep the first domain as the default without adding another field.
- Preserve the existing flat Kubernetes and registry fields.
- Replace the singular provider `load_balancer` with a named `load_balancers` map whose entries specify `public`, `subnets`, and explicit listeners.
- Keep `port`, optional `target_port`, and `pool` on each listener so every target group registers VMs from the selected pool.
- Add `domains[].load_balancer`, defaulting to `main`, and optional `kubernetes.api_load_balancer` references; validate configured references resolve.
- Validate that domain and API load-balancer references have their required listener mappings and reject duplicate external ports.
- Add resolved helpers for the default domain and domainless registry hostname.

### 2. Extend the cluster-create wizard

- Prompt directly for the optional domain rather than asking a separate yes/no question.
- When entered, treat it as the exact ingress apex and ask for its optional DNS provider.
- With a domain, write the domain, `k8s.<domain>`, `registry.<domain>`, and `kubernetes.api_load_balancer: "main"` explicitly.
- Without a domain, ask for and write `kubernetes.api_hostname` without managed load-balancer or DNS wiring.
- For domain-configured clusters, keep registry ingress disabled and the Kubernetes API public by default.
- Do not expose multiple domains or advanced DNS settings in the wizard.

### 3. Expose the load-balancer DNS target

- Extend the Nstance network module input to accept Podplane's explicit listener-to-target port mappings.
- Preserve listener identity through target-group outputs so each pool registers only with its configured listeners.
- Extend each Nstance load-balancer output with its canonical hosted-zone ID in addition to its existing DNS name.
- Preserve named load-balancer keys through Podplane's generated Terraform and Nstance modules.

### 4. Generate DNS resources and outputs

- For each managed domain, generate apex and wildcard records targeting its selected load balancer.
- When configured, generate an exact Kubernetes API record targeting its independently selected load balancer.
- Route53 should use alias A and, for dual-stack load balancers, alias AAAA records.
- Use standard Terraform-provider authentication; do not store DNS credentials in cluster config.
- For domains without a provider, emit the load-balancer target and provider-neutral manual instructions.
- Suppress manual instructions for managed domains and output only their configured names and targets.

### 5. Propagate cluster endpoint values

- Pass the explicit Kubernetes API hostname through generated Terraform and `mutable.env`.
- Pass `api_port` as the external client port while keeping kube-apiserver's target port at 6443.
- Include it in kube-apiserver certificate SANs and verify that applying a hostname change reissues the certificate and restarts kube-apiserver without replacing VMs.
- Pass the explicit registry hostname to vmconfig and Netsyseed.
- Use `<cluster-id>-registry.local` when no domain is configured.
- Leave service-account issuer behavior unchanged and outside this work.

### 6. Wire seeded ingress configuration

- Continue passing all configured domains to the Traefik component, with the first marked as its default.
- Ensure no ingress, cert-manager, or Traefik components are enabled solely because a domain is absent.
- Keep registry ingress disabled unless explicitly enabled; when enabled, route `registry.<default-domain>` through the default domain's selected load balancer.
- Keep ACME and cert-manager DNS-01 automation aligned with [ACME.md](./ACME.md), including per-domain issuer selection.

### 7. Verify end-to-end behavior

- Add config parsing, validation, JSON Schema, wizard, Terraform generation, and Netsyseed tests.
- Cover domainless explicit API hostnames, missing API hostname validation, no managed API load balancer, manual DNS, Route53, multiple domains, multiple load balancers, explicit target-port mappings, missing required listeners, duplicate ports, listener-specific pools, and explicit hostname overrides.
- Include the shared AWS load-balancer/port-6443 scenario as an acceptance test.
- Verify generated DNS outputs, kubeconfig connectivity, mutable API hostname updates, node-local registry pulls, port-forwarded pushes, and optional registry ingress.
- Validate/lint generated Terraform for each supported infrastructure/DNS-provider combination before advertising that combination.

## Future work

- Add first-class cross-account DNS-provider references; initial Route53 support assumes the hosted zone is in the cluster AWS account.
- Add Cloudflare DNS generation without changing the domain or load-balancer model.
- Add Cloudflare `domains[].provider.proxied`, defaulting to false. Apply it to the exact Kubernetes API record only when `api_port` is 443.
- Verify the split Cloudflare-proxied API/port-443 scenario, including origin TLS and Kubernetes streaming requests.
- Add other DNS providers independently. Do not claim support where a provider cannot reliably map a zone apex to the load balancer. Google Cloud DNS, for example, requires stable A/AAAA targets or another apex-alias solution for an AWS NLB.
