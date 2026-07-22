# Scale-to-Zero Kubernetes Nodes

## Goal

Scale-to-zero should feel like **Wake-on-LAN for Kubernetes**. Podplane may stop every Kubernetes node, then wake the cluster for:

- an external kube-apiserver request;
- an external Traefik ingress request;
- an upcoming CronJob.

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
Nstance supports both scenarios; the initial nstance-operator automatic policy permits only the first, while forced sleep can use either.

## Automatic sleep

nstance-operator owns generic Kubernetes automatic-sleep policy. Podplane enables and configures it rather than implementing Kubernetes eligibility itself. The feature is disabled by default; initial defaults are one remaining node, 30 minutes of inactivity, a ten-minute CronJob look-ahead, and a two-minute wake lead. Activity windows remain independently configurable per listener.

When enabled, nstance-operator initiates automatic sleep only when all of these hold:

1. exactly one Kubernetes node remains;
2. no Machine, MachinePool, or Nstance operation is awaiting scale-up;
3. no nonterminal or deleting Job exists;
4. no CronJob with `.spec.suspend=false` is due within the configurable look-ahead window;
5. external ingress and kube-apiserver traffic have both been inactive for their independently configurable windows.

Exactly one node is the initial default, not an Nstance limitation. nstance-operator calls `SleepTenant` for every shard, processing the shard hosting its coordinating Kubernetes node last. Each Nstance-server performs the final local traffic check before committing sleep, closing the race between observation and VM removal. Sleep uses orderly workload termination; forced sleep may still interrupt workloads because the final node has nowhere to drain to.

## Traffic activity

External kube-apiserver exposure remains direct and never depends on Traefik.

Each control-plane or ingress node uses eBPF to count active TCP connections on the production ports appropriate to its role:

- control-plane only: kube-apiserver port 6443;
- ingress only: Traefik port 443;
- combined control-plane and ingress: ports 6443 and 443.

Podplane supplies these roles through generated Nstance group configuration, and vmconfig initializes only the corresponding port keys. This supports clusters where control-plane and ingress instances share a group or use separate groups.

### Loading and pinning

vmconfig installs Debian's `bpftool` and a root-owned systemd oneshot service on control-plane and ingress nodes. The service loads the vmconfig-supplied ELF, infers tracepoint attachments from section names, uses `autoattach` to pin the resulting BPF links, and uses `pinmaps` to pin maps separately:

```text
/sys/fs/bpf/nstance/counters/
├── links/
└── maps/
    └── active_connections
```

Both links and maps are pinned: maps remain readable independently, while links keep programs attached after bpftool exits. The service grants nstance-agent read access only to the pins it must validate and the counter map. nstance-agent remains unprivileged and never loads or modifies BPF objects.

### Agent reporting

vmconfig enables collection with:

```ini
NSTANCE_EBPF_COUNTERS_PATH=/sys/fs/bpf/nstance/counters
```

nstance-agent validates the pinned links, opens `maps/active_connections` read-only using `cilium/ebpf`, and includes every map entry in its normal health report:

```jsonc
{
  "ebpf_counters": {
    "443": 0,
    "6443": 0
  }
}
```

If link validation or a configured map read fails, the report includes `ebpf_error`; otherwise the error is omitted. If the environment variable is unset, or a valid map has no entries, no counters are reported and sleep is not blocked.

### Final sleep guard

`SleepTenant(if_not_busy=true)` checks every remaining tenant instance in the local shard using standard agent-health freshness. A non-empty `ebpf_error` or any nonzero reported counter refuses the entire shard's sleep; zero or absent counters permit it. No server-side sleep configuration is required. Forced sleep bypasses this check. Reports never include client addresses or connection tuples.

## CronJob wake deadlines

Before sleep, nstance-operator finds the earliest future execution across all CronJobs whose Kubernetes `.spec.suspend` is false, honoring Kubernetes cron syntax and `.spec.timeZone`. It persists:

```text
wake_at = next_run - wake_lead_time
```

The lead time defaults to two minutes and is configurable. nstance-operator installs the deadline on one wake-capable shard; that shard wakes when it arrives, even without network traffic. After every wake, nstance-operator recomputes the next deadline because schedules may have changed.

Any Job without terminal `Complete` or `Failed` status blocks sleep; deleting Jobs block until deletion completes. A Kubernetes-suspended Job is exempt only when it has no active or terminating Pods. CronJob `status.active` is advisory; ownership is matched by UID. Kubernetes-suspended CronJobs do not schedule wakes.

## Wake sequence

Wake is deduplicated within a shard by a durable compare-and-swap from desired `asleep` to desired `awake`, incrementing the generation. The first network or timer trigger performs the transition; concurrent triggers observe the same awake generation.

1. `nstance-proxy` invokes the local, listener-scoped `WakeTenant` operation and holds the connection.
2. Nstance commits desired `awake` with a new shard generation and reconciles local infrastructure.
3. Required NAT instances become healthy before dependent nodes.
4. Nstance waits for any configured group to provide an agent-healthy instance with the target port ready, then returns its private address.
5. `nstance-proxy` forwards held connections to that address.
6. Once Kubernetes starts, nstance-operator wakes any other shards required by MachinePools.
7. Nstance waits for production NLB registration or tunnel-connector activation to be accepted, then bypasses `nstance-proxy`; existing proxied connections may finish there.

The local `nstance-proxy` can invoke only `WakeTenant` over a root-owned Unix socket. Nstance-server remains inaccessible from the public network.

## CLI

Podplane adds:

```sh
podplane cluster sleep
podplane cluster sleep --force
```

The command uses `SelfSubjectAccessReview` to verify that the user may patch the CAPI `Cluster`, then writes `nstance.dev/sleep-request=<timestamp>` or `force:<timestamp>` and disconnects immediately. nstance-operator emits a Kubernetes Event and removes the annotation on rejection, or calls `SleepTenant` for every shard and removes it after all requests are durable. If the operator restarts first, the remaining annotation causes a safe retry. Initially this is limited to `podplane:admins`, which are bound to `cluster-admin`; nstance-operator requires CAPI Cluster get/list/watch/patch/update permissions.

Normal sleep applies the automatic-sleep guards. `--force` bypasses node-count, pending-scale, Job, CronJob, and traffic checks after explicit confirmation, but still requests orderly shutdown and warns that active workloads and connections can be interrupted. There is no wake command: any kubectl, Helm, kube-apiserver, or ingress request reaches `nstance-proxy` and wakes a shard.

## Implementation areas

- **Nstance operator:** configurable Kubernetes eligibility, Job/CronJob evaluation, wake deadlines, all-shard sleep ordering, and post-wake MachinePool restoration.
- **Nstance server:** durable per-tenant shard state, timers, final busy check, and local reconciliation.
- **Nstance agent:** read-only pinned-map collection and eBPF counter/error reporting.
- **vmconfig:** `knc`/`knd` BPF object, bpftool oneshot loader, pin permissions, `knc`/`nst` tunnel services, `nst` kind, optional `nstance-proxy`, and secret receive watchers.
- **Podplane CLI:** generated sleep policy and the annotation-based sleep command.
- **Components:** deploy/configure nstance-operator and the required operator permissions.

Tests must cover one-node and redundant two-zone sleep, all-shard sleep ordering, local wake, simultaneous wakes in different shards, zero-sized shard exclusion, zero, nonzero, absent, failed, and stale eBPF reports, pinned-link loss, active/terminating Jobs, CronJob time zones/deadlines, simultaneous network/timer wakeups, shard-leader failover, forced sleep, and NAT dependencies.
