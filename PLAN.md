# Implementation Plan: Load Balancing, Managed NAT, and Scale to Zero

> **STATUS**: In progress

## Delivery constraints
- Roll out AWS NLB with Cloud Managed NAT first, then Google Cloud parity, Podplane Managed NAT on both providers, and finally Cloudflare Tunnel exposure.
- Put provider behavior behind shared Nstance and Podplane contracts; do not duplicate orchestration in provider-specific implementations.
- Podplane exposes Kubernetes API and ingress together: one optional singular `cluster.tunnel` config alongside domains and ACME selects tunnels for both; absence selects provider-default NLBs for both. Tunnel mode generates no NLB resources or registrations. No per-endpoint selectors, custom third mode, or mixed exposure is supported.
- Support only existing user-managed Cloudflare tunnels: `kind`, `id`, and one `credential_secret` reference shared by node and `nst` processes. Users pre-create the credential secret and manage tunnel DNS/edge settings. Resolve the immutable image from VM manifests; do not create tunnels or read credential payloads through Terraform.
- NAT mode is `provider` or `nstance`. Podplane exposes NAT group deployment settings, an optional fixed IPv4 count per NAT service subnet, and only the default tenant's group and instance-type ladder. Nstance retains its richer runtime policy contract and supplies timing and threshold defaults.
- Tunnel ingress reuses kube-apiserver's node certificate provisioning mechanism: one agent-generated ingress keypair per Envoy-capable node, signed by the existing Nstance cluster CA, with every configured apex/wildcard, localhost, loopback IPs, and the node's assigned IPv4/IPv6 addresses in its SANs and node-local SDS delivery. Reject tunnel plus ACME and disable the operator's complete ingress certificate manager in tunnel mode. NLB mode retains operator-owned per-domain fallback/ACME bundles and CSI/SDS delivery. Follow [LB.md](./LB.md#ingress-certificates-and-sds); add no cluster-leader certificate manager, private-key storage resource, renewal scheduler, or CA-rotation machinery for this path.
- Changing NLB/tunnel mode requires infrastructure deletion and redeployment, not an in-place switch. Preserving Netsy data alone is not a complete state-preserving migration. NAT mode remains independent and retains the switching contract in [NAT.md](./NAT.md).
- Podplane has no user-facing `SleepPolicy` and generates no sleep-policy or timing overrides. Supported AWS/Google `recommended` deployments generate wake-capable routing without a sleep-policy flag; manual CLI sleep remains supported and Nstance/operator defaults keep automatic sleep disabled. Nstance and Components retain their own policy configurability.
- Keep `nstance-dev/nstance/deploy/tf` as the OpenTofu/Terraform source of truth; `terraform-aws-nstance` and `terraform-google-nstance` remain generated release mirrors.
- These breaking configuration, protobuf, and storage contracts were introduced in Nstance 2.0.0. Use a compatible released 2.x artifact set; do not implement Nstance 1.x compatibility or mixed-version operation.
## Current delivery boundary and follow-ups
- AWS/Google generation, node-certificate/SDS delivery, and the Terraform provider's registration-key/seed-bootstrap code exist. Their implementation does not establish a published compatible release set or successful VM/cloud deployment; keep the overall status `In progress`.
- The Terraform provider must adopt a Podplane dependency containing `userdata.Options.ServerEnv`; its older dependency currently prevents compilation. Its recommended-cloud bootstrap fixtures also need the required secrets-provider configuration before their recovery tests pass.
- vmconfig's working manifests now pin Nstance 2.2.0 with verified release checksums. Publishing/adopting those manifests, seed regeneration, and real VM/cloud validation remain separate delivery gates, not reasons to reimplement existing code.
- Two design follow-ups remain proposals, not settled implementation requirements: typed infrastructure inputs instead of standard `values_content` wiring, and a metadata-only tunnel-secret preflight using existing permissions. See the provider interface proposal below and [LB.md](./LB.md#proposed-credential-preflight).
## Cross-repository delivery order
1. Establish Nstance APIs, durable state, validation, and deterministic test fakes.
2. Add vmconfig `nst` and `nat` kinds plus node telemetry support.
3. Implement Nstance provider routing, sleep/wake, NAT dependencies, and the operator policy.
4. Package `nstance-operator` and its dependencies in Components.
5. Add Podplane config, OpenTofu/Terraform generation, seed values, and CLI commands.
6. Publish Components, regenerate/audit snapshots with seedgen, and publish the versioned seeds.
7. Publish Nstance OpenTofu/Terraform mirrors and pin released Nstance, vmconfig, Components, and seed artifacts.
8. Run provider end-to-end tests with forced sleep before enabling automatic sleep.
## Repository: `nstance-dev/nstance`
### Phase 1: Freeze shared configuration and API contracts
- [x] Replace the current coarse load-balancer configuration with the provider-specific `load_balancers` shapes from LB.md:
  - [x] AWS target-group entries with ARN, listener port, target port, and proxy port.
  - [x] Google Cloud `GCE_VM_IP` NEG names plus forwarding-rule IP/port metadata.
  - [x] Provider-neutral tunnel listeners with target and proxy ports.
- [x] Use `google` as the Google Cloud infrastructure and load-balancer provider literal and `google-secret-manager` as its Secret Manager provider literal. The former `gcp` and `gcp-secret-manager` literals are unsupported in 2.0.0.
- [x] Add the top-level tenant `nat` map from NAT.md and validate:
  - [x] exactly one NAT group per configured tenant;
  - [x] no fixed `size` on a derived NAT group;
  - [x] cross-tenant subnet exclusivity;
  - [x] tenant-owned optional `public_addresses`, `last_instance_grace_period`, ordered `instance_type_ladder`, CPU/conntrack/packet-drop thresholds, scaling windows, cooldown, and replacement timeout.
- [x] Define shared proxy structs used by the nstance-server control plane and `nstance-server proxy`. Represent static listener identity as the map key; include only tenant, groups, target port, proxy port, and optional frontend IP in each value.
- [x] Extend the operator gRPC contract with idempotent `SleepTenant` and `WakeTenant` result/status semantics needed for all-shard compensation. Make `if_not_busy`, optional `wake_at`, optional listener-scoped wake, optional ready upstream, and already-awake/asleep results explicit.
- [x] Have nstance-operator read bootstrap trust from a Kubernetes ConfigMap and the one-time registration nonce from a dedicated Secret in the cluster-specific Netsy seed.
- [x] Define and test the cross-repository registration nonce claims/signature contract in public `pkg/nonce`, and retain the PKCS#8 PEM Ed25519 signing-key storage contract. Keep Nstance's lazy create-if-absent key generation for non-Podplane deployments; the Podplane Terraform provider may create or read the same key without importing Nstance's `internal` packages.
- [x] Keep that bootstrap integration secrets-backend-aware: AWS defaults to `aws-parameter-store` for direct secret storage and as the object-storage encryption-key provider, while Google Cloud defaults to `google-secret-manager`. Do not assume that an AWS cluster uses Secrets Manager.
- [x] Make operator registration retry-safe by durably and atomically binding the exact nonce JWT to its byte-identical public-key PEM on first use. Accept that pair after a lost response and reject a different PEM, including across leader replacement.
- [x] After durably storing and verifying its operator certificate, have nstance-operator delete only the nonce bootstrap Secret. It must not delete the CA configuration or its managed key/certificate Secret.
- [x] Replace agent-side metric history with one `metrics` observation per health report; `NSTANCE_REPORT_INTERVAL` controls report transmission and metric collection occurs while constructing each report.
- [x] Extend the agent health-report contract with:
  - [x] current CPU, per-core CPU, and memory utilization;
  - [x] optional `network_interface` cumulative byte, packet, and drop counters selected by `NSTANCE_METRICS_INTERFACE`, plus `network_interface_error`;
  - [x] conntrack count/limit and explicit collection errors when interface metrics are enabled;
  - [x] `ebpf_counters` keyed by port and `ebpf_error`;
  - [x] raw cumulative network-interface counters suitable for server-side rate and window derivation.
- [x] Define NAT-instance readiness as agent registration followed by a successful health report; do not add a separate forwarding-readiness metric.
- [x] Normalize every protobuf enum to a prefixed zero-valued `UNSPECIFIED` member and consistently prefixed operational values.
- [x] Move the Go puidv7 dependency to `github.com/puidv7/puidv7-go`.
- [x] Regenerate protobuf code and update server, agent, operator, and tests atomically for the 2.0.0 contract; do not retain legacy fields or decoders solely for 1.x clients/configuration.
**Exit gate:** configuration validation rejects every invalid collision/topology listed in the three specs, and all Nstance consumers use the new generated protobuf and configuration contracts.
### Phase 2: Add durable tenant sleep state and transition serialization
- [x] Add `shard/<shard>/tenants.jsonc` storage models and ETag/generation compare-and-swap helpers.
- [x] Implement idempotent sleep-entry add/update and removal without writing desired group sizes into tenant state.
- [x] Add a tenant-scoped transition lock that serializes:
  - [x] normal and forced sleep;
  - [x] network and timer wake;
  - [x] proxy payload arrival during sleep;
  - [x] provider target cutovers.
- [x] Resume sleep-state reconciliation from durable state after process restart or shard-leader replacement.
- [x] Implement per-shard wake timers from `wake_at` and converge concurrent wake calls through the same CAS operation in the tenant-state manager.
- [x] Start and stop the tenant-state manager with shard leadership and compose timer and network wakes with reconciliation.
- [x] Ensure effective group size is zero only while the sleep entry exists, while preserving administrator/operator desired sizes in the existing group state.
- [x] Keep tenant state and server behavior shard-local. An nstance-server must not enumerate, contact, coordinate with, or wait for other shards; cross-shard workflows belong to nstance-operator or nstance-admin.
**Exit gate:** storage tests cover CAS conflicts, restart/failover, simultaneous wake triggers, sleep updates, and removal of empty tenant keys; reconciliation tests prove desired sizes survive sleep.
### Phase 3: Implement listener derivation, `nstance-server proxy`, and `nstance-server tunnel`
- [x] Derive the static `listeners` map at config load and reload time, with listener identity represented only by the map key. Use `<load-balancer-key>:<proxy-port>` for AWS and tunnel listeners; for Google Cloud, use `<load-balancer-key>:<port>` when the port is unique or `<load-balancer-key>/<frontend-ip>:<port>` when a shared port requires frontend-IP dispatch. Do not infer runtime wake eligibility from the static group size; reject cross-tenant references, invalid ports, duplicate Google Cloud frontend-IP/port selectors, proxy-port collisions, and server-port collisions.
- [x] Add one root-owned, proxy-accessible Unix gRPC control socket with a versioned initial-and-replacement listener configuration stream; make `nstance-server proxy` wait for the initial snapshot, validate complete replacements, reconcile listeners in place, retain the last valid snapshot only during a brief reconnect, stop leader-owned work on stream loss, and persist no proxy configuration state.
- [x] Add the `nstance-server proxy` subcommand:
  - [x] bind configured ports and dispatch Google Cloud shared ports by accepted frontend IP;
  - [x] expose health checks that never call `WakeTenant`;
  - [x] hold client connections for a bounded configurable timeout;
  - [x] call only listener-scoped `WakeTenant` over the root-owned Unix socket;
  - [x] forward to the returned private `IP:target_port` and let established connections drain after direct routing is restored.
- [x] Keep listener-scoped `WakeTenant` as the only operation besides the listener stream on the same root-owned, proxy-accessible Unix gRPC control socket.
- [x] Start and stop the wake server with shard leadership and return an upstream only after agent health and target-port readiness are both satisfied.
- [x] Add `nstance-server tunnel` as a separate privileged root supervisor process with a strictly permissioned Unix gRPC control socket, without distributing another binary:
  - [x] Keep node tunnel lifecycle intrinsic to the `knc` VM lifecycle; Nstance publishes no node tunnel lifecycle state and nstance-agent reports no tunnel-specific readiness.
  - [x] Use ordinary fresh agent health and requested upstream TCP readiness as Nstance's production recovery barrier.
  - [x] Have the supervisor initiate and reconnect the local stream to nstance-server, independently read the same root-controlled shard configuration, and resolve referenced secret/storage files itself.
  - [x] Define server messages containing only allowlisted tunnel name, desired running/stopped state, revision, and renewable lease; never send images, commands, mounts, manifests, secrets, credentials, storage contents, or provider details.
  - [x] Send heartbeats while leadership is held, revoke immediately on explicit leadership loss, and retain current state through an ambiguous disconnect only until the bounded lease expires before stopping fail-closed.
  - [x] Remove stale supervisor-owned static-Pod manifests and resolved files on startup before loading configuration because leases do not survive supervisor restart.
  - [x] Add a provider-neutral top-level `tunnels` block with immutable digest-pinned container images and mounted secret/storage files.
  - [x] Generate hardened kubelet static Pods and support either dedicated tunnel Pods or independently controlled logical endpoints in one shared local agent while any endpoint is active.
  - [x] Keep Cloudflare on dedicated processes sharing one existing tunnel credential, permit a future ngrok shared-agent implementation, and keep all provider-specific images/configuration in containers and generated deployment configuration rather than Nstance or vmconfig code.
  - [x] Implement revisioned readiness, restart/backoff, and no lifecycle state files.
- [x] Implement bounded secret-cache miss coalescing and validated file-patch delivery with hash/completion publication through existing agent file streams; instance rotation removes files that are no longer configured.
- [x] Add `server.files` generation and an atomic server-local file writer.
- [x] Publish `server.files` to the configured local files directory as part of the server composition.
**Exit gate:** proxy integration tests cover connection holding, timeout, listener isolation, frontend-IP dispatch, runtime exclusion of groups with zero preserved desired size, partial upstream health, restart, and concurrent wake calls.
### Phase 4: Strengthen provider load-balancer adapters and cutover state machines
- [x] Add provider lifecycle inspection for registered, healthy/routable, draining, partially registered, and fully deregistered targets. Ordinary deletion requires full deregistration; committed sleep permits termination of confirmed draining targets, but unknown or still-routable targets block deletion.
- [x] Keep provider transition progress in reconciled state so leadership changes safely continue or roll back a cutover.
- [x] Implement the common sleep sequence:
  1. register the shard-local proxy;
  2. wait for provider health/routability;
  3. start draining the shard-local production targets;
  4. for normal sleep, wait for a fresh agent report and perform the busy check immediately before withdrawal completes;
  5. restore production on activity/error, otherwise finish withdrawal and commit sleep. Forced sleep commits after confirmed withdrawal from new-connection selection, skipping activity checks and the remaining drain period without changing shared provider settings.
- [x] Implement the common wake sequence:
  1. restore desired infrastructure;
  2. wait for upstream and production provider health;
  3. withdraw proxy with draining;
  4. leave proxy installed and continue reconciliation if production readiness times out.
- [x] Never permit the intended registration sets to become empty, and never interpret provider fail-open routing as readiness.
- [x] AWS adapter:
  - [x] register instance targets with per-listener port overrides;
  - [x] inspect aggregate target health and deregistration completion across every configured target group;
  - [x] keep NLB and target-group cross-zone routing disabled and declaratively managed by OpenTofu/Terraform; do not expose runtime cross-zone mutation through nstance-server.
- [x] Google Cloud adapter:
  - [x] inspect configured NEGs at startup/reload and validate `GCE_VM_IP`, zone, and unique subnet mapping;
  - [x] select the NEG by the instance interface's subnet;
  - [x] add/remove VM-IP endpoints and inspect aggregate frontend/backend health and draining;
  - [x] support the Nstance-server subnet endpoint during sleep.
- [x] Tunnel adapter/state machine:
  - [x] while going to sleep, wait for `nstance-server tunnel` to report the requested local wake-tunnel name/revision ready before terminating control-plane VMs;
  - [x] while waking, keep the wake tunnel desired until a control-plane agent is freshly healthy and the requested local upstream is ready;
  - [x] permit node/`nst` tunnel-process overlap only during transitions and require the `nst` tunnel process to be absent in steady-state awake operation;
  - [x] reconcile local wake-tunnel ownership through shard leadership without persisting a distributed old/new-leader handoff; renew the new leader's desired name/revision lease, wait for readiness, and accept a bounded interruption during failover.
**Exit gate:** deterministic provider/tunnel-process fake tests cover healthy cutover, every timeout/rollback point, OpenTofu/Terraform-managed cross-zone routing remaining disabled, tunnel-process ordering, supervisor-initiated reconnect, heartbeat renewal, explicit revocation, ambiguous-disconnect lease expiry, fail-closed stopping, readiness/failure, process restart/backoff, hardened static Pods, dedicated and shared-agent layouts, absence of provider details and lifecycle state files, fail-open, shard-leader replacement, and no-empty-route invariants during ordinary sleep/wake cutovers. Live cloud testing is deferred to the final cross-repository matrix after OpenTofu/Terraform and vmconfig artifacts exist.
### Phase 5: Implement placement-aware Podplane Managed NAT
- [x] Change subnet placement to fill sorted `/26` node subnets to 90% of AWS-conservative capacity, then round-robin across available slots.
- [x] Model dedicated NAT members as derived per-populated-subnet instances rather than a fixed-size group. Exclude them from MachinePool import and replica distribution.
- [x] Add durable subnet-to-NAT assignment state, including each instance's node subnet, lifecycle, and optional fixed public IPv4.
- [x] Support optional fixed IPv4 egress without requiring it:
  - [x] AWS uses an Elastic IP reassociated with the active VM's primary ENI;
  - [x] Google Cloud uses a reserved regional external IPv4;
  - [x] each populated node subnet receives at most one configured fixed address, and explicit exhaustion blocks only NAT creation for another subnet.
- [x] Add provider route APIs and ownership guards:
  - [x] AWS subnet route-table default route to the selected primary ENI, with an optional Elastic IP association for fixed egress;
  - [x] Google Cloud subnet-specific node tags and tagged default routes to a forwarding VM;
  - [x] while Podplane Managed NAT is configured, correct only the tenant subnet's owned translation destination (`0.0.0.0/0` or `64:ff9b::/96`), even if its target was changed manually; retain cluster/resource ownership checks without a current-next-hop guard or persisted previous ENI IDs;
  - [x] leave peering routes, native IPv6 `::/0`, and unrelated destinations untouched, and preserve provider NAT Gateway routes during cleanup.
- [x] Enforce node creation ordering: select the subnet, wait for its NAT agent to register and submit a successful health report, install and verify the route, then create the node. Do not require a separate forwarding-readiness metric.
- [x] Implement last-node grace handling while awake, NAT-first wake, node-first sleep, and cleanup when changing back to Cloud Managed NAT. During normal or forced sleep, keep NAT until its last dependent workload is removed, then remove it on the next reconciliation without the grace period; retain fixed public IPs for reuse.
- [x] Implement Nstance's side of both mode transitions:
  - [x] Cloud Managed to Podplane Managed: establish healthy next hops and install Podplane Managed routes after deployment configuration enables them; retain provider/object-storage reachability and document the temporary general IPv4 egress outage.
  - [x] Podplane Managed to Cloud Managed: stop managed-route reconciliation and terminate tracked NAT instances after configuration disables Podplane Managed NAT; cleanup leaves provider NAT Gateway routes intact. Deployment tooling must coordinate the route handoff as specified in NAT.md and the Podplane rollout phase below.
- [x] Preserve native IPv6 routing through both mode changes; optional NAT64 remains a `nat` VM behavior rather than a provider route-mode change and may use the optional fixed IPv4 for translated traffic.
- [x] Implement vertical A/B replacement for dedicated NAT instances using readiness defined as agent registration plus a successful health report and the tenant-owned ordered `instance_type_ladder`. Start the replacement on its ordinary interface, wait for health, then reuse the subnet's existing fixed IPv4 when configured: reassociate the AWS Elastic IP and update the primary-ENI route, or move the Google Cloud reserved external IPv4 and update the tagged route. On nstance-server, derive packet-drop rates from consecutive timestamped reports and evaluate CPU, conntrack, and packet-drop thresholds and windows. Retain cumulative byte and packet counters for observability, but do not use them for scaling until a dependable provider-specific capacity signal exists. Scale up when any evaluated threshold remains exceeded for the configured two-to-five-minute window; scale down only when all evaluated metrics remain at or below their thresholds for the configured 20-to-30-minute window; enforce the configurable cooldown.
- [x] Replace failed or externally deleted NAT VMs through the NAT manager, preserving the workload subnet and optional fixed address. Leave existing routes in place when health is stale or missing, and switch only to a freshly healthy NAT. After interrupted replacement cleanup, persist routing-retry intent and restore the surviving NAT's route and address when fresh health returns, including after restart and without new workload creation. Retiring instances must not remove a replacement's route.
- [x] When fixed addresses are exhausted, prefer already-populated subnets with remaining capacity before blocking node scale-up. Do not block replacement of an existing NAT VM, because it reuses that subnet's current address.
**Exit gate:** tests cover placement capacity, concurrent creates, translation-destination ownership and correction with and without fixed IPv4, untouched peering/native IPv6/provider NAT Gateway routes, tenant conflicts, both ordered mode transitions, retained provider/object-storage access, mixed metric windows, alternate-subnet placement under fixed-address exhaustion, same-address vertical replacement/cooldown, external deletion, interrupted cutover, missing/stale health across restart, later fresh health and restoration retries without new workloads, NAT-first wake, and node-first sleep on both providers.
### Phase 6: Implement Kubernetes sleep policy in `nstance-operator`
- [x] Add operator configuration for disabled-by-default automatic sleep with defaults from ZERO.md: one remaining node, 30-minute inactivity, ten-minute CronJob look-ahead, and two-minute wake lead.
- [x] Watch CAPI Cluster/Machine/MachinePool and Nstance resources, Nodes, Jobs, CronJobs, and relevant Pods.
- [x] Reconcile `nstance.dev/sleep-request` annotations for normal and `force:<timestamp>` requests.
- [x] Evaluate normal-sleep eligibility exactly as specified, including pending scale operations, nonterminal/deleting Jobs, suspended Jobs with active/terminating Pods, CronJob UID ownership, time zones, and independent listener activity windows.
- [x] Calculate the earliest CronJob execution and place `wake_at` on one wake-capable shard.
- [x] Require self-managed nstance-operator deployments to run on a control-plane node. Require any in-cluster datastore quorum needed by the Kubernetes API to reside wholly in that node's shard, or to remain external and awake.
- [x] Process all shards with the operator node's shard last.
- [x] Before calling the final coordinating shard, persist a `finalizing` checkpoint containing that shard and every previously accepted shard. Require no Kubernetes write after a successful final call, and retain the sleep-request and progress annotations throughout sleep.
- [x] Persist all-shard compensation progress so restart resumes `WakeTenant` calls until every partially slept shard is awake; only then emit rejection and remove the annotation.
- [x] After any local wake, derive restoration from authoritative shard status, wake all remaining shards, remove the transaction annotations only after every shard is awake, and recompute the next CronJob deadline.
- [x] Add RBAC markers/manifests for CAPI Cluster patching and all watched workload resources.
**Exit gate:** envtest/operator integration tests cover eligibility, forced sleep, shard ordering, the pre-final-shard checkpoint with immediate Kubernetes API loss, partial failure compensation, operator restart during forward progress, finalization, or compensation, CronJob semantics, and post-wake restoration.
### Phase 7: Update canonical AWS and Google OpenTofu/Terraform modules
- [x] Change shared module input/output contracts under `deploy/tf/common` first, then update both provider implementations.
- [x] AWS network/account/shard modules:
  - [x] one regional target group per exposed listener, shared across zones/shards;
  - [x] TCP health checks, draining, OpenTofu/Terraform-managed cross-zone routing disabled on the NLB and target groups, and target-group metadata including listener/target/proxy ports;
  - [x] one route table per node subnet;
  - [x] Cloud Managed versus Podplane Managed NAT switch;
  - [x] optional pre-provisioned Elastic IPs for fixed NAT egress;
  - [x] public service subnet placement and public IPv4 for `nst` under Podplane Managed NAT;
  - [x] scoped ELB, route, ENI, EC2, and S3 permissions, a free S3 gateway endpoint, and explicitly opted-in private interface endpoints rather than automatic EC2 endpoint creation;
  - [x] security groups that expose only configured NLB health/proxy paths while keeping Nstance registration, operator, agent, election, and server-health endpoints private.
- [x] Google Cloud network/account/shard modules:
  - [x] regional external passthrough Network Load Balancers;
  - [x] one `GCE_VM_IP` NEG per logical membership set, zone, and eligible production/Nstance subnet;
  - [x] backend services, forwarding rules, reserved frontend IPs, health checks, and firewall rules;
  - [x] Cloud NAT restricted to node subnet ranges in Cloud Managed mode;
  - [x] Podplane Managed tagged routes and optional reserved external IPv4 addresses for fixed NAT egress;
  - [x] `can_ip_forward`, external IPv4 for `nst` in Podplane Managed mode, Private Google Access, and scoped NEG/route permissions;
  - [x] firewall rules that expose only configured load-balancer health/proxy paths while keeping every nstance-server API private.
- [x] Pass only static resource identity/configuration to nstance-server; do not put dynamic target membership in OpenTofu/Terraform state.
- [x] Accept one shard-module `tunnels` map combining listeners and runtime configuration. Derive matching `load_balancers` entries with `provider = "tunnel"` instead of requiring a duplicate input map; support inline non-secret `env`/`json`/`string` templates alongside secret/storage references.
- [x] Add `nst` vmconfig userdata inputs for optional proxy/tunnel settings and shard endpoints, and pass each tenant's optional fixed public IPv4 entries directly through the generated Nstance configuration. Do not place an operator nonce in VM userdata, OpenTofu/Terraform outputs, or state.
- [x] Update examples and module contract tests for Cloud Managed NAT, Podplane Managed NAT, NLBs, tunnels, and NAT mode changes that preserve node subnets.
**Exit gate:** `tofu validate` and provider-specific fixture plans pass; plans show no node-subnet replacement when switching NAT modes, no duplicated AWS target groups or same-zone Google NEGs, and no public path to nstance-server APIs. Network tests prove ordinary and fixed NAT egress, fixed-address reuse during replacement, public traffic reaching configured proxy/health listeners, and the absence of public access to `WakeTenant`, which remains Unix-socket-only.
### Phase 8: Publish compatible Nstance 2.x artifacts
- Document breaking configuration/protobuf/storage changes, IAM additions, disruptive NAT route switches, Cloudflare trust boundaries, deployment requirements, and rollback boundaries; do not provide 1.x migration shims.
- Release the server binary and its proxy/tunnel subcommands, agent, operator, Helm chart, and Terraform source changes together as one compatible pinned 2.x release set.
- State explicitly that mixed Nstance 1.x/2.x deployments and 1.x configuration or storage formats are unsupported.
- Trigger the established OpenTofu/Terraform sync workflow from this canonical repository and record the exact source commit in both generated mirror commits.
**Exit gate:** all required Nstance 2.x artifacts are published, generated OpenTofu/Terraform mirror commits reference the release source commit, and downstream repositories can pin the complete immutable version set.
## Repository: `podplane/vmconfig`
### Phase 1: Introduce reusable service configuration and new VM kinds
- [x] Extract only the shared install/configure/restart machinery into `templates/common`, then build the VM kinds as `common + knd`, `common + knd + knc`, `common + nst`, and `common + nat`, in overlay order.
- [x] Keep `nst` and `nat` independent. Only the `nat` kind configures forwarding and SNAT.
- [x] Add `nst` kind for Nstance-server VMs and `nat` kind for dedicated NAT VMs, each with manifests for Debian 13 on amd64 and arm64.
- [x] Use the same architecture-specific Debian base image for every VM kind; kinds vary by installed packages and configuration, not by OS image.
- [x] Add manifest dependencies for `nstance-server`. Include `bpftool` and required eBPF runtime libraries in every `knd`/`knc` manifest; their small installed footprint does not justify conditional manifest machinery. Tunnel provider binaries are supplied by immutable container images rather than vmconfig packages.
- [x] Preserve strict artifact verification, reproducible packaging, standard Fluent Bit, and the existing receive/reconfigure workflow.
**Exit gate:** package and install tests cover all four kinds (`knd`, `knc`, `nst`, and `nat`) on both architectures without regressing existing manifests. Kubernetes-node fixtures download, verify, and install the pinned active-connection counter toolchain. Tunnel provider binaries remain absent from vmconfig packages.
### Phase 2: Add active connection counters to Kubernetes nodes
- [x] Build and package the vmconfig-owned eBPF ELF for active TCP connections. Commit its source and generated object, pin the Clang version with mise, and have CI fail when rebuilding produces a different object.
- [x] Derive the monitored ports from each group's production listeners: ingress ports only, kube-apiserver port only, their union for a shared group, or no counters for a group with neither role.
- [x] Add a root-owned oneshot service that loads the ELF with `bpftool`, auto-attaches tracepoint programs, and pins links and maps under `/sys/fs/bpf/nstance/counters`.
- [x] Grant nstance-agent read access only to link metadata needed for validation and the `active_connections` map.
- [x] Configure the loaded program to ignore ports outside the derived set. When no ports require monitoring, leave the installed toolchain dormant: do not enable the loader, attach programs, create pins, set `NSTANCE_EBPF_COUNTERS_PATH`, or require counters from the agent.
- [x] Ensure reloads update configured ports without leaving stale links/maps and that pin loss is visible as `ebpf_error` rather than silently reporting zero.
**Exit gate:** VM tests cover ingress-only, kube-apiserver-only, combined, and disabled configurations; generate traffic on configured and ignored ports; and verify zero/nonzero counts, pinned-link loss, permissions, reboot persistence, and complete runtime absence when disabled.
### Phase 3: Implement `nst` proxy and tunnel services
- [x] Configure `nstance-server` with the existing vmconfig user-data and watch machinery rather than the Nstance module's generic server userdata.
- Treat `nst`'s `mutable.env` as provision-once configuration, not an agent-updated file. Render non-secret server settings through the same `podplane_userdata` data source used for other kinds; share SSH/telemetry/registry values without copying static access keys into user-data. Configuration changes roll `nst` VMs. Derive the server ID from existing provider-instance detection without changing the provider-assigned hostname.
- [x] Enable `nstance-server proxy` only when sleep support is configured. Keep it running while awake and asleep; only Nstance changes external routing.
- [x] Create runtime users and strict ownership for the root-owned, proxy-accessible Unix gRPC control socket; configure `nstance-server proxy` to reconnect and reconcile streamed listener snapshots in place without disk-backed configuration.
- [x] Install and harden a separate privileged root `nstance-server tunnel` process and its strictly permissioned Unix gRPC socket. Nstance and vmconfig code must remain provider-neutral; deployment-generated root-controlled shard configuration supplies immutable images and mounted secret/storage file references.
- [x] Have `nstance-server tunnel` independently resolve files, initiate/reconnect its server stream, enforce renewable leases, report revisioned readiness/failure, apply restart/backoff, and generate hardened kubelet static Pods without lifecycle state files.
- [x] Give `nst` the same containerd and Podplane Registry configuration as `knd`, allowing tunnel Pods to use the same S3/GCS-backed image distribution. Configure its standalone kubelet to use the standard `/etc/kubernetes/manifests` static-Pod directory rather than a tunnel-specific path.
- [x] Keep node tunnels VM-lifecycle-owned and entirely outside Nstance control.
- Extend the existing `knc`/`knd` receive workflow for tunnel-mode `ingress.server.crt` and its agent-generated `ingress.server.key`; validate the pair and atomically publish `/opt/podplane/secrets/ingress/node.pem` in a dedicated root-owned directory readable only by the SDS runtime group. Do not expose unrelated agent keys or make private files world-readable. Request no ingress material when ingress is absent.
- Deliver the existing public Nstance cluster CA to node and `nst` tunnel containers before startup, without requiring the Podplane operator or publishing ingress private keys. Keep DNS origin verification enabled on local and wake-proxy routes.
- Generate Cloudflare-specific deployment configuration, without adding Cloudflare logic to vmconfig:
  - one node tunnel process per control-plane VM for kube-apiserver plus optional ingress;
  - one dedicated tunnel process per eligible Nstance server, sharing the node tunnels' existing tunnel ID and credential-secret reference;
  - HTTPS origin verification using the existing Nstance cluster CA and the configured API/ingress DNS identity;
  - no TLS verification disablement;
  - no secrets in arguments, logs, generated OpenTofu/Terraform values, or world-readable files.
**Exit gate:** local VM tests prove Unix-stream reconnect and snapshot replacement, in-place proxy listener reconciliation, supervisor-initiated reconnect, heartbeat renewal, immediate revocation, bounded disconnect lease expiry, fail-closed stopping, wake-tunnel readiness/failure and restart/backoff, static-Pod hardening, secret/storage resolution without server delivery, dedicated and shared-agent layouts, absence of provider fields and lifecycle state files, atomic matching ingress bundles and permission isolation, and TLS verification for both API and ingress origins without operator startup.
### Phase 4: Implement NAT host behavior
- [x] In the `nat` overlay, configure IPv4 forwarding, source NAT, conntrack limits, and Nstance agent reporting of current CPU and memory, conntrack count and limit, and raw cumulative byte, packet, and drop counters for `NSTANCE_METRICS_INTERFACE`, including explicit collection errors. Do not calculate rates or windows in vmconfig or report a separate forwarding-readiness metric.
- [x] Support optional stateful NAT64 with Jool, defaulting to `64:ff9b::/96`. Require an IPv6 source allowlist and IPv4 pool address, block translation to non-global IPv4 destinations, and verify signed Jool modules against the exact running kernel before activation.
- [x] Keep `nat` images free of containerd, kubelet, their users, runtime images, configuration, and services.
- [x] Ensure AWS source/destination-check and Google `can_ip_forward` remain Terraform/provider concerns, not shell-script cloud API calls.
- [x] Make forwarding activation/deactivation idempotent and safe across reboot.
**Exit gate:** network-namespace or VM integration tests verify IPv4 forwarding, SNAT, NAT64 translation and filtering, idempotent deactivation, container-runtime absence, raw cumulative interface counters, conntrack and current-utilization reporting, and explicit collection errors for `nat`; Nstance integration tests verify readiness from registration plus a successful health report, without a forwarding-readiness metric.
### Phase 5: Publish manifests
- Publish all new kind/architecture packages and manifests.
- Pin a compatible Nstance 2.x release in each new manifest.
- Leave historical immutable manifests unchanged, but do not support mixing incompatible release contracts.
**Exit gate:** immutable manifest/package versions are published for every required kind and architecture, verify successfully from a clean cache, and are recorded in the cross-repository release set.
## Repository: `podplane/components`
### Phase 1: Add Nstance component charts and wire dependencies
- Add `nstance-operator-crds` by vendoring the Nstance CRDs through the repository's existing CRD update process.
- Add an opinionated `nstance-operator` chart based on the upstream Nstance Helm chart, following Podplane conventions for namespaces, image mirroring, security context, probes, leader election, and CRD skipping.
- Configure dependencies in `platform-components`:
  - `secrets-store-csi-driver` depends on `secrets-store-csi-driver-crds`;
  - the selected Secrets Store CSI provider depends on `secrets-store-csi-driver`;
  - `podplane-operator` depends on `podplane-operator-crds`, the CSI driver, and the provider selected for its workload CA mount;
  - `cluster-api` depends on `cluster-api-crds` and `podplane-operator`;
  - `nstance-operator` depends on `nstance-operator-crds` and `cluster-api`.
- Use Kubernetes `podCertificate` projections for CAPI and Nstance webhook Service certificates, signed by `certificates.podplane.dev/workload`, and annotate webhook configurations and CAPI conversion CRDs with `certificates.podplane.dev/inject-ca-from: workload`. Follow [CERTS.md](./CERTS.md#platform-serving-certificates-and-ca-injection); do not introduce cert-manager resources or TLS Secrets for these webhooks.
- Keep certificate dependencies separate from Nstance registration. The CSI stack delivers the workload CA key to `podplane-operator` for signing webhook certificates; Nstance registration uses the seeded 30-minute nonce Secret and its own public CA ConfigMap, not CSI or the Podplane Secrets API.
- Mark the Nstance/Cluster API provider-overlay entries `core: true` but `enabled: false` in chart defaults. Podplane enables them only for AWS/Google `recommended` clusters; the existing recommended Pod Certificate stack satisfies both CAPI and Nstance webhook certificate requirements.
- Add nstance-operator images and both architectures to `manifests/components.json` with provider metadata for AWS and Google Cloud; do not mark them as generic recommended addons.
**Exit gate:** Helm lint/render tests show the complete dependency graph, projected webhook certificates, and workload-CA injection annotations, with CRDs handled only by the CRD releases and no cert-manager resources. Cluster tests verify webhook startup and certificate renewal through the Pod Certificate stack.
### Phase 2: Add seed-contained operator bootstrap and RBAC
- Expose values for cluster ID, tenant, shard registration/operator endpoints, public CA ConfigMap, nonce Secret name/key, and automatic-sleep policy.
- Reference a dedicated bootstrap Secret that the Podplane Terraform provider inserts into the cluster-specific Netsy seed. Do not render the nonce into HelmRelease values or a chart-owned Secret.
- Give the nonce a 30-minute TTL, make it single-use, tenant- and cluster-scoped, and restrict it to operator registration. Once an operator certificate exists, nstance-operator must not attempt registration again.
- Delete the nonce Secret only after the certificate and matching private key have been durably stored and verified. Historical Netsy revisions, bucket versions, or backups may retain the consumed nonce; that is acceptable because Nstance has invalidated it.
- Include CAPI Cluster get/list/watch/patch/update permissions, Job/CronJob/Pod/Node reads, Event creation, Lease leader election, and Nstance CRD permissions.
- Keep the operator highly available during normal operation and schedulable on the final supported one-node topology.
- Ensure platform RBAC initially limits sleep requests to `podplane:admins`, which is bound to `cluster-admin`; the CLI's access review is an additional check, not the authorization boundary.
**Exit gate:** RBAC tests demonstrate the operator can reconcile sleep requests but cannot mutate unrelated cluster-scoped resources, and non-admin users cannot patch the sleep annotation under the shipped policy. Bootstrap tests prove the operator registers from the seeded Secret, deletes that Secret only after durable certificate storage, retries safely after a lost registration response, and reuses its managed certificate after the nonce expires.
### Phase 3: Preserve Envoy host exposure and select certificate delivery
- Keep Envoy as a DaemonSet with host ports 80 and 443 in both NLB and tunnel modes; tunnel configuration simply omits a port-80 origin route.
- Add explicit scheduling/toleration tests proving Envoy remains present on control-plane nodes when tunnel ingress is enabled.
- Use internal `platform.envoyGateway.ingress.certificates.provider`: `node` for tunnel mode, or the selected CSI provider (`aws`, `gcp`, `vault`, or `openbao`) for NLB mode, with no chart default. Preserve per-domain CSI/SDS delivery for external providers; for `node`, mount only `/opt/podplane/secrets/ingress` read-only into SDS and render no ingress `SecretProviderClass` or external-backend requirement.
- Mount both sources at the Operator's default `/var/run/podplane/ingress-certificates` and omit `--certificates-dir`. No Operator directory change is required.
- Point every tunnel ingress listener's metadata-only SDS reference at the shared `node` resource. Keep all domain/listener configuration, the Pod-local Unix socket, and sidecar hardening; mount no ingress key into the Envoy Gateway control plane.
- Omit operator ingress certificate domains in tunnel mode without disabling its separate workload/webhook certificate stack. Existing chart/runtime empty-domains behavior is sufficient; no new operator enable flag is required.
- Do not route kube-apiserver through Envoy Gateway.
**Exit gate:** render and cluster checks verify control-plane Envoy port 443 and direct API port 6443; node-local read-only delivery with one SDS resource across multiple domains; no tunnel ingress CSI class or operator certificate manager; and unchanged NLB fallback/ACME mounts and references.
### Phase 4: Correct component-set semantics and documentation
- Document `nstance-operator` as a protected AWS/Google provider overlay for `recommended` clusters, not part of the generic recommended addon set.
- Remove or correct the existing stale documentation that describes Nstance as a recommended addon before the charts exist.
- Keep `minimal`, `none`, and local/providerless seeds free of the provider Nstance overlay unless explicitly configured for a Nstance development environment.
- Publish the Components charts and component image manifest. Seed generation and publication belong to `podplane/seedgen` and `podplane/seeds`, respectively.
**Exit gate:** an immutable Components release contains the charts, provider metadata, and complete multi-architecture image manifest required to regenerate both seeds.
## Repository: `podplane/operator`
### Phase 1: Support node-local ingress through SDS
- Extend the SDS loader to serve `node.pem` as one `node` resource for every configured domain, validating key correspondence, validity, server-authentication usage, chain signatures, and all apex/wildcard SAN coverage while accepting the additional localhost and node/loopback IP SANs. Keep the existing exact per-domain validation for NLB/ACME bundles.
- Preserve initial fail-closed serving and last-known-good handling for invalid later updates, and reload atomic file replacements without restarting Envoy.
- Use one comma-separated `--node-certificate` domain list for `node.pem`, exclusive with repeatable `--domain-certificate=<resource>=<domain>` flags. Reserve the resource name `node`; retain the existing certificate-directory default.
- Keep fallback and ACME issuance disabled through the existing absent/empty `ingress_certificates.domains` configuration; add no ingress issuer or Nstance signing client to the operator.
**Exit gate:** SDS tests exercise one certificate covering multiple domains, missing SANs, wrong domains, key mismatch, expiry, initial absence, valid atomic replacement, and invalid-update retention. Existing per-domain NLB/ACME tests remain unchanged in behavior.
## Repository: `podplane/podplane`
### Phase 1: Add user-facing configuration and schemas
- Add one optional singular `cluster.tunnel` config alongside `cluster.domains` and `cluster.acme`, independent of the infrastructure provider. Absent means provider-default NLBs for both Kubernetes API and ingress; present means tunnels for both and no NLB resources or registrations.
- Require an existing tunnel's `kind`, `id`, and one `credential_secret` reference; resolve its image from VM manifests. Reject per-endpoint routing selectors, a custom third mode, mixed API-NLB/ingress-tunnel exposure, and in-place NLB/tunnel switching. Keep node and `nst` process lifecycles separate while sharing tunnel identity and credentials; do not narrow Nstance's provider-neutral multi-entry `tunnels` or runtime listener contracts.
- Add NAT mode (`provider` or `nstance`), optional `fixed_public_ipv4_count` per NAT service subnet, group deployment settings (`arch`, starting `instance_type`, public `subnet_pool`), and only `group` and `instance_type_ladder` under `tenants.default`. Generate address references and use Nstance's policy defaults; do not expose its timing, thresholds, or direct `public_addresses` in Podplane cluster config.
- Do not add a Podplane `SleepPolicy` or generate policy/timing overrides. Nstance/operator defaults own activity windows, CronJob look-ahead, wake lead, proxy connection holding, and provider cutover/upstream readiness timing; automatic sleep remains disabled by default without disabling manual CLI sleep.
- Validate:
  - scale-to-zero requires an AWS or Google Cloud cluster using the `recommended` seed;
  - supported AWS/Google `recommended` deployments have wake-capable API and ingress routing in either exposure mode without a sleep-policy flag;
  - `cluster.tunnel` and `cluster.acme` are mutually exclusive;
  - `nst` placement is public/direct when Podplane Managed NAT is selected and private by default for Cloud Managed NAT;
  - NLB/tunnel listener and proxy-port rules from LB.md;
  - NAT group/subnet/tenant constraints from NAT.md;
  - Google Cloud's equal passthrough ports and AWS proxy-port uniqueness.
- Update JSON Schema, config examples, TUI defaults, config docs, and focused validation tests together.
**Exit gate:** both exposure modes with both NAT modes round-trip through AWS/Google config parsing and schema validation; unsupported selectors, mixed/custom modes, Podplane sleep-policy settings, invalid no-wake cases, and port collisions fail before OpenTofu/Terraform generation.
### Phase 2: Generate infrastructure and Nstance/vmconfig inputs
- Generalize OpenTofu/Terraform generation to AWS and Google Cloud module sources with equivalent topology inputs.
- Emit only the selected cloud's required provider and account/project locals. Expose AWS `enable_ssm` versus Google `enable_iap` and `enable_os_login`; share non-secret host settings between NST and managed nodes, and omit root-disk size overrides when unset on either provider.
- Emit Cloud Managed versus Podplane Managed NAT resources without changing node-subnet identity.
- Default new node placement subnets to `/26` without rewriting existing subnet CIDRs. Generate the top-level tenant-keyed Nstance `nat` configuration with the group, instance-type ladder, and optional generated address references; omit policy overrides so Nstance defaults apply. Generate the referenced group's deployment template, public service subnet pool, and starting `instance_type`, plus the `nat` vmconfig manifest references.
- In NLB mode, generate provider-specific load-balancer metadata in the frozen Nstance configuration shape, including all production and Nstance-server subnets required for Google NEGs. Preserve each Google forwarding-rule frontend IP and its single equal frontend, proxy, and target port so Nstance can derive the finalized static listener map keys and frontend-IP dispatch selectors. Tunnel mode emits only tunnel routing entries, with no NLB resources or registrations.
- Select and pin `knc`, `knd`, `nst`, and `nat` vmconfig manifests by architecture. Pass listener-derived active-connection ports only to affected `knc`/`knd` groups participating in guarded normal sleep; tunnel provider software remains in digest-pinned container images.
- Replace generic Nstance-server userdata with the `nst` vmconfig kind. Enable proxy services and wake routing for supported AWS/Google `recommended` deployments without a sleep-policy flag or timing overrides, and pass the root-controlled provider-neutral `tunnels` block in tunnel mode through generated deployment configuration.
- In tunnel mode, generate `ingress.server` with all configured apex/wildcard DNS SANs plus `localhost`, and IP SANs for `127.0.0.1`, `::1`, and the Envoy node's assigned IPv4/IPv6 addresses through the existing instance templates; omit unassigned address families. Use server-only usage and the existing 8760-hour VM certificate lifetime. Request an agent-owned ingress key/certificate on every Envoy-capable node, including `knc`; reuse the existing Nstance template/file contracts rather than adding cluster-scoped issuance or secret resources. Node retirement is the lifetime strategy, not a hard guarantee against delayed replacement.
- Set Envoy's `ingress.certificates.provider` to `node` in tunnel mode or the selected CSI provider in NLB mode. In tunnel mode, explicitly remove operator ingress certificate configuration through seed merging so base domains, defaults, and overlays cannot re-enable fallback or ACME; preserve Envoy domains and the independent workload/webhook stack. In NLB mode, retain operator-owned fallback with optional ACME.
- Match captured SDS mounts to the Components host path and default container directory, and remove captured `--certificates-dir` overrides when generating cluster-specific seeds.
- Generate node and `nst` tunnel runtime configuration, manifest-derived immutable image references, inline non-secret JSON, and mounted credential/storage references without creating provider tunnels or serializing credentials into OpenTofu/Terraform values, process arguments, logs, or world-readable files. Copy only the public cluster CA into each shard's tunnel file store.
- Use existing public HTTPS Cloudflare tunnels with no port-80 origin. Document user-managed DNS and hostname-scoped HTTP-to-HTTPS edge redirects rather than creating them through Terraform.
- Generate stable API/ingress endpoint outputs:
  - NLB API defaults to external port 6443 and embeds the Kubernetes CA;
  - Cloudflare API uses external port 443 and public CA roots;
  - kube-apiserver remains independent of Envoy Gateway.
- Pass only non-sensitive operator bootstrap configuration—cluster ID, tenant, shard addresses, CA ConfigMap name, and nonce Secret name/key—into component values. The state-safe provider operation inserts the actual one-time nonce directly into the cluster-specific Netsy seed.
**Exit gate:** golden OpenTofu/Terraform and seed tests cover AWS and Google Cloud for NLB and Cloudflare API-plus-ingress tunnels with both NAT modes; supported recommended deployments include wake routing without policy/timing overrides, and tunnel deployments contain no NLB resources or registrations. Multi-domain tunnel nodes receive one ingress certificate request with the complete SAN set; operator ingress management remains absent after merging while Envoy stays configured. NLB fallback/ACME generation is preserved. Sensitive values are absent from generated non-sensitive files.

Current golden scenarios are `aws-nlb-provider-nat` and `google-tunnel-nstance-nat`. They are representative combinations, not a restriction coupling a cloud, tunnel choice, or NAT mode; the broader deployment matrix remains a separate validation gate. Local clusters independently select the same node-certificate/SDS path using the local fake Nstance CA, without enabling tunnels.

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
  - `podplane-operator`, its CRDs, the Secrets Store CSI driver/CRDs, and the selected workload-CA provider if not already enabled by the selected seed;
  - `cluster-api-crds` and `cluster-api`;
  - `nstance-operator-crds` and `nstance-operator`;
  - existing provider-specific storage components.
- Enable the certificate stack for webhook TLS as defined in [CERTS.md](./CERTS.md), not for Nstance registration. Keep the registration nonce in its seeded Secret; do not route registration through CSI or the Podplane Secrets API.
- Apply the Nstance overlay only after loading `recommended` and only for AWS/Google providers.
- Leave `recommendedAddons` unchanged with respect to Nstance.
- Leave `minimal` and `none` without the Nstance operator, CAPI, or operator-bootstrap resources, and leave local/providerless recommended seeds without the overlay.
- Generate component values for shard endpoints and bootstrap ConfigMap/Secret references without conflating these with user-selectable recommended addons. Do not generate sleep-policy or timing overrides; retain Nstance/operator defaults and Components' own configurable policy settings.
- Add tests for AWS/Google versus local/providerless seeds and for recommended/minimal/none behavior.
**Exit gate:** AWS/Google cluster-specific snapshots derived from `recommended` contain the Pod Certificate stack, CAPI, the Nstance operator, and bootstrap objects, with no cert-manager resources; generic recommended addon enumeration does not include Nstance; `minimal`, `none`, and local/providerless snapshots omit the Nstance overlay without changing their independent certificate provisioning rules. Running-operator verification occurs after the cluster boots from the uploaded seed in the final end-to-end matrix.
### Phase 5: Add `podplane cluster sleep`
- Add `podplane cluster sleep` and `podplane cluster sleep --force` under the Kubernetes-context command conventions, without requiring a sleep-policy flag.
- Resolve `--context` / `--kubeconfig`, defaulting to the current kubeconfig context rather than implicitly reading a cluster config.
- Issue a `SelfSubjectAccessReview` for patching the target CAPI Cluster before mutation.
- Normal mode writes `nstance.dev/sleep-request=<timestamp>` and returns immediately.
- Forced mode displays an explicit interruption warning and confirmation, then writes `force:<timestamp>`.
- Do not add a wake command; document that API or ingress traffic wakes the cluster.
- Report annotation/authorization failures clearly without waiting for the cluster to sleep.
**Exit gate:** command tests cover access denial, normal and forced annotations, confirmation, noninteractive behavior, config/context selection, and immediate disconnect.
### Phase 6: Provider rollout and Nstance 2.x deployment
- First enable forced sleep on a single-zone AWS test cluster, then multi-zone AWS, then equivalent Google Cloud topologies.
- Roll out Cloudflare API and ingress tunnel exposure together on recreated infrastructure; verify both wake paths and administrative recovery, not an API-only or mixed-mode deployment.
- Verify that every replacement/woken Envoy node obtains its own ingress key and certificate before TLS serving; node and `nst` tunnels accept it through the existing cluster CA with hostname verification. Prove this path does not wait for operator ingress issuance or read an old node's key from external storage.
- Keep automatic sleep off unless the pinned operator, proxy, NAT, and exposure release set passes the provider matrix.
- Require coordinated, compatible Nstance 2.x deployment rather than mixed 1.x/2.x operation; document cluster recreation where an existing deployment cannot be cut over atomically.
- Define deployment behavior:
  - add provider-default components and wake-capable routing for supported recommended deployments while leaving policy/timing settings to Nstance/operator defaults;
  - require infrastructure deletion/redeployment when changing NLB/tunnel exposure; do not promise that retaining Netsy data alone preserves complete cluster state;
  - rotate Nstance-server VMs when adopting `nst` or changing NAT placement;
  - never rotate Kubernetes nodes solely to change NAT mode;
  - verify Nstance has observed Podplane Managed NAT disablement before handing the translation route to Cloud Managed NAT; do not rely on the old next hop as a guard;
  - during sleep/wake transitions within the selected exposure mode, preserve old direct routing until the proxy path has passed readiness; this is not an in-place exposure-mode migration.
- Update user docs for cost/availability tradeoffs, Cloudflare trust, NAT connection resets, force-sleep risk, and rollback.
**Exit gate:** forced sleep passes the staged AWS and Google Cloud rollout; Cloudflare API and ingress wake both work on recreated tunnel deployments; NAT-only changes preserve Kubernetes nodes under NAT.md's contract; exposure-mode changes require deletion/redeployment; automatic sleep remains gated off pending the final matrix.
## Repository: `podplane/terraform-provider-podplane`
### Phase 1: Add state-safe Nstance operator seed bootstrap
- Add a state-safe signing-key resource that conditionally creates Nstance's registration key in its encrypted secrets store or adopts the existing key after an interrupted apply/Nstance lazy initialization. State contains only backend identity, version, and hash; updates never rotate the key automatically.
- Make the existing Netsy seed resource depend on the signing-key resource and boundedly wait for Nstance's public `ca.crt`.
- In the seed resource's Create operation only, read and decrypt the signing key into memory and use Nstance's public `pkg/nonce` contract to mint the 30-minute tenant- and cluster-scoped nonce JWT. Inject the nonce Secret and CA ConfigMap and upload the completed cluster-specific seed without intermediate plaintext files.
- Before reading the key, detect and adopt an already-completed seed; normal Read/refresh inspects only non-secret metadata. This recovers both a write-before-state signing-key interruption and an upload-before-state seed interruption.
- Keep the signing key and nonce out of schema values, plans, state, outputs, command arguments, diagnostics, logs, and temporary files; discard in-memory plaintext after Create.
- Rely on Nstance's durable, atomic binding of the exact nonce JWT to the byte-identical operator public-key PEM: retrying that exact pair after a lost response succeeds, while any different JWT or PEM is rejected, including after leader replacement. Provider create/adopt behavior must tolerate delayed bootstrap and apply interruption without attempting to recreate that registration record.
- Do not add automatic initialized-cluster repair to the Terraform provider. Document the simple recovery contract: use the canonical Nstance admin nonce command and `kubectl` to replace the bootstrap Secret when the cluster is live; destroy and recreate a cluster that never became live. Never re-upload an initial seed over live Netsy state.
**Exit gate:** provider integration tests cover key create/adopt races, seed create/adopt paths, proof that key reads occur only when seed creation is required, the 30-minute nonce TTL, delayed CA bootstrap, use of public `pkg/nonce`, apply interruption, and leader change. Registration tests prove that the exact JWT plus byte-identical public-key PEM succeeds after a lost response and that a different JWT or PEM is rejected. Tests permit the nonce only inside the encrypted cluster-specific seed; live-cluster expiry recovery remains a documented manual procedure and cluster test.
### Proposed interface follow-up: resolved infrastructure inputs
Standard generated wiring currently uses `values_content`. The discussed alternative is a small set of typed inputs for Terraform-resolved shard endpoints, registry bucket/role, and DNS identifiers, translated into component values inside `netsyseed` and shared with the local CLI. Keep cluster config as the source for facts it already contains, `values_content`/`values_file` as customization escape hatches, and `nstance_bootstrap` separate. This refactor is not implemented or a finalized public schema; do not make it a prerequisite for the existing bootstrap path.
## Repository: `podplane/seedgen`
seedgen turns a running reference cluster into audited, provider-neutral minimal and recommended Netsy snapshots. Provider-required component enablement is applied later by Podplane while it builds the cluster-specific snapshot; seedgen does not own that selection logic.
### Phase 1: Confirm provider-neutral filtering
- Verify that existing include/exclude and expected-record rules retain the updated `platform-components` release while excluding per-cluster runtime credentials and identity resources.
- Do not add rendered Nstance operator, Cluster API, CA, nonce, or operator identity records to the generic seed merely because AWS/Google clusters enable them later.
- Keep generic `minimal` free of the Nstance operator and Cluster API; the Nstance provider overlay applies only to an AWS/Google cluster-specific snapshot derived from `recommended`. Certificate-stack selection follows [CERTS.md](./CERTS.md) independently; reusable seeds contain neither cert-manager resources nor workload CA private keys.
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
- Build a requirement-to-test matrix, then exercise AWS and Google Cloud in one/multiple zones, both NAT modes, NLB for both API and ingress, and Cloudflare tunnels for both on both providers; wake-capable recommended generation without policy/timing overrides; manual forced/normal sleep and automatic sleep configured directly through Nstance/Components; rejected mixed/custom exposure and exposure changes requiring deletion/redeployment; shared/separate listener groups in Nstance; one-to-zero and redundant two-zone sleep; timer/network wake; provider/proxy timeouts; operator restart; and shard-leader replacement.
- Keep automatic sleep disabled by default until both providers pass all routing, NAT, sleep-compensation, packaging, bootstrap, release, and deployment gates from the repository phases above.
