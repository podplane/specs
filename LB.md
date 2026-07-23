# NLBs and Tunnels for Kubernetes API & Ingress

## Goal

Podplane supports two ways to expose both Traefik ingress and kube-apiserver:

1. provider Network Load Balancers (NLBs) on AWS and Google Cloud; or
2. outbound tunnels through a provider-neutral interface, initially supporting only Cloudflare Tunnels.

Both integrate with scale-to-zero without placing `nstance-proxy` in the normal production data path. See [ZERO.md](./ZERO.md) for sleep/wake terminology.

## Production routing

While Kubernetes is awake:

- NLB ports 80 and 443 route directly to Traefik on cluster nodes;
- the NLB's configured external API port, defaulting to 6443, routes directly to kube-apiserver port 6443 on control-plane nodes;
- tunnel connectors run on control-plane VMs and use local kube-apiserver and Traefik ports.

Kube-apiserver must not depend on Traefik for ingress, because a broken ingress controller must not lock administrators out of the cluster.

Nstance registers agent-healthy instances with AWS target groups or Google Cloud zonal NEGs and deregisters them before deletion. Registration is ready for cutover only when the provider reports the target healthy and routable, not merely when it accepts the membership change. A bounded readiness timeout aborts the cutover and retains or restores the old path. The load balancer performs ongoing health checks after cutover.

## Scale-to-zero `nstance-proxy`

When Podplane sleep support is enabled, vmconfig enables a separate, minimal `nstance-proxy` systemd service on every `nst` (nstance-server) VM using a generated environment setting. The service remains running while the tenant is awake and asleep; Nstance changes routing, not its process lifecycle. Without sleep support, vmconfig does not enable it. The nstance-server process is never publicly exposed.

Each shard leader manages its configured proxy listeners. While a local tenant is asleep, a listener whose group has nonzero preserved size routes NLB or tunnel traffic to local `nstance-proxy`; a zero-sized group does not advertise that wake target. After a shard leadership change, the new leader installs its routes and the old leader removes or disarms its routes; no server-to-server forwarding is required.

When a cluster goes to sleep, Nstance changes each configured proxy listener's external traffic path to `nstance-proxy` before removing its final production targets. `nstance-proxy`:

1. accepts and holds connections for a configurable, long bounded timeout;
2. invokes blocking `WakeTenant(listener)` through a root-owned local Unix socket;
3. forwards held connections to the healthy `IP:target_port` returned by Nstance;
4. continues serving established proxied connections while Nstance restores direct production routing.

Proxy listeners are independent: a held request proceeds as soon as its configured group and target port are healthy.

`WakeTenant` uses a shard-local durable compare-and-swap to remove the tenant's sleep entry, so concurrent `nstance-proxy` and timer triggers deduplicate to the same awake state.

## Proxy configuration

`load_balancers` is the source of truth for external routing and proxy listeners. Its provider-specific shape reflects each routing mechanism.

AWS configuration names every target group and its public, production, and proxy ports:

```jsonc
"load_balancers": {
  "control-plane": {
    "provider": "aws",
    "target_groups": [{
      "arn": "arn:aws:elasticloadbalancing:...:targetgroup/api/...",
      "listener_port": 6443,
      "target_port": 6443,
      "proxy_port": 6443
    }]
  }
}
```

Terraform creates one regional AWS target group for each exposed listener: Traefik ports 80 and 443 and the configured kube-apiserver listener. The same target-group ARNs are passed to every shard. Shard leaders register and deregister only their local production and proxy targets; target groups are not duplicated per Availability Zone or shard.

Google Cloud configuration lists its zonal `GCE_VM_IP` NEGs and passthrough frontends. The list includes every eligible production subnet and the Nstance server subnet without repeating those subnet IDs:

