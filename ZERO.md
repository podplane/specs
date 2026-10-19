# Scale-to-Zero Kubernetes Nodes

> **STATUS**: In progress

## Goal

Scale-to-zero should feel like **Wake-on-LAN for Kubernetes**. Podplane may stop every Kubernetes node, then wake the cluster for:

- an external kube-apiserver request;
- an external Envoy Gateway ingress request;
- an upcoming CronJob.

Nstance Server VMs remain running by default, normally one per zone/shard. They can use very small instance types because they provide the durable control plane while Kubernetes is asleep. Their `nst` configuration is provisioned once; changing it rolls the VMs rather than applying an agent runtime update.

They remain in private service subnets by default. `cluster.nat.mode = "nstance"` places them in public service subnets/direct external egress as described in [NAT.md](./NAT.md), allowing tunnels to remain available while tenant groups are asleep. Changing this placement may rotate Nstance Server VMs but never Kubernetes nodes. NAT mode (`provider` or `nstance`) is independent of using NLBs or tunnels.

## Durable tenant state

A Podplane Kubernetes cluster can be awake or asleep. In Nstance, sleep state is tracked independently for each tenant in each shard. Sleeping sets every local tenant group to an effective size of zero and terminates its VMs while preserving desired sizes; Nstance-server VMs remain running. It does not suspend, hibernate, or preserve VM execution state. “Going to sleep” and “waking” describe incomplete reconciliation rather than additional persisted states.

Each shard stores machine-managed dynamic tenant state in `shard/<shard>/tenants.jsonc`, separate from the administrator-managed `tenants` configuration in `config.jsonc`:

```jsonc
{
  "default": {
    "sleep": {
      "wake_at": "2026-07-22T04:58:00Z"
    }
  }
}
```

A tenant's `sleep` entry means asleep; its absence means awake. `wake_at` is omitted when no timer wake is configured. A tenant key is removed when it has no dynamic state, and the empty object is retained when no tenant has dynamic state. Future tenant-scoped runtime features may use explicit sibling fields; arbitrary extension fields are not supported. Updates use the storage object's native ETag or generation as the compare-and-swap boundary and retry a conflicting read-modify-write; the document has no separate application-level generation.

Nstance adds idempotent operations:

- `SleepTenant`: its request carries `tenant`, `if_not_busy`, and optional `wake_at`; its response returns `SLEPT`, `ALREADY_ASLEEP`, or `BUSY`, the current `AWAKE` or `ASLEEP` status, and the effective optional `wake_at`;
- `WakeTenant`: its request carries `tenant` and an optional listener identity; its response returns `WOKE` or `ALREADY_AWAKE`, the current status, and an optional ready upstream. A listener-scoped wake waits for and returns an upstream, while an unscoped compensation wake does not require one.

This includes the tenant's configured NAT group in that shard. On wake, normal reconciliation restores desired sizes; NAT dependencies ensure required NAT instances become healthy before nodes launch. `nstance-server proxy` uses configured listener-to-group mappings and needs no previous-size knowledge.

During normal or forced sleep, Nstance keeps each NAT instance until its last
dependent workload is removed, then removes it on the next reconciliation
without the grace period used for awake tenants. Fixed public IPs stay reserved
for reuse, as specified in [NAT.md](./NAT.md).

The shard leader reconciles its local record and resumes unfinished work after leadership changes. Nstance-server does not enumerate, contact, coordinate with, or wait for other shards. Sleep, wake, routing cutovers, and instance reconciliation use only the current shard's state and resources. Cross-shard Kubernetes policy belongs to nstance-operator, while administrative cross-shard operations belong to nstance-admin. On AWS, OpenTofu/Terraform keeps cross-zone routing disabled so traffic arriving in a zone reaches only that shard's production or wake-proxy targets, as described in [LB.md](./LB.md).

