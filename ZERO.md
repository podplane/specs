# Scale-to-Zero Kubernetes Nodes

## Goal

Scale-to-zero should feel like **Wake-on-LAN for Kubernetes**. Podplane may stop every Kubernetes node, then wake the cluster for:

- an external kube-apiserver request;
- an external Traefik ingress request;
- an upcoming CronJob;
- an explicit administrative wake.

Nstance Server VMs remain running by default, normally one per zone/shard. They can use very small instance types because they provide the durable control plane while Kubernetes is asleep.

They remain in private service subnets by default. Podplane Managed NAT places them in public service subnets/direct external egress as described in [NAT.md](./NAT.md), allowing tunnels and optional small-cluster NAT to remain available while tenant groups are asleep. Changing this placement may rotate Nstance Server VMs but never Kubernetes nodes.

## Durable tenant state

A Podplane Kubernetes cluster can be awake or asleep. In Nstance, sleep state is tracked independently for each tenant in each shard. Sleeping sets every local tenant group to an effective size of zero and terminates its VMs while preserving desired sizes; Nstance-server VMs remain running. It does not suspend, hibernate, or preserve VM execution state. “Going to sleep” and “waking” describe incomplete reconciliation rather than additional persisted states.

Each shard stores durable state under `shard/<shard>/sleep.jsonc`:

```jsonc
{
  "default": {
    "desired_state": "asleep",
    "generation": 12,
    "wake_at": "2026-07-22T04:58:00Z"
  }
}
```

Nstance adds idempotent operations:

- `SleepTenant`: set the tenant's desired state to `asleep` in the receiving server's shard, with optional `wake_at` and guarded `if_not_busy` behavior;
- `WakeTenant`: set the tenant's desired state to `awake` in that shard.

This includes the tenant's configured NAT group in that shard. On wake, normal reconciliation restores desired sizes; NAT dependencies ensure required NAT instances become healthy before nodes launch. `nstance-proxy` uses configured listener-to-group mappings and needs no previous-size knowledge.

The shard leader reconciles its local record and resumes unfinished work after leadership changes. Nstance does not coordinate sleep/wake through the cluster leader or contact other shards.

## Supported topologies

Nstance sleep/wake is not limited by node count:

- **One node to zero:** the shard preserving that node's desired size advertises `nstance-proxy`; a request wakes that shard.
- **Two nodes to zero across two zones:** each shard preserves one desired node and advertises `nstance-proxy`. A request wakes whichever shard receives it; once Kubernetes starts, nstance-operator restores the other shard. Concurrent requests may wake both shards, which is safe.

Shards do not advertise a listener's wake target when its configured group has preserved size zero because waking that shard would not restore the listener's upstream.
Nstance supports both scenarios; the initial Podplane automatic policy permits only the first, while forced sleep can use either.

## Automatic sleep

Podplane operator initiates automatic sleep only when all of these hold:

1. exactly one Kubernetes node remains;
2. no Machine, MachinePool, or Nstance operation is awaiting scale-up;
3. no nonterminal or deleting Job exists;
4. no CronJob with `.spec.suspend=false` is due within the configurable look-ahead window;
5. external ingress and kube-apiserver traffic have both been inactive for their independently configurable windows.

Exactly one node is an initial Podplane automatic-sleep policy, not an Nstance limitation. nstance-operator calls `SleepTenant` for every shard, processing the shard hosting its coordinating Kubernetes node last. Each Nstance-server performs the final local traffic check before committing sleep, closing the race between observation and VM removal. Sleep uses orderly workload termination; forced sleep may still interrupt workloads because the final node has nowhere to drain to.

## Traffic activity

External kube-apiserver exposure remains direct and never depends on Traefik.

Nstance derives activity ports from `proxy.listeners`. An agent reports fixed, low-cardinality counters for each configured listener whose group matches that instance:

- cumulative bytes and packets;
- active data-bearing TCP connections;
- last activity time for each port.