```jsonc
"load_balancers": {
  "control-plane": {
    "provider": "gcp",
    "network_endpoint_groups": [
      "control-plane-us-central1-a",
      "control-plane-nstance-us-central1-a"
    ],
    "frontends": [{
      "ip": "34.10.20.30",
      "port": 6443
    }]
  }
}
```

At startup and configuration reload, Nstance reads each configured NEG from Google Cloud and uses its immutable subnetwork metadata to build the subnet-to-NEG lookup for that logical load balancer. Validation requires each NEG to use `GCE_VM_IP`, belong to the shard's zone, and map to a unique subnet within the logical load balancer. Terraform creates the NEGs, attaches them to the regional backend service, and passes only their names to Nstance. Multiple shards sharing a zone and subnet receive the same NEG name and independently mutate only their own endpoints.

Tunnel configuration names only production and local proxy ports:

```jsonc
"load_balancers": {
  "control-plane-tunnel": {
    "provider": "tunnel",
    "listeners": [{
      "target_port": 6443,
      "proxy_port": 16443
    }]
  }
}
```

Tunnel listeners need no user-defined name; generated runtime keys use `<load-balancer-key>:<proxy-port>`. Hostnames, tunnel IDs, credentials, and provider commands remain in vmconfig and `proxy.files`.

Each logical load balancer may be referenced by any number of groups within exactly one tenant. Their instances form its combined backend set, and `nstance-proxy` forwards to any healthy instance among them. Groups requiring different membership use separate logical load-balancer keys; the existing group `load_balancers` list is unchanged. AWS requires a distinct `proxy_port` for each listener sharing an `nst` VM because the backend sees the VM address, not the NLB address. Google Cloud passthrough frontends may share a local port because `nstance-proxy` can dispatch by the preserved forwarding-rule destination IP.

At startup and configuration reload, nstance-server derives static runtime configuration from `load_balancers` and group references:

```jsonc
{
  "listeners": {
    "control-plane:6443": {
      "proxy_port": 6443,
      // Google Cloud only when this port is shared:
      "destination_ip": "34.10.20.30",
      "tenant": "default",
      "groups": ["control-plane"],
      "target_port": 6443
    }
  }
}
```

`destination_ip` is present only when required to disambiguate Google Cloud frontends sharing a port. The file contains no provider resources or dynamic upstream addresses. AWS instead uses the configured `proxy_port` as the target-registration port override; Google Cloud requires frontend, proxy, and target ports to match.

Both processes use structs from a shared Nstance proxy package. Nstance-server atomically writes `/run/nstance/nstance-proxy.json` with root ownership and proxy-readable permissions; `nstance-proxy` waits for it at startup and reloads after replacement. The file is not rewritten for instance membership changes.

`WakeTenant(listener)` validates the listener, idempotently wakes its tenant, waits for an agent-healthy instance from any configured group and for `target_port` readiness, then returns one private `IP:target_port`. This remains the only nstance-proxy-to-server operation. `nstance-proxy` understands no provider APIs, Kubernetes, kube-apiserver, or Traefik.

Every derived listener is a wake trigger and activity source. Validation rejects cross-tenant references, invalid ports, AWS and tunnel proxy-port collisions, Google Cloud frontend collisions, unequal Google passthrough ports, and collisions with nstance-server ports.

## Network Load Balancers

Podplane-generated Terraform supports equivalent AWS and Google Cloud behavior using provider-native resources. A `load_balancers` key is a logical set of backends with common membership, not necessarily a physical load balancer.

AWS uses one regional target group per listener, shared by all shards and Availability Zones, and may send production and `nstance-proxy` traffic to different target ports through a registration override. Cross-zone load balancing is disabled on these target groups during normal awake operation. Because a sleeping tenant may have a wake-capable proxy target in only one shard, the shard leader beginning sleep idempotently enables cross-zone load balancing on all of its target groups and waits for provider confirmation before withdrawing production targets.

