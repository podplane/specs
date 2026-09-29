# NLBs and Tunnels for Kubernetes API & Ingress

> **STATUS**: In progress

## Goal

Podplane supports two ways to expose both Envoy Gateway ingress and kube-apiserver:

1. provider Network Load Balancers (NLBs) on AWS and Google Cloud; or
2. outbound tunnels through a provider-neutral interface, initially supporting only Cloudflare Tunnels.

Both integrate with scale-to-zero without placing the `nstance-server proxy` process in the normal production data path. See [ZERO.md](./ZERO.md) for sleep/wake terminology.

## Production routing

While Kubernetes is awake:

- NLB ports 80 and 443 route directly to Envoy on cluster nodes;
- the NLB's configured external API port, defaulting to 6443, routes directly to kube-apiserver port 6443 on control-plane/ingress nodes;
- tunnel processes run on control-plane/ingress VMs and use local kube-apiserver and Envoy ports.

Kube-apiserver must not depend on Envoy Gateway for ingress, because a broken ingress controller must not lock administrators out of the cluster.

Nstance registers agent-healthy instances with AWS target groups or Google Cloud zonal NEGs and deregisters them before deletion. Registration is ready for cutover only when the provider reports the target healthy and routable, not merely when it accepts the membership change. A bounded readiness timeout aborts the cutover and retains or restores the old path. The load balancer performs ongoing health checks after cutover.

## Scale-to-zero proxy

When Podplane sleep support is enabled, vmconfig enables a separate, minimal `nstance-server proxy` systemd service on every `nst` (nstance-server) VM using a generated environment setting. The service remains running while the tenant is awake and asleep; Nstance changes routing, not its process lifecycle. Without sleep support enabled, vmconfig does not enable it. The nstance-server control-plane process is never publicly exposed.

Each shard leader manages its configured proxy listeners. While a local tenant is asleep, a listener whose group has nonzero preserved size routes NLB or tunnel traffic to the local `nstance-server proxy`; a zero-sized group does not advertise that wake target. After a shard leadership change, the new leader installs its routes and the old leader removes or disarms its routes; no server-to-server forwarding is required.

When a cluster goes to sleep, Nstance changes each configured proxy listener's external traffic path to `nstance-server proxy` before removing its final production targets. The proxy:

1. accepts and holds connections for a configurable, long bounded timeout;
2. invokes blocking `WakeTenant(listener)` through the shared root-owned, proxy-accessible Unix gRPC control socket;
3. forwards held connections to the healthy `IP:target_port` returned by Nstance;
4. continues serving established proxied connections while Nstance restores direct production routing.

Proxy listeners are independent: a held request proceeds as soon as its configured group and target port are healthy.

`WakeTenant` uses a shard-local durable compare-and-swap to remove the tenant's sleep entry, so concurrent proxy and timer triggers deduplicate to the same awake state.

## Proxy configuration

The nstance-server configuration file's `load_balancers` field is the source of truth for external routing and proxy listeners. Its provider-specific shape reflects each routing mechanism. The first three examples below are fragments of that file; the later `listeners` examples are generated runtime snapshots streamed to `nstance-server proxy`, not nstance-server configuration files.

An AWS entry in the nstance-server configuration file names every target group and its public, production, and proxy ports:

```jsonc
// nstance-server configuration file example
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

Terraform creates one regional AWS target group for each exposed listener: Envoy ports 80 and 443 and the configured kube-apiserver listener. The same target-group ARNs are passed to every shard. Shard leaders register and deregister only their local production and proxy targets; target groups are not duplicated per Availability Zone or shard.

A Google Cloud entry in the nstance-server configuration file lists its zonal `GCE_VM_IP` NEGs and passthrough frontends. The list includes every eligible production subnet and the Nstance server subnet without repeating those subnet IDs:

```jsonc
// nstance-server configuration file example
"load_balancers": {
  "control-plane": {
    "provider": "google",
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

A tunnel entry in the nstance-server configuration file names only production and local proxy ports:

```jsonc
// nstance-server configuration file example
"load_balancers": {
  "control-plane-tunnel": {
    "provider": "tunnel",
    "tunnel": "control-plane-wake",
    "listeners": [{
      "target_port": 6443,
      "proxy_port": 16443
    }]
  }
}
```

Generated runtime listener keys use `<load-balancer-key>:<proxy-port>`. A tunnel load balancer references an allowlisted name from the separate provider-neutral `tunnels` block described below; it does not embed provider configuration.

Each logical load balancer may be referenced by any number of groups within exactly one tenant. Their instances form its combined backend set, and `nstance-server proxy` forwards to any healthy instance among them. Groups requiring different membership use separate logical load-balancer keys; the existing group `load_balancers` list is unchanged. AWS requires a distinct `proxy_port` for each listener sharing an `nst` VM because the backend sees the VM address, not the NLB address. Google Cloud passthrough frontends may share a local port because the proxy can dispatch by the preserved forwarding-rule destination IP.

At startup and configuration reload, nstance-server derives the following runtime listener snapshot from `load_balancers` and group references and streams it to `nstance-server proxy`. This is not an nstance-server configuration-file fragment:

```jsonc
// derived configuration provided to nstance-server proxy
{
  "listeners": {
    "control-plane:6443": {
      "proxy_port": 6443,
      "tenant": "default",
      "groups": ["control-plane"],
      "target_port": 6443
    }
  }
}
```

Listener identity is the key in the `listeners` map and is not duplicated in the listener value. AWS and tunnel identities use `<load-balancer-key>:<proxy-port>`. A Google listener uses `<load-balancer-key>:<port>` when its port is unique across configured Google frontends. When the port is shared, its identity uses `<load-balancer-key>/<destination-ip>:<port>` and `destination_ip` is included in the listener value. Standard host-and-port formatting keeps IPv6 identities unambiguous.

The following is another generated runtime listener snapshot rather than an nstance-server configuration-file fragment. It shows two Google Cloud frontends sharing port 443 and therefore requiring destination-IP dispatch:

```jsonc
// derived configuration provided to nstance-server proxy
{
  "listeners": {
    "control-plane/34.10.20.30:443": {
      "proxy_port": 443,
      "destination_ip": "34.10.20.30",
      "tenant": "default",
      "groups": ["control-plane"],
      "target_port": 443
    },
    "ingress/34.10.20.31:443": {
      "proxy_port": 443,
      "destination_ip": "34.10.20.31",
      "tenant": "default",
      "groups": ["ingress"],
      "target_port": 443
    }
  }
}
```

The listener snapshot contains no provider resources or dynamic upstream addresses. AWS uses the configured `proxy_port` as the target-registration port override. A Google frontend's single `port` supplies its frontend, proxy, and target ports; duplicate destination-IP and port selectors are rejected.

Both processes use the shared `pkg/proxy` configuration structs. Each listener value contains only `tenant`, `groups`, `target_port`, `proxy_port`, and optional `destination_ip`. All communication between nstance-server and `nstance-server proxy` uses one root-owned, proxy-accessible Unix gRPC control socket. Its versioned service exposes only a versioned initial-and-replacement listener-configuration stream and listener-scoped `WakeTenant`; it carries no secrets, commands, provider-specific configuration, or dynamic membership state.

`nstance-server proxy` waits for the initial listener snapshot before serving, validates each complete replacement, and reconciles listener changes in place without disrupting unchanged listeners. During a brief control-stream disconnect it retains the last valid snapshot while reconnecting, but immediately stops leader-owned work; a prolonged disconnect makes the proxy unavailable rather than extending stale authority. No proxy configuration state is written to disk.

`WakeTenant(listener)` validates the listener, idempotently wakes its tenant, waits for an agent-healthy instance from any configured group and for `target_port` readiness, then returns one private `IP:target_port`. This and the listener stream are the only proxy-facing operations. `nstance-server proxy` understands no provider APIs, Kubernetes, kube-apiserver, or Envoy Gateway.

Every derived listener is a wake trigger and activity source. Validation rejects cross-tenant references, invalid ports, duplicate Google Cloud destination-IP and port selectors, collisions with nstance-server ports, and any reuse of an AWS or tunnel proxy port, including by a Google listener.

## Network Load Balancers

Podplane-generated Terraform supports equivalent AWS and Google Cloud behavior using provider-native resources. A `load_balancers` key is a logical set of backends with common membership, not necessarily a physical load balancer.

AWS uses one regional target group per listener, shared by all shards and Availability Zones, and may send production and `nstance-server proxy` traffic to different target ports through a registration override. Cross-zone load balancing is disabled on these target groups during normal awake operation. Because a sleeping tenant may have a wake-capable proxy target in only one shard, the shard leader beginning sleep idempotently enables cross-zone load balancing on all of its target groups and waits for provider confirmation before withdrawing production targets.

Enabling cross-zone load balancing requires no cluster-wide coordination because concurrent enable operations are idempotent. The Nstance cluster leader coordinates only its re-disablement: after Kubernetes is awake, production routing has been restored, and proxy targets have been removed in every shard, it disables cross-zone load balancing on all target groups. This decision is serialized against new sleep transitions so the cluster leader cannot disable the setting after a shard has enabled it for sleep. Cross-zone control requires `elasticloadbalancing:ModifyTargetGroupAttributes` scoped to the configured target groups; shard leaders continue to own only their local target membership.

Google Cloud uses `GCE_VM_IP` NEGs instead of unmanaged instance groups because a Compute Engine VM can belong to only one instance group. The same VM interface can be represented as an endpoint in multiple NEGs, and the same NEG can back multiple backend services. This allows one VM—particularly an Nstance proxy—to participate in multiple logical load balancers without coupling their membership. It also permits separate NEGs for Envoy ingress on port 443 and kube-apiserver traffic on port 6443, regardless of whether they are backed by one Nstance group or separate groups.

Google Cloud uses a regional external passthrough Network Load Balancer with one `GCE_VM_IP` zonal NEG per logical membership set, zone, and eligible subnet. Terraform attaches all of those NEGs to the logical load balancer's regional backend service; a backend service cannot mix instance groups and NEGs. While awake, Nstance registers each production VM interface with the configured NEG whose discovered subnetwork matches the interface. While asleep, it registers the shard leader's interface with the NEG matching the Nstance server subnet. Google Cloud preserves the forwarding-rule IP and port, so its listener, production, and proxy ports are equal; separate load balancers may reuse that port because their forwarding-rule IPs distinguish tenants.

Google Cloud forwarding-rule IPs are generated provider metadata, not addresses assigned to the backend VM. `nstance-server proxy` listens on `0.0.0.0:<proxy_port>` and uses the accepted connection's local address to select the configured listener. Terraform must create firewall rules for client traffic and health checks. A future provider option may add Google Cloud proxy Network Load Balancers when port translation is required.

Sleep transition:

1. register the wake-capable shard leader's `nstance-server proxy` target and wait for it to become provider-health-check healthy and routable while production remains registered; if readiness times out, retain production routing and abort sleep;
2. deregister the production targets with connection draining enabled while `nstance-server proxy` remains registered and can forward to the existing upstream;
3. once the provider confirms that production targets are draining and receive no new connections, wait for the next normal agent report and perform the final `if_not_busy` check described in [ZERO.md](./ZERO.md);
4. if proxy payload arrived, a connection remains active, or a required report failed, restore production registration before deregistering `nstance-server proxy` and aborting;
5. otherwise wait for production targets to finish deregistering, commit sleep, and terminate production instances.

The final sleep decision and `WakeTenant` are serialized by the tenant transition lock. A proxy arrival either aborts pending sleep or, if sleep commits first, immediately wakes the tenant. Provider API acceptance alone is not the withdrawal barrier: the provider adapter must wait for the target or endpoint to finish draining and reach its fully deregistered state.

Wake transition:

1. wake the tenant and wait for the requested upstream;
2. register production targets and wait for them to become provider-health-check healthy and routable while `nstance-server proxy` remains registered; if readiness times out, retain the proxy path and continue reconciliation;
3. during this overlap, either path may serve traffic;
4. deregister `nstance-server proxy` targets while allowing their active connections to drain.

On AWS, the cluster leader disables target-group cross-zone load balancing only after step 4 has completed for the cluster. This ordering prevents an NLB node in an Availability Zone without a wake-capable target from losing its route during wake.

nstance-server performs target membership changes because it already owns instance lifecycle and limits cloud API access to one security boundary. IAM permissions must be scoped to the cluster's target groups and NEGs; Google Cloud additionally grants the NEG read permission required to resolve each configured NEG's subnet.

The design must account for provider fail-open behaviour and must never intentionally leave both production and proxy registration sets empty or rely on fail-open traffic as evidence that a target is ready during transition. `nstance-server proxy` health checks must not invoke `WakeTenant`; payload activity on production ports remains the sleep guard described in [ZERO.md](./ZERO.md).

## Nstance Server VM

This will require customisation, where today Podplane uses the current nstance module default userdata.

For `vmconfig` we add an `nst` kind instead of Nstance's generic server userdata. It installs nstance-server, Fluent Bit, optional separate `nstance-server proxy` and privileged `nstance-server tunnel` services, and the standard vmconfig configuration/watch machinery. Provider-specific tunnel software and generated configuration live in immutable container images and deployment configuration, not in Nstance or vmconfig code.

Nstance Server VMs use private service subnets by default. Podplane Managed NAT always gives them internet access independent of the NAT group, which sleeps with the tenant:

- AWS places each `nst` VM in a public service subnet with a public IPv4.
- Google Cloud assigns each `nst` VM an external IPv4; Google Cloud has no public/private subnet distinction.

Inbound security-group/firewall access remains denied except for explicitly configured load-balancer health and proxy paths, and nstance-server is never exposed publicly. Enabling Podplane Managed NAT later may rotate Nstance Server VMs through their managed groups, but does not replace Kubernetes nodes.

## Tunnel placement

The tunnel feature contract is provider-neutral, with Cloudflare Tunnel as the initial implementation.

Provider-specific tunnel clients, commands, and configuration live in immutable container images and generated deployment configuration. Neither Nstance nor vmconfig contains provider-specific tunnel logic.

Each control-plane/ingress VM runs one tunnel process for both kube-apiserver and ingress. `knc` is the VM-level control-plane role. Podplane runs Envoy as a host-port DaemonSet on every schedulable node, including control-plane/ingress nodes, so the tunnel uses stable local upstreams:

- kube-apiserver on port 6443;
- Envoy on port 443 when ingress is enabled.

No separate ingress-node role or live upstream discovery is required. Components must continue scheduling Envoy on control-plane/ingress nodes when tunnel ingress is enabled. The tunnel process still runs when ingress is disabled because kube-apiserver remains an endpoint; its configuration simply omits ingress routes.

Tunnel providers handle public HTTP-to-HTTPS redirects at their edge and do not forward port 80 to Envoy. The initial Cloudflare implementation uses a hostname-scoped redirect rule; a future ngrok implementation can use its redirect Traffic Policy. Providers without an edge redirect may expose HTTPS only rather than adding a port-80 tunnel upstream.

Each `knc` control-plane VM's production control-plane tunnel is owned by the VM lifecycle and remains entirely outside Nstance control. While a shard is asleep, a wake-capable shard leader advertises a separate wake tunnel targeting its local `nstance-server proxy`. Reconciliation toward asleep waits for that local wake path before terminating control-plane VMs. Reconciliation toward awake restores the VMs, waits for ordinary fresh agent health and the requested upstream TCP port, then withdraws the wake path. The external tunnel endpoint remains stable.

A separate `nstance-server tunnel` process is the privileged root supervisor for local `nst` wake tunnels. It independently reads the same root-controlled shard configuration as the unprivileged server process and resolves all referenced secret and storage files itself. It initiates the local Unix gRPC stream to the server process, which never sends images, commands, mounts, manifests, secrets, credentials, storage contents, or provider details.

Server messages contain only an allowlisted tunnel name, desired `running` or `stopped` state, revision, and renewable lease. The supervisor rejects names absent from its local configuration, reports revisioned readiness and failure status, applies restart and bounded backoff, and uses root only for the privileges needed to prepare hardened kubelet static Pods. The generated Pods use least privilege, immutable digest-pinned images, explicit mounted files, and no access to the supervisor socket or unrelated host state.

The shard leader renews each desired-state lease with heartbeats. Explicit leadership loss immediately sends or causes revocation. An ambiguous disconnect leaves the current state in place through a bounded lease while `nstance-server tunnel` reconnects; if it cannot renew before expiry, the supervisor stops the tunnel fail-closed. On restart, the supervisor removes stale manifests before loading configuration because its in-memory leases have been lost. A new leader requests its local wake endpoint and waits for matching revision readiness before any destructive transition. No distributed handoff or lifecycle state files are used.

The model permits either a dedicated static Pod per tunnel or multiple logical endpoints managed by one shared local agent while any endpoint is active. The initial Cloudflare deployment uses dedicated tunnel processes and credentials. An ngrok deployment may use one agent with independently controlled endpoints. The configuration expresses these generic lifecycle modes, while the protocol exposes neither the mode nor provider-specific fields.

Cloudflare tunnel processes using one credential provide availability but no traffic steering. Therefore production and wake tunnel processes may overlap only during transitions, when both paths can serve requests; the wake tunnel process must be withdrawn while the tenant is awake so normal traffic bypasses `nstance-server proxy`.

The initial Cloudflare implementation uses public HTTPS applications, not Cloudflare's arbitrary-TCP client mode. Clients connect to Cloudflare's HTTPS edge on port 443 without `cloudflared`; Cloudflare terminates client TLS and sends HTTPS through the tunnel to kube-apiserver on local port 6443 or Envoy on local port 443. The `nst` wake tunnel process sends the same HTTPS origin traffic to local `nstance-server proxy`, which transparently forwards it to the selected production port after wake. Locally managed ingress rules allow the control-plane and `nst` tunnel processes to use these different local targets.

Cloudflare is therefore a trust boundary for Kubernetes API bearer tokens and content. In tunnel mode, Podplane generates the kubeconfig API URL on external port 443 and relies on the operating system's public CA roots for the Cloudflare edge certificate instead of embedding the kube-apiserver CA. Cloudflare-to-origin connections remain TLS-protected. Generated deployment configuration gives the Cloudflare container the Podplane cluster CA and API hostname for the API route, and the configured ingress issuer CA or public roots and ingress hostname for ingress routes. TLS verification must not be disabled.

## Tunnel configuration and files

The root-controlled shard configuration has a dedicated provider-neutral `tunnels` block. Each allowlisted entry selects an immutable container image and mounts secret or storage sources as files:

```jsonc
"tunnels": {
  "control-plane-wake": {
    "image": "registry.example/tunnel@sha256:0123456789abcdef...",
    "files": {
      "/run/tunnel/credential": {
        "kind": "secret",
        "source": "control-plane-tunnel-credential"
      },
      "/etc/tunnel/config.json": {
        "kind": "storage",
        "source": "tunnels/control-plane.json"
      }
    }
  }
}
```

The image reference must be immutable. File destinations are container mount paths; configuration validation rejects writable image tags, duplicate destinations, unsafe paths, unsupported file kinds, and tunnel names not referenced consistently by wake routing. The privileged `nstance-server tunnel` process, not the unprivileged server process, resolves these sources and materializes root-only host files before generating the static Pod. The same sources may independently appear in production VM deployment configuration.

Secret values are not persisted in shared Nstance state and must not appear in generated Terraform values, process arguments, logs, gRPC messages, or world-readable files. `nstance-server tunnel` resolves current source contents whenever it starts a tunnel Pod; the unprivileged server process has no tunnel secret or storage resolution path.

Provider-specific images and configuration are generated by deployment tooling and consumed by containers. Nstance and vmconfig code remain provider-neutral. Production control-plane tunnels remain VM-lifecycle-owned and do not use this desired-state protocol.

## Configuration

Cluster configuration selects NLB, tunnel, or custom exposure independently from application domain configuration. It defines stable external API and ingress endpoints while provider-specific settings remain under the selected infrastructure/tunnel provider.

Podplane generates one target-group or frontend entry for each exposed API or ingress route and assigns its logical load balancer to the appropriate control-plane or ingress group. Nstance derives proxy listeners from that data. For NLB ingress this may include ports 80 and 443; tunnel ingress uses only 443 because the tunnel provider handles HTTP-to-HTTPS redirection.

Validation requires public/direct external placement for `nst` VMs whenever Podplane Managed NAT is enabled. Cloud Managed NAT defaults `nst` placement to private.

Scale-to-zero requires an exposure method capable of reaching `nstance-server proxy`, unless `wake_at` guarantees recovery. Podplane must not allow an indefinitely sleeping cluster with no wake path.

## Implementation areas

- **Podplane CLI:** exposure config, validation, generated Terraform, and endpoint outputs.
- **Nstance Terraform modules:** port-aware AWS target groups and Google Cloud regional external passthrough NLBs, `GCE_VM_IP` NEGs, forwarding-rule metadata, firewall rules, and health checks.
- **Nstance server:** versioned proxy service, lease-based tunnel desired state, target membership, and ordered transitions; no tunnel workload or provider configuration delivery.
- **Nstance tunnel supervisor:** root-controlled config and file resolution, outbound Unix gRPC stream, lease enforcement, hardened static-Pod generation, readiness, and restart/backoff.
- **vmconfig:** VM-lifecycle-owned production `knc` tunnels; installation and hardening of separate `nstance-server proxy` and `nstance-server tunnel` processes; and provider-neutral local routing.
- **Components/Envoy Gateway:** retain control-plane host exposure on port 443 for tunnels and ports 80/443 for NLBs; no kube-apiserver proxying.

Tests must cover shared and separate listener groups, same-tenant shared and cross-tenant rejected load balancers, AWS regional target groups shared across shards and Availability Zones, proxy-port uniqueness, cross-zone enable-before-sleep and disable-after-wake ordering, cluster-leader replacement during each transition, Google Cloud forwarding-rule-address dispatch and equal-port validation, versioned Unix-stream initial/replacement snapshots and reconnect, in-place listener reconciliation, Google Cloud `GCE_VM_IP` NEG metadata resolution and subnet selection, shared same-zone NEG membership, public-HTTPS tunnel kubeconfig and origin verification, one and multiple wake-capable shards, zero-sized group exclusion, partial upstream health, provider-health readiness timeouts and rollback, AWS fail-open without treating an unhealthy target as ready, shard-leader replacement, bounded connection holding, supervisor-initiated reconnect, heartbeat renewal, explicit revocation, bounded ambiguous-disconnect lease expiry, fail-closed stopping, immutable-image and file-mount validation, static-Pod hardening, dedicated processes and shared-agent endpoints, absence of provider fields in the protocol, direct kube-apiserver access, and sleep/wake transitions without empty registration sets.
