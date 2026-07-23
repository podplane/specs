# Implementation Plan: Load Balancing, Managed NAT, and Scale to Zero
## Delivery constraints
- Roll out AWS NLB with Cloud Managed NAT first, then Google Cloud parity, Podplane Managed NAT on both providers, and finally Cloudflare Tunnel exposure.
- Put provider behavior behind shared Nstance and Podplane contracts; do not duplicate orchestration in provider-specific implementations.
- Keep `nstance-dev/nstance/deploy/tf` as the Terraform source of truth; `terraform-aws-nstance` and `terraform-google-nstance` remain generated release mirrors.
- Release the Nstance changes as 2.0.0. Breaking configuration, protobuf, and storage contracts are allowed; do not implement Nstance 1.x compatibility or mixed-version operation.
## Cross-repository delivery order
1. Establish Nstance APIs, durable state, validation, and deterministic test fakes.
2. Add vmconfig `nst` and `nat` kinds plus node telemetry support.
3. Implement Nstance provider routing, sleep/wake, NAT dependencies, and the operator policy.
4. Package `nstance-operator` and its dependencies in Components.
5. Add Podplane config, Terraform generation, seed values, and CLI commands.
6. Publish Components, regenerate/audit snapshots with seedgen, and publish the versioned seeds.
7. Publish Nstance Terraform mirrors and pin released Nstance, vmconfig, Components, and seed artifacts.
8. Run provider end-to-end tests with forced sleep before enabling automatic sleep.
## Repository: `nstance-dev/nstance`
### Phase 1: Freeze shared configuration and API contracts
- Replace the current coarse load-balancer configuration with the provider-specific `load_balancers` shapes from LB.md:
  - AWS target-group entries with ARN, listener port, target port, and proxy port.
  - Google Cloud `GCE_VM_IP` NEG names plus forwarding-rule IP/port metadata.
  - Provider-neutral tunnel listeners with target and proxy ports.
- Add the top-level tenant `nat` map from NAT.md and validate:
  - at most one dedicated NAT group per tenant;
  - at least one of dedicated or small-cluster NAT when the tenant entry exists;
  - no fixed `size` on a derived NAT group;
  - initial-subnet reservation and cross-tenant subnet exclusivity;
  - identity-pool, grace-period, vertical-scaling, and threshold settings.
- Define shared proxy structs in a public/internal-neutral package used by both `nstance-server` and the new `nstance-proxy` binary. Include only static listener identity, tenant, groups, target port, proxy port, and optional destination IP.
- Extend the operator gRPC service with idempotent `SleepTenant` and `WakeTenant` operations and status needed for all-shard compensation. Make `if_not_busy`, optional `wake_at`, listener-scoped wake, and already-awake/asleep results explicit.
- Have nstance-operator read bootstrap trust from a Kubernetes ConfigMap and the one-time registration nonce from a dedicated Secret in the cluster-specific Netsy seed.
- Treat the registration signing-key storage format and nonce claims/signature as a tested cross-repository bootstrap contract. Keep Nstance's lazy create-if-absent key generation for non-Podplane deployments; the Podplane Terraform provider may create or read the same key without importing Nstance's `internal` packages.
- Make operator registration retry-safe for the same persisted public key and nonce so a lost response cannot consume the nonce without allowing the operator to recover its certificate.
- After durably storing and verifying its operator certificate, have nstance-operator delete only the nonce bootstrap Secret. It must not delete the CA configuration or its managed key/certificate Secret.
- Extend agent health reports with:
  - `ebpf_counters` keyed by port and `ebpf_error`;
  - NAT throughput, packet rate, conntrack count/limit, CPU, and packet-drop metrics;
  - forwarding readiness needed before route cutover.
- Regenerate protobuf code and update server, agent, operator, and tests atomically for the 2.0.0 contract; do not retain legacy fields or decoders solely for 1.x clients/configuration.
**Exit gate:** configuration validation rejects every invalid collision/topology listed in the three specs, and all Nstance consumers use the new generated protobuf and configuration contracts.
### Phase 2: Add durable tenant sleep state and transition serialization
- Add `shard/<shard>/tenants.jsonc` storage models and ETag/generation compare-and-swap helpers.
- Implement idempotent sleep-entry add/update and removal without writing desired group sizes into tenant state.
- Add a tenant-scoped transition lock that serializes:
  - normal and forced sleep;
  - network and timer wake;
  - proxy payload arrival during sleep;
  - provider target cutovers.