Enabling cross-zone load balancing requires no cluster-wide coordination because concurrent enable operations are idempotent. The Nstance cluster leader coordinates only its re-disablement: after Kubernetes is awake, production routing has been restored, and proxy targets have been removed in every shard, it disables cross-zone load balancing on all target groups. This decision is serialized against new sleep transitions so the cluster leader cannot disable the setting after a shard has enabled it for sleep. Cross-zone control requires `elasticloadbalancing:ModifyTargetGroupAttributes` scoped to the configured target groups; shard leaders continue to own only their local target membership.

Google Cloud uses `GCE_VM_IP` NEGs instead of unmanaged instance groups because a Compute Engine VM can belong to only one instance group. The same VM interface can be represented as an endpoint in multiple NEGs, and the same NEG can back multiple backend services. This allows one VM—particularly an Nstance proxy—to participate in multiple logical load balancers without coupling their membership. It also permits separate NEGs for Traefik ingress on port 443 and kube-apiserver traffic on port 6443, regardless of whether they are backed by one Nstance group or separate groups.

Google Cloud uses a regional external passthrough Network Load Balancer with one `GCE_VM_IP` zonal NEG per logical membership set, zone, and eligible subnet. Terraform attaches all of those NEGs to the logical load balancer's regional backend service; a backend service cannot mix instance groups and NEGs. While awake, Nstance registers each production VM interface with the configured NEG whose discovered subnetwork matches the interface. While asleep, it registers the shard leader's interface with the NEG matching the Nstance server subnet. Google Cloud preserves the forwarding-rule IP and port, so its listener, production, and proxy ports are equal; separate load balancers may reuse that port because their forwarding-rule IPs distinguish tenants.

Google Cloud forwarding-rule IPs are generated provider metadata, not addresses assigned to the backend VM. `nstance-proxy` listens on `0.0.0.0:<proxy_port>` and uses the accepted connection's local address to select the configured listener. Terraform must create firewall rules for client traffic and health checks. A future provider option may add Google Cloud proxy Network Load Balancers when port translation is required.

Sleep transition:

1. register the wake-capable shard leader's `nstance-proxy` target and wait for it to become provider-health-check healthy and routable while production remains registered; if readiness times out, retain production routing and abort sleep;
2. deregister the production targets with connection draining enabled while `nstance-proxy` remains registered and can forward to the existing upstream;
3. once the provider confirms that production targets are draining and receive no new connections, wait for the next normal agent report and perform the final `if_not_busy` check described in [ZERO.md](./ZERO.md);
4. if proxy payload arrived, a connection remains active, or a required report failed, restore production registration before deregistering `nstance-proxy` and aborting;
5. otherwise wait for production targets to finish deregistering, commit sleep, and terminate production instances.

The final sleep decision and `WakeTenant` are serialized by the tenant transition lock. A proxy arrival either aborts pending sleep or, if sleep commits first, immediately wakes the tenant. Provider API acceptance alone is not the withdrawal barrier: the provider adapter must wait for the target or endpoint to finish draining and reach its fully deregistered state.

Wake transition:

1. wake the tenant and wait for the requested upstream;
2. register production targets and wait for them to become provider-health-check healthy and routable while `nstance-proxy` remains registered; if readiness times out, retain the proxy path and continue reconciliation;
3. during this overlap, either path may serve traffic;
4. deregister `nstance-proxy` targets while allowing their active connections to drain.

On AWS, the cluster leader disables target-group cross-zone load balancing only after step 4 has completed for the cluster. This ordering prevents an NLB node in an Availability Zone without a wake-capable target from losing its route during wake.

nstance-server performs target membership changes because it already owns instance lifecycle and limits cloud API access to one security boundary. IAM permissions must be scoped to the cluster's target groups and NEGs; Google Cloud additionally grants the NEG read permission required to resolve each configured NEG's subnet.

