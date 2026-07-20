# Scale-to-Zero Kubernetes Nodes

## Goal

Scale-to-zero should feel like **Wake-on-LAN for Kubernetes**. Podplane may stop every Kubernetes node, then wake the cluster for:

- an external kube-apiserver request;
- an external Traefik ingress request;
- an upcoming CronJob;
- an explicit administrative resume.

Nstance Server VMs remain running by default, normally one per zone/shard. They can use very small instance types because they provide the durable control plane while Kubernetes is asleep.

They remain in private service subnets by default. If Cloudflare Tunnel and Podplane Managed NAT are both enabled, their placement follows [LB.md](./LB.md): they receive independent direct internet egress so the tunnel remains available while the NAT group is suspended. Changing this placement may rotate Nstance Server VMs but never Kubernetes nodes.

## Durable group suspension

Suspension is a flag on each Nstance group. It preserves the group's desired configuration and size while making its effective size zero.

Nstance adds idempotent operations:

- `SuspendAllGroups`: mark every group suspended, with an optional `wake_at` and guarded `if_not_busy` behavior;
- `ResumeAllGroups`: clear suspension from every group.

This deliberately includes special groups such as Podplane NAT. On resume, normal reconciliation restores group intent; NAT dependencies ensure required NAT instances become healthy before nodes are launched. The waiting room needs no group names or previous-size knowledge, as it's a separate process from nstance-server.

These API operations are tenant/cluster-scoped, generation-based operations stored once in cluster object state. Each shard acknowledges and reconciles the generation independently; they do not depend on nstance-operator being alive during wake. One cluster-level owner runs the durable `wake_at` timer. A restart or leader change must not lose or duplicate a wake operation.

## Automatic suspension

Podplane operator initiates automatic suspension only when all of these hold:

1. exactly one Kubernetes node remains;
2. no Machine, MachinePool, or Nstance operation is awaiting scale-up;
3. no nonterminal or deleting Job exists;
4. no unsuspended CronJob is due within the configurable look-ahead window;
5. external ingress and kube-apiserver traffic have both been inactive for their independently configurable windows.

The operator sends the suspension request through nstance-operator. Nstance-server performs the final traffic check before committing suspension, closing the race between observation and VM removal. Suspension uses orderly workload termination; forced suspension may still interrupt workloads because the final node has nowhere to drain to.

## Traffic activity

External kube-apiserver exposure remains direct and never depends on Traefik.

Nstance agent extends its existing health report with fixed counters for inbound application payload on target ports 80, 443, and 6443:

- cumulative bytes and packets;
- active data-bearing TCP connections;
- last activity time for each port.

Only externally originated traffic counts. Classification covers direct public-client sources, AWS NLB addresses used for IPv6-to-IPv4 translation, and configured `nst`/cloudflared connector addresses while excluding other cluster/VPC traffic. Observation must cover Cilium's eBPF host-port path rather than assuming netfilter conntrack. Reports contain only counters and timestamps, never client addresses or connection tuples.

TCP load-balancer health probes normally establish a connection without application payload and therefore do not count. An active data-bearing connection keeps the node busy, conservatively preserving Kubernetes watches, exec, logs, and port-forwarding.

Unsupported or stale activity telemetry is treated as busy, never idle.

## CronJob wake deadlines

Before suspension, Podplane operator finds the earliest future execution across all unsuspended CronJobs, honoring Kubernetes cron syntax and `.spec.timeZone`. It persists:

```text
wake_at = next_run - wake_lead_time
```

The lead time defaults to two minutes and is configurable. Nstance wakes the cluster when the deadline arrives, even without network traffic. After every wake, Podplane recomputes the next deadline because schedules may have changed.

Any Job without terminal `Complete` or `Failed` status blocks suspension; deleting Jobs block until deletion completes. A suspended Job is exempt only when it has no active or terminating Pods. CronJob `status.active` is advisory; ownership is matched by UID. Suspended CronJobs do not schedule wakes.

## Wake sequence

Wake is deduplicated with a durable compare-and-swap from `asleep` to `waking`. Only the process which performs that transition starts the background wake operation; concurrent network, timer, or administrative triggers observe `waking` and reuse the same operation instead of starting another. If leadership changes, the new owner resumes that wake generation.

1. The waiting room invokes the single local, cluster-scoped `ResumeAllGroups` operation.
2. Nstance clears durable suspension and reconciles infrastructure.
3. Required NAT instances become healthy before dependent nodes.
4. Nstance waits for the kube-apiserver and/or Traefik upstream requested by held traffic.
5. Held connections are forwarded.
6. Production NLB or tunnel routing bypasses the waiting room; existing proxied connections may finish there.

The local operation is exposed only over a root-owned Unix socket. Nstance-server remains inaccessible from the public network.

## CLI

Podplane adds:

```sh
podplane cluster sleep
podplane cluster sleep --force
podplane cluster wake
```

Normal sleep applies the automatic-suspension guards. `--force` bypasses node-count, pending-scale, Job, CronJob, and traffic checks after explicit confirmation. It still requests orderly shutdown but warns that active workloads and connections can be interrupted. `wake` invokes the same idempotent resume operation used by the waiting room.

## Implementation areas

- **Podplane operator:** eligibility, Job/CronJob evaluation, wake deadline, and nstance-operator request.
- **Nstance operator:** suspension coordination across shard groups.
- **Nstance server:** durable suspension, timers, final busy check, and resume reconciliation.
- **Nstance agent:** privacy-preserving per-port activity counters.
- **vmconfig:** `nst` kind, waiting room, secret receive watcher, and services.
- **Podplane CLI:** sleep/wake commands and generated configuration.
- **Components:** deploy/configure nstance-operator and the required operator permissions.

Tests must cover races, stale metrics, active/terminating Jobs, CronJob time zones and deadlines, leader failover, multi-zone resume, forced sleep, NAT dependencies, and simultaneous network/timer wakeups.