Only externally originated traffic counts. Classification covers direct public-client sources, AWS NLB addresses used for IPv6-to-IPv4 translation, and configured `nst`/cloudflared connector addresses while excluding other cluster/VPC traffic. Observation must cover Cilium's eBPF host-port path rather than assuming netfilter conntrack. Reports contain only counters and timestamps, never client addresses or connection tuples.

Every configured listener participates in `if_not_busy`; no separate activity flag is required. TCP load-balancer health probes normally establish a connection without application payload and therefore do not count. An active data-bearing connection keeps the node busy, conservatively preserving Kubernetes watches, exec, logs, and port-forwarding.

Unsupported or stale activity telemetry is treated as busy, never idle.

## CronJob wake deadlines

Before sleep, Podplane operator finds the earliest future execution across all CronJobs whose Kubernetes `.spec.suspend` is false, honoring Kubernetes cron syntax and `.spec.timeZone`. It persists:

```text
wake_at = next_run - wake_lead_time
```

The lead time defaults to two minutes and is configurable. nstance-operator installs the deadline on one wake-capable shard; that shard wakes when it arrives, even without network traffic. After every wake, Podplane recomputes the next deadline because schedules may have changed.

Any Job without terminal `Complete` or `Failed` status blocks sleep; deleting Jobs block until deletion completes. A Kubernetes-suspended Job is exempt only when it has no active or terminating Pods. CronJob `status.active` is advisory; ownership is matched by UID. Kubernetes-suspended CronJobs do not schedule wakes.

## Wake sequence

Wake is deduplicated within a shard by a durable compare-and-swap from desired `asleep` to desired `awake`, incrementing the generation. The first network, timer, or administrative trigger performs the transition; concurrent triggers observe the same awake generation.

1. `nstance-proxy` invokes the local, tenant-scoped `WakeTenant` operation.
2. Nstance commits desired `awake` with a new shard generation and reconciles local infrastructure.
3. Required NAT instances become healthy before dependent nodes.
4. Nstance waits for the requested listener's configured group and target port.
5. Held connections are forwarded.
6. Once Kubernetes starts, nstance-operator wakes any other shards required by MachinePools.
7. Nstance waits for the listener's production NLB target or tunnel connector, then bypasses `nstance-proxy`; existing proxied connections may finish there.

The local `nstance-proxy` can invoke only `WakeTenant` over a root-owned Unix socket. Nstance-server remains inaccessible from the public network.

## CLI

Podplane adds:

```sh
podplane cluster sleep
podplane cluster sleep --force
podplane cluster wake
```

Normal sleep applies the automatic-sleep guards and coordinates all shards. `--force` bypasses node-count, pending-scale, Job, CronJob, and traffic checks after explicit confirmation. It still requests orderly shutdown but warns that active workloads and connections can be interrupted. `wake` invokes the same idempotent `WakeTenant` operation on one wake-capable shard; nstance-operator restores any additional required shards after Kubernetes starts.

## Implementation areas

- **Podplane operator:** eligibility, Job/CronJob evaluation, wake deadline, and nstance-operator request.
- **Nstance operator:** all-shard sleep ordering and post-wake MachinePool restoration.
- **Nstance server:** durable per-tenant shard state, timers, final busy check, and local reconciliation.
- **Nstance agent:** privacy-preserving activity counters for configured proxy listeners.
- **vmconfig:** `knc`/`nst` tunnel services, `nst` kind, optional `nstance-proxy`, and secret receive watchers.
- **Podplane CLI:** sleep/wake commands and generated configuration.
- **Components:** deploy/configure nstance-operator and the required operator permissions.

Tests must cover one-node and redundant two-zone sleep, all-shard sleep ordering, local wake, simultaneous wakes in different shards, zero-sized shard exclusion, stale metrics, active/terminating Jobs, CronJob time zones/deadlines, simultaneous network/timer wakeups, shard-leader failover, forced sleep, and NAT dependencies.