The design must account for provider fail-open behaviour and must never intentionally leave both production and `nstance-proxy` registration sets empty or rely on fail-open traffic as evidence that a target is ready during transition. `nstance-proxy` health checks must not invoke `WakeTenant`; payload activity on production ports remains the sleep guard described in [ZERO.md](./ZERO.md).

## Nstance Server VM

This will require customisation, where today Podplane uses the current nstance module default userdata.

For `vmconfig` we add an `nst` kind instead of Nstance's generic server userdata. It installs nstance-server, Fluent Bit, optional `nstance-proxy`, the configured tunnel client, and the standard vmconfig configuration/watch machinery. Tunnel dependencies and provider-specific configuration remain in vmconfig, not Nstance.

Nstance Server VMs use private service subnets by default. Podplane Managed NAT always gives them internet access independent of the NAT group, which sleeps with the tenant:

- AWS places each `nst` VM in a public service subnet with a public IPv4.
- Google Cloud assigns each `nst` VM an external IPv4; Google Cloud has no public/private subnet distinction.

Inbound security-group/firewall access remains denied except for explicitly configured load-balancer health and `nstance-proxy` paths, and nstance-server is never exposed publicly. Enabling Podplane Managed NAT later may rotate Nstance Server VMs through their managed groups, but does not replace Kubernetes nodes.

## Tunnel placement

The tunnel feature contract is provider-neutral, with Cloudflare Tunnel as the initial implementation.

vmconfig installs the selected tunnel client on both `knc` control-plane VMs and `nst` Nstance Server VMs. Provider-specific vmconfig variants/kinds may package different clients without adding provider logic to Nstance.

Each control-plane VM runs one tunnel process for both kube-apiserver and ingress. `knc` is the VM-level control-plane role. Podplane currently runs Traefik as a host-port DaemonSet on every schedulable node, including control-plane nodes, so the tunnel uses stable local upstreams:

- kube-apiserver on port 6443;
- Traefik on port 443 when ingress is enabled.

No separate ingress-node role, Traefik sidecar, or live upstream discovery is required. Components must continue scheduling Traefik on control-plane nodes when tunnel ingress is enabled. The tunnel process still runs when ingress is disabled because kube-apiserver remains an endpoint; its configuration simply omits ingress routes.

Tunnel providers handle public HTTP-to-HTTPS redirects at their edge and do not forward port 80 to Traefik. The initial Cloudflare implementation uses a hostname-scoped redirect rule; a future ngrok implementation can use its redirect Traffic Policy. Providers without an edge redirect may expose HTTPS only rather than adding a port-80 tunnel upstream.

While a shard's desired state is awake, its control-plane connectors advertise the production tunnel path. While asleep, a wake-capable shard leader advertises its local `nstance-proxy` connector. Reconciliation toward asleep waits for that connector before terminating control-plane VMs; reconciliation toward awake keeps it until a control-plane connector and the requested local upstream are ready. The external tunnel endpoint remains stable.

Nstance publishes generic connector state by logical load-balancer key, such as `/run/nstance/connectors/control-plane-tunnel.active`. vmconfig maps that key to its tunnel service. Nstance does not interpret tunnel-provider configuration.

Cloudflare replicas using one tunnel credential provide availability but no traffic steering. Therefore control-plane and `nstance-proxy` connectors may overlap only during transitions, when both paths can serve requests; the `nstance-proxy` connector must be withdrawn while the tenant is awake so normal traffic bypasses `nstance-proxy`.

The initial Cloudflare implementation uses public HTTPS applications, not Cloudflare's arbitrary-TCP client mode. Clients connect to Cloudflare's HTTPS edge on port 443 without `cloudflared`; Cloudflare terminates client TLS and sends HTTPS through the tunnel to kube-apiserver on local port 6443 or Traefik on local port 443. The `nst` connector sends the same HTTPS origin traffic to local `nstance-proxy`, which transparently forwards it to the selected production port after wake. Locally managed ingress rules allow the control-plane and `nst` connectors to use these different local targets despite sharing one tunnel credential.

