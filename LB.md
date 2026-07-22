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

Nstance registers agent-healthy instances with AWS target groups or Google Cloud instance groups and deregisters them before deletion. Ordinary registration is complete when the provider accepts it; the load balancer performs ongoing health checks. Nstance waits for provider-reported health only as a barrier before switching between production and `nstance-proxy` routes.

## Scale-to-zero `nstance-proxy`

When Podplane sleep support is enabled, vmconfig enables a separate, minimal `nstance-proxy` systemd service on every `nst` (nstance-server) VM using a generated environment setting. The service remains running while the tenant is awake and asleep; Nstance changes routing, not its process lifecycle. Without sleep support, vmconfig does not enable it. The nstance-server process is never publicly exposed.

Each shard leader manages its configured proxy listeners. While a local tenant is asleep, a listener whose group has nonzero preserved size routes NLB or tunnel traffic to local `nstance-proxy`; a zero-sized group does not advertise that wake target. After a shard leadership change, the new leader installs its routes and the old leader removes or disarms its routes; no server-to-server forwarding is required.

When a cluster goes to sleep, Nstance changes each configured proxy listener's external traffic path to `nstance-proxy` before removing its final production targets. `nstance-proxy`:

1. accepts and holds connections for a configurable, long bounded timeout;
2. invokes `WakeTenant` through a root-owned local Unix socket;
3. waits for the requested upstream to become healthy;
4. forwards held connections;
5. continues serving established proxied connections while Nstance restores direct production routing.

Proxy listeners are independent: a held request proceeds as soon as its configured group and target port are healthy.

`WakeTenant` uses a shard-local durable compare-and-swap from desired `asleep` to `awake`, so concurrent `nstance-proxy`, timer, and administrative triggers produce only one new local generation.

## Proxy configuration

Nstance server configuration maps each local `nstance-proxy` port to a tenant and production group:

```jsonc
"proxy": {
  "files": {
    "tunnel.json": {
      "kind": "secret",
      "source": "cluster-tunnel"
    }
  },
  "listeners": {
    "6443": {
      "tenant": "red",
      "group": "control-plane"
    },
    "6444": {
      "tenant": "blue",
      "group": "control-plane",
      "target_port": 6443,
      "lb_port": 6443
    }
  }
}
```

The listener map key is the local port bound by `nstance-proxy`. `target_port` is the port on production instances and `lb_port` is the NLB/public inbound listener port; both default to the map key. Map keys must be globally unique on an Nstance-server VM, while different tenants may reuse `target_port` and `lb_port` on separate load balancers. Validation rejects unknown tenants or groups, invalid or duplicate local ports, and collisions with nstance-server ports.

Every listener is a wake trigger and activity source. The mapping drives production target registration, wake-capable-shard selection, activity reporting, upstream readiness, and proxy routing. A single group may serve several listeners, or listeners may use separate groups.

At startup and configuration reload, nstance-server atomically writes derived listener configuration into its fixed local receive directory. `nstance-proxy` reads it, binds the map-key ports, and reloads after atomic replacement. It understands only ports, tenants, and groups—not Kubernetes, kube-apiserver, or Traefik.

## Network Load Balancers

Podplane-generated Terraform supports equivalent AWS and Google Cloud behavior using provider-native resources.

AWS uses dedicated `nstance-proxy` target groups containing each wake-capable shard's current leader. Google Cloud uses equivalent per-instance `nstance-proxy` backends such as `GCE_VM_IP` zonal NEGs rather than complete Nstance-server managed instance groups. An `nst` VM is never added to a production instance group. For each listener, the public `lb_port` forwards to the local proxy map-key port while asleep and to production `target_port` while awake.

Sleep transition:

1. register the wake-capable shard leader's `nstance-proxy` target and wait for load-balancer health;
2. keep the `nstance-proxy` route disarmed while production remains active;
3. switch new flows to the `nstance-proxy` route and include any `nstance-proxy` arrival in the final `if_not_busy` check;
4. abort if traffic arrives, otherwise commit sleep;
5. deregister production targets only after the `nstance-proxy` path is available.