- Resume incomplete reconciliation from durable state after process restart or shard-leader replacement.
- Add per-shard wake timers from `wake_at`; concurrent timer and network wakes must converge through the same CAS operation.
- Ensure effective group size is zero only while the sleep entry exists, while preserving administrator/operator desired sizes in the existing group state.
- Keep tenant state shard-local. Add only the AWS cross-zone disable barrier to cluster-leader coordination.
**Exit gate:** storage tests cover CAS conflicts, restart/failover, simultaneous wake triggers, sleep updates, and removal of empty tenant keys; reconciliation tests prove desired sizes survive sleep.
### Phase 3: Implement listener derivation and `nstance-proxy`
- Derive listener-to-tenant/group mappings at config load and reload time. Reject cross-tenant references, zero-sized wake targets, invalid ports, provider-specific collisions, and server-port collisions.
- Atomically write `/run/nstance/nstance-proxy.json` with root ownership and proxy-readable permissions. Rewrite it only for static config changes, not membership changes.
- Add the `nstance-proxy` binary:
  - bind configured ports and dispatch Google Cloud shared ports by accepted destination IP;
  - expose health checks that never call `WakeTenant`;
  - hold client connections for a bounded configurable timeout;
  - call only listener-scoped `WakeTenant` over the root-owned Unix socket;
  - forward to the returned private `IP:target_port` and let established connections drain after direct routing is restored.
- Add server-side Unix-socket authorization and return an upstream only after agent health and target-port readiness are both satisfied.
- Write generic tunnel-process desired state under `/run/nstance/tunnels/<load-balancer-key>.active`; keep tunnel-provider commands out of Nstance.
- Implement bounded secret-cache miss coalescing and atomic delivery of `proxy.files` to the local receive directory and existing agent file streams.
**Exit gate:** proxy integration tests cover connection holding, timeout, listener isolation, destination-IP dispatch, zero-sized group exclusion, partial upstream health, restart, and concurrent wake calls.
### Phase 4: Strengthen provider load-balancer adapters and cutover state machines
- Replace register/deregister methods that return on API acceptance with operations that expose and wait for provider lifecycle states: registered, healthy/routable, draining, and fully deregistered.
- Keep provider transition progress in reconciled state so leadership changes safely continue or roll back a cutover.
- Implement the common sleep sequence:
  1. perform any provider prerequisite barrier, including enabling and confirming AWS target-group cross-zone balancing;
  2. register proxy;
  3. wait for provider health/routability;
  4. start production draining;
  5. wait for a fresh agent report and perform the final busy check;
  6. restore production on activity/error, otherwise finish withdrawal and commit sleep.
- Implement the common wake sequence:
  1. restore desired infrastructure;
  2. wait for upstream and production provider health;
  3. withdraw proxy with draining;
  4. leave proxy installed and continue reconciliation if production readiness times out.
- Never permit the intended registration sets to become empty, and never interpret provider fail-open routing as readiness.
- AWS adapter:
  - register instance targets with per-listener port overrides;
  - wait for target health and deregistration completion;
  - enable target-group cross-zone balancing before sleep;
  - let the cluster leader disable it only after all shards have restored production and removed proxies, serialized against new sleeps.
- Google Cloud adapter:
  - inspect configured NEGs at startup/reload and validate `GCE_VM_IP`, zone, and unique subnet mapping;
  - select the NEG by the instance interface's subnet;
  - add/remove VM-IP endpoints and wait for backend health/draining;
  - support the Nstance-server subnet endpoint during sleep.
- Tunnel adapter/state machine:
  - while going to sleep, wait for the `nst` wake tunnel process to be active before terminating control-plane VMs;
  - while waking, keep the wake tunnel process active until a control-plane production tunnel process and the requested local upstream are ready;
  - permit production/wake tunnel-process overlap only during transitions and require the wake process to be absent in steady-state awake operation;
  - treat tunnel-process readiness and withdrawal as reconciled state that resumes after leader replacement.
**Exit gate:** deterministic provider/tunnel-process fake tests cover healthy cutover, every timeout/rollback point, AWS cross-zone confirmation before draining, tunnel-process ordering, fail-open, shard and cluster leader replacement, and no-empty-route invariants. Live cloud testing is deferred to the final cross-repository matrix after Terraform and vmconfig artifacts exist.
### Phase 5: Implement placement-aware Podplane Managed NAT
- Change subnet placement to fill sorted `/26` node subnets to 90% of AWS-conservative capacity, then round-robin across available slots.
- Model dedicated NAT members as derived per-populated-subnet instances rather than a fixed-size group. Exclude them from MachinePool import and replica distribution.
- Add stable identity allocation state:
  - AWS ENI plus EIP identity;
  - Google Cloud reserved internal plus regional external IPv4 identity;
  - claim/release semantics and explicit exhaustion conditions.
- Add provider route APIs and ownership guards:
  - AWS subnet route-table default route to the selected forwarding ENI;
  - Google Cloud subnet-specific node tags and tagged default routes to a forwarding VM;
  - mutate a route only while Podplane Managed NAT is configured, the resource is scoped to this cluster, and its current next hop is still Podplane-managed, so stale reconciliation cannot overwrite Terraform's Cloud Managed NAT cutover.