Cloudflare is therefore a trust boundary for Kubernetes API bearer tokens and content. In tunnel mode, Podplane generates the kubeconfig API URL on external port 443 and relies on the operating system's public CA roots for the Cloudflare edge certificate instead of embedding the kube-apiserver CA. Cloudflare-to-origin connections remain TLS-protected. For the API route, vmconfig configures `cloudflared` with the Podplane cluster CA and API hostname. For ingress routes, it uses the configured ingress issuer CA or public roots and the ingress hostname. TLS verification must not be disabled.

## Proxy files and tunnel secrets

`proxy.files` reuses the existing `FileConfig` schema and generation code for tunnel secrets and other server-local files. It is not included in the generated `nstance-proxy` configuration. Nstance-server writes these files atomically to its fixed receive directory; vmconfig validates and installs them for local services. The same tunnel secret source may also appear in a `knc` template and be delivered through the existing Nstance-agent file channel.

Nstance's secret cache remains in-memory and per server process. Initial misses for one source are coalesced so local and agent delivery cause one provider read per cache lifetime. Secret values are not persisted in shared Nstance state. vmconfig-owned watchers install them with strict ownership and modes and restart only affected services; secrets must not appear in generated Terraform values, process arguments, logs, or world-readable files.

Nstance writes only generic proxy, file, and connector-state data. vmconfig translates it into provider-specific service configuration, keeping Cloudflare- or ngrok-specific paths and commands out of Nstance.

## Configuration

Cluster configuration selects NLB, tunnel, or custom exposure independently from application domain configuration. It defines stable external API and ingress endpoints while provider-specific settings remain under the selected infrastructure/tunnel provider.

Podplane generates one target-group or frontend entry for each exposed API or ingress route and assigns its logical load balancer to the appropriate control-plane or ingress group. Nstance derives proxy listeners from that data. For NLB ingress this may include ports 80 and 443; tunnel ingress uses only 443 because the tunnel provider handles HTTP-to-HTTPS redirection.

Validation requires public/direct external placement for `nst` VMs whenever Podplane Managed NAT is enabled. Cloud Managed NAT defaults `nst` placement to private.

Scale-to-zero requires an exposure method capable of reaching `nstance-proxy`, unless `wake_at` guarantees recovery. Podplane must not allow an indefinitely sleeping cluster with no wake path.

## Implementation areas

- **Podplane CLI:** exposure config, validation, generated Terraform, and endpoint outputs.
- **Nstance Terraform modules:** port-aware AWS target groups and Google Cloud regional external passthrough NLBs, `GCE_VM_IP` NEGs, forwarding-rule metadata, firewall rules, and health checks.
- **Nstance server:** proxy configuration, target/connector membership, cached local/agent file delivery, and ordered membership transitions.
- **vmconfig:** `knc`/`nst` tunnel services, optional `nstance-proxy`, local routing, and secret watchers.
- **Components/Traefik:** retain control-plane host exposure on port 443 for tunnels and ports 80/443 for NLBs; no kube-apiserver proxying.

Tests must cover shared and separate listener groups, same-tenant shared and cross-tenant rejected load balancers, AWS regional target groups shared across shards and Availability Zones, proxy-port uniqueness, cross-zone enable-before-sleep and disable-after-wake ordering, cluster-leader replacement during each transition, Google Cloud forwarding-rule-address dispatch and equal-port validation, derived proxy-file generation, initial secret-miss coalescing, Google Cloud `GCE_VM_IP` NEG metadata resolution and subnet selection, shared same-zone NEG membership, public-HTTPS tunnel kubeconfig and origin verification, one and multiple wake-capable shards, zero-sized group exclusion, partial upstream health, provider-health readiness timeouts and rollback, AWS fail-open without treating an unhealthy target as ready, shard-leader replacement, bounded connection holding, direct kube-apiserver access, and sleep/wake transitions without empty registration sets.
