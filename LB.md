# NLBs and Tunnels for Kubernetes API & Ingress

## Goal

Podplane supports two ways to expose both Traefik ingress and kube-apiserver:

1. provider Network Load Balancers (NLBs) on AWS and Google Cloud; or
2. outbound tunnels through a provider-neutral interface, initially supporting only Cloudflare Tunnels.

Both will integrate with scale-to-zero without placing the waiting room in the normal production data path. See [ZERO.md](./ZERO.md) for definition of "awake" / "asleep" terminology.

## Production routing

While Kubernetes is "awake":

- external port 80 and 443 routes directly to Traefik on cluster nodes;
- the configured external API port, defaulting to 6443, routes directly to kube-apiserver target port 6443 on control-plane nodes;

Important note: kube-apiserver cannot not depend on Traefik for ingress, as a broken ingress controller would lock administrators out from being able to fix it.

Nstance registers and deregisters healthy instances with AWS target groups or Google Cloud instance groups. Production registration must wait for provider load-balancer health, not only the first Nstance agent report.

## Scale-to-zero waiting room interoperability

Each `nst` VM runs a separate, minimal waiting-room process. The nstance-server process is never publicly exposed.

When a cluster is put to sleep, Nstance changes both external traffic paths to the waiting room before removing the final production targets. The waiting room:

1. accepts and holds connections for a configurable, long bounded timeout;
2. invokes `ResumeAllGroups` once through a root-owned local Unix socket;
3. waits for the requested upstream to become healthy;
4. forwards held connections;
5. continues serving established proxied connections while Nstance restores direct production routing.

The kube-apiserver and ingress health are independent: traffic waiting for one does not require the other to be ready - in other words, if the request which triggered a resume is for kube-apiserver, the request will be proxieid even before traefik is health.

Wake and routing operations are idempotent.

## Network Load Balancers & Waiting Room interopability

Podplane-generated Terraform supports equivalent AWS and Google Cloud behavior using provider-native resources.

AWS uses dedicated waiting target groups. Google Cloud waiting backends reference the existing Nstance-server managed instance groups, or both paths use `GCE_VM_IP` zonal NEGs; an `nst` VM is never added to the production unmanaged instance group.

Sleep transition:

1. register waiting-room targets and wait for load-balancer health;
2. keep wake behavior disarmed while production remains active;
3. switch new flows to the waiting backend and include any waiting-room arrival in the final `if_not_busy` check;
4. abort if traffic arrives, otherwise commit suspension;
5. deregister production targets only after the waiting path is available.

Wake transition:

1. resume groups and wait for requested upstream health;
2. register production targets and wait for load-balancer health;
3. direct new connections to production;
4. deregister waiting-room targets after its active connections drain.

nstance-server performs target membership changes because it already owns instance lifecycle and limits cloud API access to one security boundary. IAM permissions must be scoped to the cluster's target groups, instance groups, and health queries.

The design must account for provider fail-open behaviour and must never intentionally leave an empty production/waiting target set during transition.

## Nstance Server VM

This will require customisation, where today Podplane uses the current nstance module default userdata.

For `vmconfig` we add an `nst` kind, and use that instead of using Nstance's generic server userdata. It installs nstance-server, Fluent Bit, the waiting-room service described in [LB.md](./LB.md), and the standard vmconfig configuration/watch machinery.

nstance-server may write configured secrets atomically into a fixed receive directory e.g. for running a local tunnel service (per [LB.md](./LB.md)). A vmconfig-owned systemd path watcher validates them, copies them to service-specific destinations with strict ownership and modes, and restarts only affected services. Nstance remains unaware of Cloudflare-specific paths or service wiring.

Nstance Server VMs use private service subnets by default. When both a tunnel and Podplane Managed NAT are enabled, they require internet access independent of the suspendable NAT group:

- AWS places each `nst` VM in a public service subnet and assigns it a stable Elastic IP.
- Google Cloud assigns each `nst` VM a stable external IPv4; Google Cloud has no public/private subnet distinction.

Inbound security-group/firewall access remains denied except for explicitly configured load-balancer health and waiting-room paths, and the nstance-server process is never exposed publicly. Enabling this combination later may rotate Nstance Server VMs through their managed groups, but does not replace Kubernetes nodes.

## Tunnels

The tunnel feature contract is provider-neutral, with Cloudflare Tunnel as the initial implementation.

`cloudflared` runs on each `nst` VM and maintains outbound connectivity. In normal operation it forwards directly to Traefik or kube-apiserver, bypassing the waiting-room process, and nstance-server can auto-update upstream backends in its configuration file based on changes to the appropriately annotated groups (noting that the control-plane and ingress feature could be separate groups or one group). At zero nodes, Nstance switches its local upstream to the waiting room; DNS and the external tunnel endpoint remain stable.

Tunnel credentials are secrets which nstance-server fetches (e.g. from AWS Secrets Manager) and writes the configured secret atomically to its fixed receive directory; the `nst` vmconfig watcher installs it for `cloudflared` and restarts the service. Secrets must not appear in generated Terraform values, process arguments, logs, or world-readable files.

The mechanism for updating the `cloudflared` upstream backends could actually be a generic nstance-server file write, watched by the same secrets watch process, and therefore processed and distributed by a script which exists only in vmconfig - therefore simplifying the nstance-server implementation.

If many tunnels become supported, alternate `nst` vmconfig kinds could be created to keep configuration scoped to only the tunnel provider desired. But for now, it will remain a single kind, which only enables the configured tunnel (if any) based on an environment variable from userdata.

## Configuration

Cluster configuration selects NLB, tunnel, or custom exposure independently from application domain configuration. It defines stable external API and ingress endpoints while provider-specific settings remain under the selected infrastructure/tunnel provider.

Validation requires public/direct external placement for `nst` VMs when a tunnel and Podplane Managed NAT are both enabled. All other combinations default `nst` placement to private.

Scale-to-zero requires at least one exposure method capable of reaching the waiting room. Clusters without it may still be slept and woken explicitly through the CLI but cannot wake from external traffic.

## Implementation areas

- **Podplane CLI:** exposure config, validation, generated Terraform, and endpoint outputs.
- **Nstance Terraform modules:** waiting-room NLB targets and health checks for AWS and Google Cloud.
- **Nstance server:** target health/membership and atomic routing transitions.
- **vmconfig:** `nst` waiting-room and Cloudflare services, local routing, and secret watcher.
- **Components/Traefik:** retain direct host exposure on port 443; no kube-apiserver proxying.

Tests must cover API-only and ingress-only clusters, NLB and tunnel modes, partial upstream health, provider API failure, leader replacement, bounded connection holding, direct kube-apiserver access, and sleep/wake transitions without an empty route.