- Enforce node creation ordering: select subnet, establish healthy NAT, install route, then create the node.
- Implement small-cluster NAT on the elected shard leader, including activation-before-route-switch and leadership failover.
- Implement last-node grace handling, NAT-first wake, node-first sleep, and cleanup when changing back to Cloud Managed NAT.
- Implement both mode transitions explicitly:
  - Cloud Managed to Podplane Managed: disable Cloud Managed NAT/update Nstance configuration, retain provider and object-storage reachability, then let Nstance establish healthy next hops and install Podplane Managed routes; document the temporary general IPv4 egress outage.
  - Podplane Managed to Cloud Managed: create and validate Cloud Managed NAT, switch node-subnet routing, then let Nstance disable small-cluster forwarding, terminate dedicated NAT instances, and release identities.
- Keep IPv6 untranslated and preserve existing IPv6 routing through both mode changes.
- Implement vertical A/B replacement for dedicated NAT instances using health, ordered instance-type ladders, and a second stable identity. Scale up when any threshold remains exceeded for the configured two-to-five-minute window; scale down only when all metrics remain below their thresholds for the configured 20-to-30-minute window; enforce the configurable ten-minute cooldown. Never do disruptive same-identity replacement on pool exhaustion.
- When identities are exhausted, prefer already-populated subnets with remaining capacity before blocking node scale-up.
**Exit gate:** tests cover placement capacity, concurrent creates, current-next-hop route ownership, tenant conflicts, leader changes, both ordered mode transitions, retained provider/object-storage access, unchanged IPv6 routing, mixed metric windows, alternate-subnet placement under identity exhaustion, vertical replacement/cooldown, NAT-first wake, and node-first sleep on both providers.
### Phase 6: Implement Kubernetes sleep policy in `nstance-operator`
- Add operator configuration for disabled-by-default automatic sleep with defaults from ZERO.md: one remaining node, 30-minute inactivity, ten-minute CronJob look-ahead, and two-minute wake lead.
- Watch CAPI Cluster/Machine/MachinePool and Nstance resources, Nodes, Jobs, CronJobs, and relevant Pods.
- Reconcile `nstance.dev/sleep-request` annotations for normal and `force:<timestamp>` requests.
- Evaluate normal-sleep eligibility exactly as specified, including pending scale operations, nonterminal/deleting Jobs, suspended Jobs with active/terminating Pods, CronJob UID ownership, time zones, and independent listener activity windows.
- Calculate the earliest CronJob execution and place `wake_at` on one wake-capable shard.
- Process all shards with the coordinating node's shard last.
- Keep the sleep-request annotation until every shard has durably accepted a successful request, then remove it; persist/retry successful completion across operator restart just as for compensation.
- Persist all-shard compensation progress so restart resumes `WakeTenant` calls until every partially slept shard is awake; only then emit rejection and remove the annotation.
- After any local wake, restore other shards required by MachinePool placement and recompute the next CronJob deadline.
- Add RBAC markers/manifests for CAPI Cluster patching and all watched workload resources.
**Exit gate:** envtest/operator integration tests cover eligibility, forced sleep, shard ordering, annotation retention until durable all-shard success, partial failure compensation, operator restart during success or compensation, CronJob semantics, and post-wake restoration.
### Phase 7: Update canonical AWS and Google Terraform modules
- Change shared module input/output contracts under `deploy/tf/common` first, then update both provider implementations.
- AWS network/account/shard modules:
  - one regional target group per exposed listener, shared across zones/shards;
  - TCP health checks, draining, cross-zone disabled initially, and target-group metadata including listener/target/proxy ports;
  - one route table per node subnet;
  - Cloud Managed versus Podplane Managed NAT switch;
  - pre-provisioned ENI/EIP NAT identities;
  - public service subnet placement and public IPv4 for `nst` under Podplane Managed NAT;
  - scoped ELB, route, ENI, EC2, S3 gateway endpoint, and EC2 interface endpoint permissions;
  - security groups that expose only configured NLB health/proxy paths while keeping Nstance registration, operator, agent, election, and server-health endpoints private.
- Google Cloud network/account/shard modules:
  - regional external passthrough Network Load Balancers;
  - one `GCE_VM_IP` NEG per logical membership set, zone, and eligible production/Nstance subnet;
  - backend services, forwarding rules, reserved frontend IPs, health checks, and firewall rules;
  - Cloud NAT restricted to node subnet ranges in Cloud Managed mode;
  - Podplane Managed tagged routes and pre-provisioned internal/external NAT identities;
  - `can_ip_forward`, external IPv4 for `nst` in Podplane Managed mode, Private Google Access, and scoped NEG/route permissions;
  - firewall rules that expose only configured load-balancer health/proxy paths while keeping every nstance-server API private.