Local wake-tunnel ownership is deliberately ephemeral rather than part of durable tenant state. The privileged root `nstance-server tunnel` process initiates its local Unix gRPC stream to the unprivileged server process, independently reads the same root-controlled shard configuration, and resolves tunnel secret/storage files itself. The shard leader sends only allowlisted tunnel name, desired running/stopped state, revision, and a renewable lease; it never sends images, commands, mounts, manifests, secrets, or provider details. Heartbeats renew authority while leadership is held. Explicit leadership loss revokes immediately; an ambiguous disconnect retains current state only for the bounded lease while reconnecting, then stops fail-closed. A restarted supervisor removes stale manifests before loading configuration because its leases were lost. The new leader waits for matching revision readiness before continuing destructive reconciliation. Nstance maintains no distributed old/new-leader tunnel handoff or lifecycle state files.

Node and `nst` tunnel processes use the same existing user-managed tunnel ID and pre-created credential-secret reference; only their local origins and lifecycles differ. Podplane does not provision the tunnel or read its credentials through Terraform. See [LB.md](./LB.md#configuration) for setup and the unimplemented metadata-only preflight proposal.

Tunnel ingress uses one node-local certificate/keypair per Envoy-capable node, signed by the existing Nstance cluster CA and covering all configured apex/wildcard domains, localhost, both loopback IPs, and that node's assigned IPv4/IPv6 addresses. The agent provisioning mechanism matches kube-apiserver's; the Podplane operator performs neither ingress fallback nor ACME issuance in tunnel mode. Both node and `nst` tunnels trust the public cluster CA and verify the configured origin hostname. After wake, the proxy forwards TLS unchanged to a newly provisioned node's certificate, so recovery needs neither an old node's private key nor operator ingress issuance. NLB deployments retain operator-owned fallback/ACME bundles. See [LB.md](./LB.md#ingress-certificates-and-sds) for certificate delivery, lifetime, and SDS requirements.

## Supported topologies

Nstance sleep/wake is not limited by node count:

- **One node to zero:** the shard preserving that node's desired size advertises `nstance-server proxy`; a request wakes that shard.
- **Two nodes to zero across two zones:** each shard preserves one desired node and advertises `nstance-server proxy`. A request wakes whichever shard receives it; once Kubernetes starts, nstance-operator restores the other shard. Concurrent requests may wake both shards, which is safe.

Shards do not advertise a listener's wake target when its configured group has preserved size zero because waking that shard would not restore the listener's upstream.
Nstance supports both scenarios; the initial nstance-operator automatic policy permits only the first, while forced sleep can use either.

## Automatic sleep

nstance-operator owns generic Kubernetes automatic-sleep policy. Podplane user config has no `SleepPolicy` and Podplane generates no policy or timing overrides; Nstance/operator defaults own those settings. Automatic sleep is disabled by default; initial defaults are one remaining node, 30 minutes of inactivity, a ten-minute CronJob look-ahead, and a two-minute wake lead. Nstance and Components retain their own policy configurability, including independently configurable activity windows per listener.

Supported Podplane AWS/Google clusters using the `platform` or `agentic` seed generate wake-capable routing and proxy services without a sleep-policy flag, even while automatic sleep is disabled. Manual CLI sleep remains supported. Exposure uses provider-default NLBs for both API and ingress when the singular optional `cluster.tunnel` config is absent, or tunnels for both with no NLB resources or registrations when present, as defined in [LB.md](./LB.md). Changing that mode requires infrastructure deletion/redeployment, not an in-place switch; retaining Netsy data alone is not a complete state-preserving migration. NAT mode remains independent under [NAT.md](./NAT.md).

When enabled, nstance-operator initiates automatic sleep only when all of these hold:

1. exactly one Kubernetes node remains;
2. no Machine, MachinePool, or Nstance operation is awaiting scale-up;
3. no nonterminal or deleting Job exists;
4. no CronJob with `.spec.suspend=false` is due within the configurable look-ahead window;
5. external ingress and kube-apiserver traffic have both been inactive for their independently configurable windows.

Exactly one node is the initial default, not an Nstance limitation. On a self-managed cluster, nstance-operator runs on a control-plane node and calls `SleepTenant` for every shard with that node's shard last. Any in-cluster etcd, Netsy, or other datastore quorum required by the Kubernetes API is wholly contained in that final shard; alternatively, it remains external to the sleeping cluster. This topology keeps the API available while earlier shards sleep. Before the final call, the operator persists a `finalizing` transaction checkpoint containing the final shard and every shard that has already accepted sleep. No successful final-shard call requires a later Kubernetes write. Each Nstance-server performs the final local traffic check before committing sleep, closing the race between observation and VM removal. Sleep uses orderly workload termination; forced sleep may still interrupt workloads because the final node has nowhere to drain to.

The all-shard operation is compensating rather than atomic. The sleep-request and progress annotations remain present while the cluster is asleep. After Kubernetes returns, nstance-operator reads every shard's authoritative status; if any shard is awake, it durably calls `WakeTenant` for all shards and removes the annotations only after every shard is awake. If any shard rejects or fails before the final checkpoint, nstance-operator similarly wakes every shard whose sleep may have succeeded and retries compensation until their sleep entries have been removed; only then does it emit the rejection Event and remove the annotations. An operator restart resumes forward progress, finalization, or compensation from the annotations and observed shard state. This prevents control-plane shutdown or a failed request from leaving the cluster indefinitely partially asleep.

## eBPF port activity monitoring

External kube-apiserver exposure remains direct and never depends on Envoy Gateway.

Each Kubernetes group that can participate in guarded normal sleep uses eBPF to count active TCP connections on every production target port derived from its referenced `load_balancers` listeners. vmconfig configures ingress ports only, kube-apiserver port 6443 only, their union for a group serving both roles, or no counters for a group serving neither role. Standard NLB ingress therefore counts ports 80 and 443, while tunnel ingress counts port 443. This listener-derived configuration supports shared or separate control-plane and ingress groups without duplicating the port model in role configuration.

### Loading and pinning

For groups requiring active connection counters, vmconfig installs Debian's `bpftool` and a root-owned systemd oneshot service. The service loads the vmconfig-supplied ELF, ignores ports outside the listener-derived set, infers tracepoint attachments from section names, uses `autoattach` to pin the resulting BPF links, and uses `pinmaps` to pin maps separately:

```text
/sys/fs/bpf/nstance/counters/
├── links/
└── maps/
    └── active_connections
```

Both links and maps are pinned: maps remain readable independently, while links keep programs attached after bpftool exits. The service grants nstance-agent read access only to the pins it must validate and the counter map. nstance-agent remains unprivileged and never loads or modifies BPF objects. vmconfig installs the small active-connection counter toolchain on every Kubernetes node, but when no production ports require monitoring it does not enable the loader, attach programs, create pins, or configure agent collection.

### Agent reporting

vmconfig enables collection with:

```ini
NSTANCE_EBPF_COUNTERS_PATH=/sys/fs/bpf/nstance/counters
```

nstance-agent validates the pinned links, opens `maps/active_connections` read-only using `cilium/ebpf`, and includes every map entry in its normal health report:

```jsonc
{
  "metrics": {
    "ebpf_counters": {
      "443": 0,
      "6443": 0
    }
  }
}
```

If link validation or a configured map read fails, the report includes `metrics.ebpf_error`; otherwise the error is omitted. If the environment variable is unset, or a valid map has no entries, no counters are reported. For an instance in a group referenced by a proxy listener, an error or missing expected port blocks normal sleep; Nstance derives those ports from the listener configuration. Other instances need no counters.

### Final sleep guard

Periodic agent reports provide the initial inactivity signal. For the final guard, `SleepTenant(if_not_busy=true)` establishes the proxy path and starts the provider-specific production-target withdrawal described in [LB.md](./LB.md). Once the provider confirms that production targets are draining and receive no new connections, Nstance waits for a subsequent normal health report from each remaining relevant agent.

Nstance refuses sleep if a required report is missing, contains `metrics.ebpf_error`, reports a nonzero active count, or if proxy payload arrives during the transition. On refusal, Nstance restores production routing before withdrawing the proxy path. Otherwise it waits for production targets to finish deregistering, commits the tenant's sleep entry, and begins instance termination. Forced sleep bypasses the activity guard and remaining drain period, but still requires a healthy wake path and provider confirmation that production targets no longer receive new connections. Instances may terminate while old connections are still draining. Reports never include client addresses or connection tuples.

## CronJob wake deadlines

Before sleep, nstance-operator finds the earliest future execution across all CronJobs whose Kubernetes `.spec.suspend` is false, honoring Kubernetes cron syntax and `.spec.timeZone`. It persists:

```text
wake_at = next_run - wake_lead_time
```

The lead time defaults to two minutes and is configurable. nstance-operator installs the deadline on one wake-capable shard; that shard wakes when it arrives, even without network traffic. After every wake, nstance-operator recomputes the next deadline because schedules may have changed.

Any Job without terminal `Complete` or `Failed` status blocks sleep; deleting Jobs block until deletion completes. A Kubernetes-suspended Job is exempt only when it has no active or terminating Pods. CronJob `status.active` is advisory; ownership is matched by UID. Kubernetes-suspended CronJobs do not schedule wakes.

## Wake sequence

Wake is deduplicated within a shard by a durable compare-and-swap that removes the tenant's sleep entry. The first network or timer trigger performs the transition; concurrent triggers observe that the entry is already absent and succeed idempotently.

1. `nstance-server proxy` invokes the local, listener-scoped `WakeTenant` operation and holds the connection.
2. Nstance removes the tenant's sleep entry and reconciles local infrastructure toward awake.
3. Required NAT instances become healthy before dependent nodes.
4. Nstance waits for any configured group to provide an agent-healthy instance with the target port ready, then returns its private address.
5. `nstance-server proxy` forwards held connections to that address.
6. Once Kubernetes starts, nstance-operator wakes any other shards required by MachinePools.
7. Nstance waits for production NLB targets to become provider-health-check healthy and routable. For tunnel exposure, it waits for ordinary fresh control-plane agent health and the requested upstream TCP port; node tunnels remain VM-lifecycle-owned and outside Nstance control. It then bypasses `nstance-server proxy`, allowing existing proxied connections to finish there. If readiness times out, it retains the proxy path and continues reconciliation. For the local `nst` wake tunnel, Nstance renews only the allowlisted name, desired running/stopped state, revision, and bounded lease over the supervisor-initiated Unix gRPC stream.

The local `nstance-server proxy` uses one root-owned, proxy-accessible Unix gRPC control socket for its versioned listener stream and listener-scoped `WakeTenant`; these are the only proxy-facing operations. On stream loss it retains the last valid snapshot only during a brief reconnect window and stops leader-owned work. Nstance-server remains inaccessible from the public network, and no proxy configuration is persisted.

## Operator registration bootstrap

Podplane supports scale-to-zero only for AWS and Google Cloud clusters using the `platform` or `agentic` seed. nstance-operator requires Nstance CRDs, Cluster API core and CRDs, and the Pod Certificate stack described in [CERTS.md](./CERTS.md#platform-serving-certificates-and-ca-injection). CAPI and Nstance project webhook Service certificates signed by `podplane-operator`, which also injects the workload CA into their webhook configurations. Its CA key is mounted through Secrets Store CSI and the selected provider; cert-manager is not used. The Nstance/CAPI provider overlay is not added to `minimal` or `none`.

Webhook TLS is separate from operator registration with Nstance: the workload CA does not replace Nstance's public CA or registration nonce, and Nstance registration does not use the Podplane Secrets API or CSI.

For supported clusters, Terraform conditionally creates or adopts Nstance's registration signing key, then the cluster-specific Netsy seed creation operation reads that key in memory, mints a tenant- and cluster-scoped single-use nonce with a 30-minute TTL, and inserts the nonce as a dedicated Kubernetes Secret. It reads the signing key only when creating the seed; refresh and completed-seed adoption use non-secret metadata. The public CA is seeded separately as a ConfigMap. Neither the signing key nor nonce appears in generic seeds, Terraform schema values/state/output, logs, diagnostics, command arguments, or temporary files.

nstance-operator deletes the nonce Secret only after durably storing and verifying its key and certificate. Registration is retry-safe after a lost response for the exact nonce JWT and byte-identical operator public-key PEM; a different PEM is rejected. An expired nonce on a live, unregistered cluster is replaced manually using the Nstance admin nonce command and `kubectl`; a cluster that never becomes live is destroyed and recreated. Initial seeds are never re-uploaded over live Netsy state.

## CLI

Podplane adds:

```sh
podplane cluster sleep
podplane cluster sleep --force
```

The command uses `SelfSubjectAccessReview` to verify that the user may patch the
CAPI `Cluster`, then writes `nstance.dev/sleep-request=<timestamp>` or
`force:<timestamp>` and disconnects immediately. nstance-operator calls
`SleepTenant` for every shard, persisting the finalizing checkpoint before the
last call. The request and progress annotations remain throughout sleep; no
Kubernetes write is required after the final shard accepts sleep. After wake,
the operator removes them only when every shard is awake. On rejection after a
partial commit, it completes durable all-shard compensation before emitting the
rejection Event and removing the annotations. An operator restart resumes the
transaction from those annotations and authoritative shard status.

Initially this is limited to `podplane:admins`, which are bound to
`cluster-admin`; nstance-operator requires CAPI Cluster
get/list/watch/patch/update permissions.

Normal sleep applies the automatic-sleep guards. `--force` bypasses node-count, pending-scale, Job, CronJob, and traffic checks after explicit confirmation, but still requests orderly shutdown and warns that active workloads and connections can be interrupted. There is no wake command: any kubectl, Helm, kube-apiserver, or ingress request reaches `nstance-server proxy` and wakes a shard.

## Implementation areas

- **Nstance operator:** configurable Kubernetes eligibility, Job/CronJob evaluation, wake deadlines, all-shard sleep ordering, and post-wake MachinePool restoration.
- **Nstance server:** durable per-tenant shard state, timers, final busy check, local reconciliation, and narrow proxy/supervisor Unix gRPC control streams.
- **Nstance tunnel supervisor:** privileged root config/file resolution, supervisor-initiated control stream, renewable-lease enforcement, hardened kubelet static-Pod generation, and readiness/restart handling.
- **Nstance agent:** read-only pinned-map collection and eBPF counter/error reporting.
- **vmconfig:** dormant-by-default `knc`/`knd` eBPF connection counters, bpftool oneshot loader, pin permissions, VM-lifecycle-owned node tunnels, provision-once `nst` configuration, and installation/hardening of separate optional `nstance-server proxy` and `nstance-server tunnel` processes.
- **Podplane CLI:** wake-capable routing for supported platform/agentic deployments and the annotation-based manual sleep command; no user `SleepPolicy` or generated policy/timing overrides.
- **Components:** deploy/configure nstance-operator and the required operator permissions.

Tests must cover one-node and redundant two-zone sleep, all-shard sleep ordering, durable compensation when a later shard rejects or fails, operator restart during compensation, local wake, simultaneous wakes in different shards, zero-sized shard exclusion, listener-derived port sets including NLB ingress port 80, zero, nonzero, absent, failed, and stale eBPF reports, traffic during production-target draining, pinned-link loss, active/terminating Jobs, CronJob time zones/deadlines, simultaneous network/timer wakeups, shard-leader failover, Unix stream reconnect and snapshot replacement, in-place listener reconciliation, supervisor-initiated reconnect, heartbeat renewal, immediate explicit revocation, bounded ambiguous-disconnect behavior, fail-closed lease expiry, hardened static Pods, dedicated and shared-agent tunnel layouts, absence of provider details in Nstance messages, absence of lifecycle state files, forced sleep, NAT dependencies, signing-key/seed create-or-adopt after interrupted applies, signing-key reads only during seed creation, lost registration responses, and bootstrap Secret deletion after durable certificate storage.