Wake transition:

1. wake the tenant and wait for the requested upstream;
2. register production targets and wait until at least one relevant target is provider-healthy;
3. direct new connections to production;
4. deregister `nstance-proxy` targets after their active connections drain.

nstance-server performs target membership changes because it already owns instance lifecycle and limits cloud API access to one security boundary. IAM permissions must be scoped to the cluster's target groups, instance groups, and health queries.

The design must account for provider fail-open behaviour and must never intentionally leave both production and `nstance-proxy` target sets empty during transition. `nstance-proxy` health checks must not invoke `WakeTenant`; payload activity on production ports remains the sleep guard described in [ZERO.md](./ZERO.md).

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

Cloudflare replicas using one tunnel credential provide availability but no traffic steering. Therefore control-plane and `nstance-proxy` connectors may overlap only during transitions, when both paths can serve requests; the `nstance-proxy` connector must be withdrawn while the tenant is awake so normal traffic bypasses `nstance-proxy`.

The initial Cloudflare implementation uses locally managed ingress rules so control-plane connectors can target local ports 6443/443 while the `nst` connector targets local `nstance-proxy`, despite sharing one tunnel credential.

## Proxy files and tunnel secrets

`proxy.files` reuses the existing `FileConfig` schema and generation code. Nstance-server writes these files atomically to its fixed receive directory; vmconfig validates and installs them for local services. The same tunnel secret source may also appear in a `knc` template and be delivered through the existing Nstance-agent file channel.

Nstance's secret cache remains in-memory and per server process. Initial misses for one source are coalesced so local and agent delivery cause one provider read per cache lifetime. Secret values are not persisted in shared Nstance state. vmconfig-owned watchers install them with strict ownership and modes and restart only affected services; secrets must not appear in generated Terraform values, process arguments, logs, or world-readable files.

Nstance writes only generic proxy, file, and connector-state data. vmconfig translates it into provider-specific service configuration, keeping Cloudflare- or ngrok-specific paths and commands out of Nstance.

## Configuration

Cluster configuration selects NLB, tunnel, or custom exposure independently from application domain configuration. It defines stable external API and ingress endpoints while provider-specific settings remain under the selected infrastructure/tunnel provider.

Podplane generates one proxy listener for each exposed API or ingress route, assigns a unique local port, and maps it to the appropriate control-plane or ingress group. For NLB ingress this may include ports 80 and 443; tunnel ingress uses only 443 because the tunnel provider handles HTTP-to-HTTPS redirection.

Validation requires public/direct external placement for `nst` VMs whenever Podplane Managed NAT is enabled. Cloud Managed NAT defaults `nst` placement to private.

Scale-to-zero requires at least one exposure method capable of reaching `nstance-proxy`. Clusters without one may still be slept and woken explicitly through the CLI but cannot wake from external traffic.

## Implementation areas

- **Podplane CLI:** exposure config, validation, generated Terraform, and endpoint outputs.
- **Nstance Terraform modules:** `nstance-proxy` NLB targets and health checks for AWS and Google Cloud.
- **Nstance server:** proxy configuration, target/connector membership, cached local/agent file delivery, and atomic routing transitions.
- **vmconfig:** `knc`/`nst` tunnel services, optional `nstance-proxy`, local routing, and secret watchers.
- **Components/Traefik:** retain control-plane host exposure on port 443 for tunnels and ports 80/443 for NLBs; no kube-apiserver proxying.

Tests must cover shared and separate listener groups, multi-tenant port overrides, duplicate local ports, proxy-file generation, initial secret-miss coalescing, NLB and tunnel modes, one and multiple wake-capable shards, zero-sized group exclusion, partial upstream health, shard-leader replacement, bounded connection holding, direct kube-apiserver access, and sleep/wake transitions without an empty route.