- Pass only static resource identity/configuration to nstance-server; do not put dynamic target membership in Terraform state.
- Add `nst` vmconfig userdata inputs, optional proxy/tunnel settings, shard endpoints, and NAT identity-pool references. Do not place an operator nonce in VM userdata, Terraform outputs, or state.
- Update examples and module contract tests for Cloud Managed NAT, Podplane Managed NAT, NLBs, tunnels, and mode changes that preserve node subnets.
**Exit gate:** `tofu validate` and provider-specific fixture plans pass; plans show no node-subnet replacement when switching NAT modes, no duplicated AWS target groups or same-zone Google NEGs, and no public path to nstance-server APIs. Network tests prove public traffic can reach configured proxy/health listeners but cannot invoke `WakeTenant`, which remains Unix-socket-only.
### Phase 8: Publish Nstance 2.0.0
- Document breaking configuration/protobuf/storage changes, IAM additions, disruptive NAT route switches, Cloudflare trust boundaries, deployment requirements, and rollback boundaries; do not provide 1.x migration shims.
- Release server, agent, operator, proxy, Helm chart, and Terraform source changes together as one pinned 2.0.0 release set.
- State explicitly that mixed Nstance 1.x/2.x deployments and 1.x configuration or storage formats are unsupported.
- Trigger the established Terraform sync workflow from this canonical repository and record the exact source commit in both generated mirror commits.
**Exit gate:** all Nstance 2.0.0 artifacts are published, generated Terraform mirror commits reference the release source commit, and downstream repositories can pin the complete immutable version set.
## Repository: `podplane/vmconfig`
### Phase 1: Introduce reusable service configuration and new VM kinds
- Extract only the shared install/configure/restart machinery into `templates/common`, then build the VM kinds as `common + knd`, `common + knd + knc`, `common + nst`, and `common + nat`, in overlay order.
- Keep `nst` and `nat` independent. Duplicate their small forwarding/SNAT configuration rather than adding a shared feature overlay; extract it later only if maintaining both copies becomes onerous.
- Add `nst` kind for Nstance-server VMs and `nat` kind for dedicated NAT VMs, each with manifests for Debian 13 on amd64 and arm64.
- Add manifest dependencies for `nstance-server` and `nstance-proxy`. Tag `bpftool` and required eBPF runtime libraries with a traffic-accounting feature for `knd`/`knc`, and tag the initial tunnel client (`cloudflared`) with a Cloudflare tunnel feature for `knc`/`nst`; untagged dependencies remain baseline.
- Extend the shared manifest `ItemFilter`, Podplane userdata renderer, and Terraform `podplane_userdata` data source with selected install features. Use the same filter for downloads, checksum metadata, and installation so an unselected dependency is neither fetched nor expected in `/opt/podplane/artifacts`.
- Write the selected immutable install-feature set before invoking `install.sh`; make `install.sh` filter generated dependency arrays from the pinned full manifest. Changing tunnel or traffic-accounting features therefore changes userdata and rotates/rebuilds affected VMs rather than mutating an already-installed machine in place.
- Preserve strict artifact verification, reproducible packaging, standard Fluent Bit, and the existing receive/reconfigure workflow.
**Exit gate:** package and install tests cover all four kinds (`knd`, `knc`, `nst`, and `nat`) on both architectures without regressing existing manifests. Disabled-feature fixtures fetch or install neither `cloudflared` nor `bpftool`; enabled fixtures download, verify, and install exactly the pinned artifacts. Local/offline dependency prefetch accepts or infers the same feature set.
### Phase 2: Add traffic accounting to Kubernetes nodes
- Build and package the vmconfig-owned eBPF ELF for active TCP connections.
- Select the traffic-accounting install feature only for `knd`/`knc` groups that can participate in guarded normal sleep and are referenced by production listeners.
- Derive the monitored ports from each group's production listeners: ingress ports only, kube-apiserver port only, their union for a shared group, or no accounting for a group with neither role.
- Add a root-owned oneshot service that loads the ELF with `bpftool`, auto-attaches tracepoint programs, and pins links and maps under `/sys/fs/bpf/nstance/counters`.
- Grant nstance-agent read access only to link metadata needed for validation and the `active_connections` map.
- Configure the loaded program to ignore ports outside the derived set. When accounting is disabled, do not download or install `bpftool`, enable the loader, attach programs, create pins, set `NSTANCE_EBPF_COUNTERS_PATH`, or require counters from the agent.
- Ensure reloads update configured ports without leaving stale links/maps and that pin loss is visible as `ebpf_error` rather than silently reporting zero.
**Exit gate:** VM tests cover ingress-only, kube-apiserver-only, combined, and disabled configurations; generate traffic on configured and ignored ports; and verify zero/nonzero counts, pinned-link loss, permissions, reboot persistence, and complete runtime absence when disabled.
### Phase 3: Implement `nst` proxy, local files, and tunnel services
- Configure `nstance-server` with the existing vmconfig user-data and watch machinery rather than the Nstance module's generic server userdata.
- Enable `nstance-proxy` only when sleep support is configured. Keep it running while awake and asleep; only Nstance changes external routing.
- Create runtime users, strict ownership, the root-owned Unix socket, proxy-readable generated config, and systemd reload behavior after atomic replacement.
- Install files received through Nstance's local fixed receive directory using strict modes and restart only affected services.
- Add provider-neutral tunnel service templates driven by generic tunnel-state files.
- Add Cloudflare-specific `knc` and `nst` configuration:
  - one production tunnel process per control-plane VM for kube-apiserver plus optional ingress;
  - one wake tunnel process per eligible Nstance server;
  - HTTPS origin verification using the cluster/ingress CA or public roots;
  - no TLS verification disablement;
  - no secrets in arguments, logs, generated Terraform values, or world-readable files.
**Exit gate:** local VM tests prove tunnel state switches between production and wake paths, secret rotation restarts only the affected tunnel service, and both API and ingress origins retain TLS verification.
### Phase 4: Implement NAT host behavior
- In the `nst` overlay, configure optional IP forwarding and source NAT for small-cluster NAT while leaving it inactive on non-leaders.
- Independently in the `nat` overlay, configure forwarding, source NAT, conntrack limits, Nstance agent reporting, packet-drop/throughput/packet-rate/CPU metrics, and readiness checks.
- Ensure AWS source/destination-check and Google `can_ip_forward` remain Terraform/provider concerns, not shell-script cloud API calls.
- Make forwarding activation/deactivation idempotent and safe across reboot and Nstance leadership transitions.
**Exit gate:** network-namespace or VM integration tests verify forwarding, SNAT, readiness, metrics, and deactivation for `nst` and `nat`.
### Phase 5: Publish manifests
- Publish all new kind/architecture packages and manifests.
- Pin the Nstance 2.0.0 release in each new manifest.
- Leave historical immutable manifests unchanged, but do not support combining them with the 2.0.0 release set.
**Exit gate:** immutable 2.0.0 manifest/package versions are published for every required kind and architecture, verify successfully from a clean cache, and are recorded in the cross-repository release set.
## Repository: `podplane/components`
### Phase 1: Add Nstance component charts and wire dependencies
- Add `nstance-operator-crds` by vendoring the Nstance CRDs through the repository's existing CRD update process.
- Add an opinionated `nstance-operator` chart based on the upstream Nstance Helm chart, following Podplane conventions for namespaces, image mirroring, security context, probes, leader election, and CRD skipping.
- Configure dependencies in `platform-components`:
  - `cert-manager` depends on `cert-manager-crds`;
  - `cluster-api` depends on `cluster-api-crds` and `cert-manager`;
  - `nstance-operator` depends on `nstance-operator-crds` and `cluster-api`.
- Do not add a Podplane Secrets, Secrets Store CSI, or cloud CSI-provider dependency for Nstance registration. The cluster-specific seed supplies the 30-minute nonce as an ordinary Kubernetes Secret.
- Mark the Nstance/Cluster API provider-overlay entries `core: true` but `enabled: false` in chart defaults. Podplane enables them only for AWS/Google `recommended` clusters; the existing recommended cert-manager installation satisfies both CAPI and Nstance webhook certificate requirements.
- Add nstance-operator images and both architectures to `manifests/components.json` with provider metadata for AWS and Google Cloud; do not mark them as generic recommended addons.
**Exit gate:** Helm lint/render tests show the complete dependency graph and render provider-enabled Nstance resources with CRDs handled only by the CRD releases.
### Phase 2: Add seed-contained operator bootstrap and RBAC
- Expose values for cluster ID, tenant, shard registration/operator endpoints, public CA ConfigMap, nonce Secret name/key, and automatic-sleep policy.
- Reference a dedicated bootstrap Secret that the Podplane Terraform provider inserts into the cluster-specific Netsy seed. Do not render the nonce into HelmRelease values or a chart-owned Secret.
- Give the nonce a 30-minute TTL, make it single-use, tenant- and cluster-scoped, and restrict it to operator registration. Once an operator certificate exists, nstance-operator must not attempt registration again.
- Delete the nonce Secret only after the certificate and matching private key have been durably stored and verified. Historical Netsy revisions, bucket versions, or backups may retain the consumed nonce; that is acceptable because Nstance has invalidated it.
- Include CAPI Cluster get/list/watch/patch/update permissions, Job/CronJob/Pod/Node reads, Event creation, Lease leader election, and Nstance CRD permissions.
- Keep the operator highly available during normal operation and schedulable on the final supported one-node topology.
- Ensure platform RBAC initially limits sleep requests to `podplane:admins`, which is bound to `cluster-admin`; the CLI's access review is an additional check, not the authorization boundary.
**Exit gate:** RBAC tests demonstrate the operator can reconcile sleep requests but cannot mutate unrelated cluster-scoped resources, and non-admin users cannot patch the sleep annotation under the shipped policy. Bootstrap tests prove the operator registers from the seeded Secret, deletes that Secret only after durable certificate storage, retries safely after a lost registration response, and reuses its managed certificate after the nonce expires.
### Phase 3: Preserve Traefik host exposure
- Keep Traefik as a DaemonSet with host ports 80 and 443 in both NLB and tunnel modes; tunnel configuration simply omits a port-80 origin route.
- Add explicit scheduling/toleration tests proving Traefik remains present on control-plane nodes when tunnel ingress is enabled.
- Do not route kube-apiserver through Traefik.
**Exit gate:** rendered manifests and a cluster smoke test verify direct local access to Traefik on control-plane port 443 and direct kube-apiserver port 6443.
### Phase 4: Correct component-set semantics and documentation
- Document `nstance-operator` as a protected AWS/Google provider overlay for `recommended` clusters, not part of the generic recommended addon set.
- Remove or correct the existing stale documentation that describes Nstance as a recommended addon before the charts exist.
- Keep `minimal`, `none`, and local/providerless seeds free of the provider Nstance overlay unless explicitly configured for a Nstance development environment.
- Publish the Components charts and component image manifest. Seed generation and publication belong to `podplane/seedgen` and `podplane/seeds`, respectively.
**Exit gate:** an immutable Components release contains the charts, provider metadata, and complete multi-architecture image manifest required to regenerate both seeds.
## Repository: `podplane/podplane`
### Phase 1: Add user-facing configuration and schemas
- Replace the current provider-only load-balancer projection with an exposure model that independently selects NLB, tunnel, or custom behavior for Kubernetes API and ingress.
- Add provider-specific NLB settings and provider-neutral tunnel settings while keeping application domain configuration separate.
- Add NAT mode (`cloud-managed` or `podplane-managed`), maximum dedicated NAT identities per zone, small-cluster NAT settings, NAT group template/scaling policy, and stable identity-pool settings.
- Add disabled-by-default sleep policy with activity windows, CronJob look-ahead, wake lead, proxy timeout, and readiness timeout.
- Validate:
  - scale-to-zero requires an AWS or Google Cloud cluster using the `recommended` seed;
  - indefinitely sleeping configurations have either a proxy-reachable exposure method or a guaranteed `wake_at` policy;
  - `nst` placement is public/direct when Podplane Managed NAT is selected and private by default for Cloud Managed NAT;
  - NLB/tunnel listener and proxy-port rules from LB.md;
  - NAT group/subnet/tenant constraints from NAT.md;
  - Google Cloud's equal passthrough ports and AWS proxy-port uniqueness.
- Update JSON Schema, config examples, TUI defaults, config docs, and focused validation tests together.
**Exit gate:** all supported AWS/Google NLB, tunnel, NAT, and sleep combinations round-trip through config parsing and schema validation; invalid no-wake and collision cases fail before Terraform generation.
### Phase 2: Generate infrastructure and Nstance/vmconfig inputs
- Generalize Terraform generation to AWS and Google Cloud module sources with equivalent topology inputs.
- Emit Cloud Managed versus Podplane Managed NAT resources without changing node-subnet identity.
- Generate one `/26` per node placement subnet and the Nstance `nat` configuration, dedicated NAT template, identity pool, and `nat` vmconfig manifest references.
- Generate provider-specific load-balancer metadata in the new Nstance config shape, including all production and Nstance-server subnets required for Google NEGs.
- Select and pin `knc`, `knd`, `nst`, and `nat` vmconfig manifests by architecture, and pass immutable install features to each userdata data source. Set the Cloudflare tunnel feature only for affected `knc`/`nst` VMs and traffic accounting only for affected `knc`/`knd` groups participating in guarded normal sleep.
- Replace generic Nstance-server userdata with the `nst` vmconfig kind and pass optional proxy/tunnel configuration through generated environment/files.
- Generate tunnel resources and secret delivery for Cloudflare without serializing credentials into normal Terraform values, process arguments, logs, or world-readable files.
- Configure the initial Cloudflare implementation as public HTTPS applications, with a hostname-scoped HTTP-to-HTTPS edge redirect and no port-80 tunnel origin.
- Generate stable API/ingress endpoint outputs:
  - NLB API defaults to external port 6443 and embeds the Kubernetes CA;
  - Cloudflare API uses external port 443 and public CA roots;
  - kube-apiserver remains independent of Traefik.
- Pass only non-sensitive operator bootstrap configuration—cluster ID, tenant, shard addresses, CA ConfigMap name, and nonce Secret name/key—into component values. The state-safe provider operation inserts the actual one-time nonce directly into the cluster-specific Netsy seed.
**Exit gate:** golden Terraform tests cover AWS and Google Cloud for NLB, Cloudflare tunnel, both NAT modes, and sleep on/off; sensitive values are absent from generated non-sensitive files.
### Phase 3: Inject operator bootstrap material into the cluster-specific seed
- Run this phase only when the AWS/Google `recommended` provider overlay enables nstance-operator; do not create a signing key, nonce, or operator bootstrap objects for `minimal`, `none`, local, or providerless clusters.
- Generate a state-safe create-if-absent signing-key step after Nstance storage and encryption are available. It adopts a key created by Nstance or an interrupted apply and never rotates it automatically.
- Make the existing cluster-specific Netsy seed resource depend on that step and the Nstance public CA. Only its Create operation reads and decrypts the signing key, mints the cluster/tenant-scoped 30-minute single-use nonce, inserts the CA ConfigMap and nonce Secret, and uploads the completed seed.
- If a completed seed already exists, adopt it without reading the signing key; normal refresh reads only non-secret seed metadata. A retry after key creation reuses that key, and a retry after seed upload adopts the seed.
- Insert the public CA ConfigMap and dedicated nonce Secret directly into the cluster-specific seed. The generic published minimal/recommended seeds remain credential-free.
- Keep the signing key and nonce out of generated HCL, plans, outputs, provider state, diagnostics, logs, and temporary files. Their handling remains in memory; the nonce's deliberate presence in the encrypted cluster-specific Netsy seed and retained object versions is the bootstrap transport.
- If the nonce expires after the cluster becomes live, document minting a replacement nonce and using `kubectl` to replace the bootstrap Secret. If the cluster never becomes live, destroy and recreate it rather than attempting to patch or re-upload its initial seed.
**Exit gate:** generated Terraform tests cover resource ordering, create/adopt paths, the 30-minute TTL, cluster/tenant scoping, and absence of plaintext bootstrap material outside the cluster-specific seed. Cluster tests prove seeded-Secret registration, post-registration Secret deletion, safe retry after a lost response, certificate reuse after nonce expiry, and manual `kubectl` replacement on a live unregistered cluster.
### Phase 4: Enable the provider overlay for Recommended
- Extend provider component selection so AWS and Google Cloud clusters using `recommended` additionally enable:
  - `cert-manager-crds` and `cert-manager` if not already enabled by the selected seed;
  - `cluster-api-crds` and `cluster-api`;
  - `nstance-operator-crds` and `nstance-operator`;
  - existing provider-specific storage components.
- Do not enable Podplane Secrets, Secrets Store CSI, or a cloud CSI secrets provider solely for Nstance bootstrap.
- Apply the Nstance overlay only after loading `recommended` and only for AWS/Google providers.
- Leave `recommendedAddons` unchanged with respect to Nstance.
- Leave `minimal` and `none` without the Nstance operator, CAPI, or operator-bootstrap resources, and leave local/providerless recommended seeds without the overlay.
- Generate component values for shard endpoints, bootstrap ConfigMap/Secret references, and sleep policy without conflating these with user-selectable recommended addons.
- Add tests for AWS/Google versus local/providerless seeds and for recommended/minimal/none behavior.
**Exit gate:** AWS/Google cluster-specific snapshots derived from `recommended` contain cert-manager, CAPI, the Nstance operator, and bootstrap objects; generic recommended addon enumeration does not include Nstance; `minimal`, `none`, and local/providerless snapshots omit the overlay. Running-operator verification occurs after the cluster boots from the uploaded seed in the final end-to-end matrix.
### Phase 5: Add `podplane cluster sleep`
- Add `podplane cluster sleep` and `podplane cluster sleep --force` under the existing cluster-config command conventions.
- Resolve kubeconfig/context through the same cluster command path used by other cluster operations.
- Issue a `SelfSubjectAccessReview` for patching the target CAPI Cluster before mutation.
- Normal mode writes `nstance.dev/sleep-request=<timestamp>` and returns immediately.
- Forced mode displays an explicit interruption warning and confirmation, then writes `force:<timestamp>`.
- Do not add a wake command; document that API or ingress traffic wakes the cluster.
- Report annotation/authorization failures clearly without waiting for the cluster to sleep.
**Exit gate:** command tests cover access denial, normal and forced annotations, confirmation, noninteractive behavior, config/context selection, and immediate disconnect.
### Phase 6: Provider rollout and Nstance 2.0 deployment
- First enable forced sleep on a single-zone AWS test cluster, then multi-zone AWS, then equivalent Google Cloud topologies.
- Enable Cloudflare API wake before Cloudflare ingress wake so direct administrative recovery is proven first.
- Keep automatic sleep off unless the pinned operator, proxy, NAT, and exposure release set passes the provider matrix.
- Require coordinated Nstance 2.0 deployment rather than mixed 1.x/2.x operation; document cluster recreation where an existing deployment cannot be cut over atomically.
- Define deployment behavior:
  - add provider-default components before enabling sleep policy;
  - rotate Nstance-server VMs when adopting `nst` or changing NAT placement;
  - never rotate Kubernetes nodes solely to change NAT mode;
  - preserve old direct routing until the proxy path has passed readiness.
- Update user docs for cost/availability tradeoffs, Cloudflare trust, NAT connection resets, force-sleep risk, and rollback.
**Exit gate:** forced sleep passes the staged AWS and Google Cloud rollout, Cloudflare API wake provides administrative recovery, deployment plans preserve production routing and Kubernetes nodes where supported, and automatic sleep remains gated off pending the final matrix.
## Repository: `podplane/terraform-provider-podplane`
### Phase 1: Add feature-aware userdata rendering
- Add an optional set of immutable install features to `podplane_userdata` and pass it into the shared Podplane userdata renderer.
- Include selected features in deterministic rendered content and state so changing tunnel or traffic-accounting requirements changes the VM configuration hash and rotates affected VMs.
- Filter manifest downloads/checksums with the shared `ItemFilter`; reject unknown features and dependencies whose feature/kind metadata cannot match the selected VM.
**Exit gate:** data-source tests prove identical manifests include `cloudflared` and `bpftool` only when their respective features are selected, remain deterministic, and preserve existing provider filters.
### Phase 2: Add state-safe Nstance operator seed bootstrap
- Add a state-safe signing-key resource that conditionally creates Nstance's registration key in its encrypted secrets store or adopts the existing key after an interrupted apply/Nstance lazy initialization. State contains only backend identity, version, and hash; updates never rotate the key automatically.
- Make the existing Netsy seed resource depend on the signing-key resource and boundedly wait for Nstance's public `ca.crt`.
- In the seed resource's Create operation only, read/decrypt the signing key into memory, mint the 30-minute tenant- and cluster-scoped single-use nonce using the provider's implementation of the tested Nstance bootstrap contract, inject the nonce Secret and CA ConfigMap, and upload the completed cluster-specific seed without intermediate plaintext files.
- Before reading the key, detect and adopt an already-completed seed; normal Read/refresh inspects only non-secret metadata. This recovers both a write-before-state signing-key interruption and an upload-before-state seed interruption.
- Keep the signing key and nonce out of schema values, plans, state, outputs, command arguments, diagnostics, logs, and temporary files; discard in-memory plaintext after Create.
- Make retries idempotent using Nstance's durable operator registration record and the persisted operator public key: tolerate delayed bootstrap, apply interruption, a lost registration response, and leader replacement.
- Do not add automatic initialized-cluster repair to the Terraform provider. Document the simple recovery contract: use the canonical Nstance admin nonce command and `kubectl` to replace the bootstrap Secret when the cluster is live; destroy and recreate a cluster that never became live. Never re-upload an initial seed over live Netsy state.
**Exit gate:** provider integration tests cover key create/adopt races, seed create/adopt paths, proof that key reads occur only when seed creation is required, the 30-minute nonce TTL, delayed CA bootstrap, nonce use, lost registration responses, apply interruption, leader change, and the shared Nstance 2.0 bootstrap format. Tests permit the nonce only inside the encrypted cluster-specific seed; live-cluster expiry recovery remains a documented manual procedure and cluster test.
## Repository: `podplane/seedgen`
seedgen turns a running reference cluster into audited, provider-neutral minimal and recommended Netsy snapshots. Provider-required component enablement is applied later by Podplane while it builds the cluster-specific snapshot; seedgen does not own that selection logic.
### Phase 1: Confirm provider-neutral filtering
- Verify that existing include/exclude and expected-record rules retain the updated `platform-components` release while excluding per-cluster runtime credentials and identity resources.
- Do not add rendered Nstance operator, Cluster API, CA, nonce, or operator identity records to the generic seed merely because AWS/Google clusters enable them later.
- Keep generic `minimal` free of the Nstance operator, Cluster API, and cert-manager; the Nstance provider overlay applies only to an AWS/Google cluster-specific snapshot derived from `recommended`.
- Change seedgen code only if the new Components release introduces generic platform records that the current profile or credential-exclusion checks cannot represent.
**Exit gate:** focused pipeline tests reject snapshots containing nonce values, generated certificates, or runtime identity Secrets, and confirm the generic minimal/recommended expectations remain provider-neutral.
### Phase 2: Regenerate and audit both snapshots
- Build a reference cluster from the released Components version, export both profiles, and write their record/report trees.
- Run expectation, integrity, revision-contiguity, and component-manifest verification for `minimal.netsy` and `recommended.netsy`.
- Review the record diff to confirm that Nstance/Cluster API workloads are not pre-enabled in either generic snapshot and that local/providerless seed semantics remain unchanged.
**Exit gate:** reproducible snapshot candidates and complete audit reports are ready for `podplane/seeds`, with no sensitive or per-cluster runtime state.
## Repository: `podplane/seeds`
### Phase 1: Update seeds
## Repository: `nstance-dev/terraform-aws-nstance`
### Phase 1: Publish a new version
## Repository: `nstance-dev/terraform-google-nstance`
### Phase 1: Publish a new version
## Repository: `podplane/specs`
### Phase 1: Track contract changes and conformance
- Update LB.md, NAT.md, or ZERO.md only when implementation discovers a genuine contract ambiguity or provider limitation; do not silently diverge in code.
- Record any accepted deviation, breaking contract, deployment requirement, or deferred provider feature before merging the affected implementation.
- Build a requirement-to-test matrix, then exercise AWS and Google Cloud in one/multiple zones, both NAT modes, NLB, forced/automatic sleep; Cloudflare API-only and API-plus-ingress on both providers; shared/separate control-plane and ingress groups; one-to-zero and redundant two-zone sleep; timer/network wake; provider/proxy timeouts; operator restart; and shard/cluster leader replacement.
- Keep automatic sleep disabled by default until both providers pass all routing, NAT, sleep-compensation, packaging, bootstrap, release, and deployment gates from the repository phases above.
