# Podplane NAT

> **STATUS**: In progress

## Goal

Podplane can use either Cloud Managed NAT or Podplane Managed NAT VMs without changing the cluster subnet model. AWS and Google Cloud use provider-native routing, but expose the same Podplane configuration and lifecycle.

## Network model

- Node IPv4 subnets are `/26`, providing 59 usable addresses on AWS and 60 on Google Cloud.
    - Nstance conservatively calculates usable capacity as the subnet size minus AWS's five reserved addresses on both providers. It fills each sorted subnet to 90% of that capacity before using the next, targeting 53 nodes for `/26`.
    - Once all configured subnets are at 90%, it should round-robin until it finds an available slot.
    - As you will learn, this will reduce the total number of required Podplane Managed NAT instances.
- Each populated node subnet has one dedicated NAT VM in a public/NAT service subnet in the same zone.
- One generated OpenTofu/Terraform variable selects `cloud-managed` or `podplane-managed` NAT. Reapplying changes Cloud Managed NAT resources, route configuration, Nstance configuration, and required NAT VMs; node-subnet definitions remain unchanged.
    - This means cluster operators can switch between Cloud Managed NAT and Podplane Managed NAT without any cluster node changes or rotations. Cloud Managed to Podplane Managed causes a temporary IPv4 egress outage while Nstance reconciles.

Podplane Managed NAT always places `nst` VMs in public service subnets on AWS and gives them external IPv4 addresses on Google Cloud. Inbound access remains denied except where explicitly required. Enabling Podplane Managed NAT may rotate Nstance-server VMs but not Kubernetes nodes.

Dedicated NAT VMs normally use their provider-assigned public IPv4 addresses.
When stable egress allowlisting is required, the cluster administrator may
configure fixed public IPv4 addresses per zone/shard. OpenTofu/Terraform
pre-provisions those addresses and passes them directly in each tenant's NAT
configuration:

- AWS uses an Elastic IP reassociated with the active VM's primary ENI.
- Google Cloud uses a reserved regional external IPv4 address.

Nstance assigns at most one fixed address to each populated node subnet and
retains that assignment across vertical replacement. The replacement starts on
its ordinary interface and becomes healthy before cutover, so it does not need
a second fixed address. Without configured fixed addresses, replacement may
change the subnet's provider-assigned public IPv4.

AWS gives each node subnet its own route table. Its default IPv4 route targets
that subnet's AWS NAT Gateway in Cloud Managed mode. In Podplane Managed mode,
it targets the active NAT VM's primary ENI. When fixed egress is configured,
Nstance reassociates the Elastic IP with that ENI before updating the route.
These are separate AWS operations; interruption between them is recovered by
idempotent reconciliation and may temporarily interrupt IPv4 egress.

Google Cloud has no subnet-associated route tables. In Podplane Managed mode,
Nstance assigns a subnet-specific network tag to each node and maintains a
tagged default route to the selected forwarding-enabled VM. A configured
reserved external IPv4 moves to the replacement before Nstance updates the
route; these are coordinated provider operations rather than one movable NIC.
In Cloud Managed mode, OpenTofu/Terraform configures Cloud NAT for the
node-subnet ranges instead.

OpenTofu/Terraform creates the route tables, Cloud Managed NAT resources, IAM
permissions, and other static network configuration. While Podplane Managed NAT
is configured, Nstance owns only the tenant subnet's translation destination:
`0.0.0.0/0` for IPv4 NAT or `64:ff9b::/96` for NAT64. AWS requires a
cluster-owned subnet route table; Google Cloud requires the tenant subnet's
named route to have the matching ownership description and destination.
Nstance may correct that destination's target even if it was changed manually.
It does not require the current next hop to belong to a known NAT VM, persist
previous ENI IDs, or rediscover a deleted VM's interface. Routes to peered VPCs,
native IPv6 `::/0`, and other unrelated destinations remain untouched.

Changing modes works as follows:

- **Cloud Managed to Podplane Managed:** OpenTofu/Terraform disables Cloud Managed NAT and updates Nstance configuration in one apply. Nstance then establishes the configured NAT next hops and installs the Podplane Managed routes. IPv4 egress is temporarily unavailable between the OpenTofu/Terraform change and Nstance reconciliation.
- **Podplane Managed to Cloud Managed:** OpenTofu/Terraform creates and validates Cloud Managed NAT. Stop Podplane Managed NAT reconciliation through Nstance configuration before switching node-subnet routing to the cloud-managed path; the old next hop is not an ownership guard. Nstance terminates its tracked NAT VMs, and cleanup leaves provider NAT Gateway routes intact.

Nstance-server must retain provider API and object-storage access during the
first transition. AWS uses its S3 gateway endpoint and direct public egress from
the public service subnet; private interface endpoints are an explicit opt-in.
Google Cloud uses Private Google Access, which is already enabled on managed
subnets. Storage/service endpoints do not provide general internet access.

Native IPv6 retains its existing provider-specific routing and needs no NAT
public address. The `nat` VM may additionally provide stateful NAT64 for
explicitly allowed IPv6 source ranges; its translated IPv4 traffic may use an
optional fixed public IPv4 in the same way as NAT44 traffic.

## Nstance NAT group

Each shard's Nstance configuration identifies at most one NAT group per tenant through a top-level `nat` map:

```jsonc
"groups": {
  "default": {
    "nat": {
      "template": "nat",
      "instance_type": "t4g.small",
      "subnet_pool": "nat"
    }
  }
},
"nat": {
  "default": {
    "group": "nat",
    "public_addresses": [
      {"ipv4": "192.0.2.10", "allocation_id": "eipalloc-0123456789abcdef0"}
    ],
    "last_instance_grace_period": "10m",
    "instance_type_ladder": ["t4g.small", "t4g.medium", "t4g.large"],
    "scale_up_thresholds": {
      "conntrack_percent": 80,
      "cpu_percent": 80,
      "packet_drops_per_second": 1
    },
    "scale_down_thresholds": {
      "conntrack_percent": 30,
      "cpu_percent": 30,
      "packet_drops_per_second": 0
    },
    "scale_up_window": "3m",
    "scale_down_window": "25m",
    "cooldown": "10m",
    "replacement_timeout": "10m"
  }
}
```

The map key is the tenant. Required `group` references the NAT group under `groups[tenant]`. A missing tenant entry disables Podplane Managed NAT for that tenant. Invalid references are configuration errors. The singular `group` reference makes multiple NAT groups for one tenant unrepresentable.

`nat.<tenant>` owns optional `public_addresses`, `last_instance_grace_period`,
`instance_type_ladder`, scale-up and scale-down thresholds and windows,
`cooldown`, and `replacement_timeout`. The referenced group owns only
deployment settings such as its template, public service `subnet_pool`, and
starting `instance_type`. The ladder is ordered from lowest to highest capacity
and must contain that starting instance type. AWS fixed-address entries contain
`ipv4` and `allocation_id`; Google Cloud entries contain only
`ipv4`. Entries are IPv4-only because native IPv6 is not translated.

`last_instance_grace_period` defaults to ten minutes and must be positive. Fixed
addresses are optional, must be valid IPv4 addresses, and cannot be shared by
tenants. The ladder must be nonempty and cannot contain empty or duplicate
instance types. Omitted threshold objects receive their complete defaults; a
supplied object must specify valid values for every threshold. CPU and conntrack
percentages must be greater than zero and no more than 100, with each scale-down
threshold below its scale-up threshold. Packet-drop thresholds must be
non-negative, with the scale-down threshold below the scale-up threshold.

When configured, the referenced group has one derived instance per populated tenant node subnet, rather than a fixed desired count. It is excluded from MachinePool import and replica distribution. Each derived member stores its subnet, instance type, route state, and lifecycle. Group `size` is rejected; `instance_type` is the starting/default size for each derived member.

When placing a node, Nstance must:

1. select its subnet;
2. ensure that subnet's NAT next hop is healthy;
3. ensure the subnet route targets it;
4. only then create the node.

Groups may span multiple subnets, so this dependency is evaluated from actual instance placement, not group configuration. Subnets using tenant-owned NAT cannot be shared with another tenant's NAT group because their routes would conflict.

When a tenant goes to sleep, Nstance keeps each NAT instance running until all
workloads that depend on it have been removed. It then removes the NAT instance
on the next reconciliation without waiting for `last_instance_grace_period`.
This applies to both normal and forced sleep. While the tenant is awake, the
configured grace period still applies after the final dependent node leaves,
avoiding unnecessary NAT replacement during brief gaps between workloads. Fixed
public IPs remain provisioned for reuse. Reconciling the shard back to awake
restores NAT before its dependent nodes.

Nstance-server owns route mutation while Podplane Managed NAT is active. Provider permissions must be limited to the cluster's route and NAT resources to keep that security boundary narrow.

## NAT VM configuration

`vmconfig` adds a `nat` kind for both supported architectures. Podplane `cluster create` selects its pinned manifest when Podplane Managed NAT is configured and passes the rendered userdata through the generated Nstance template.

The kind configures:

- IP forwarding and source NAT;
- optional stateful Jool NAT64, defaulting to the well-known `64:ff9b::/96` prefix;
- Nstance agent health/config reporting;
- conntrack limits and metrics;
- current CPU and memory metrics;
- cumulative byte, packet, and drop counters for one explicitly configured network interface;
- Fluent Bit and the standard vmconfig receive/reconfigure workflow.

NAT64 requires an IPv6 source allowlist and an IPv4 pool address. Translation to non-global IPv4 destinations is blocked by default. A custom `/96` prefix must provide a matching blocked-prefix list. Jool userspace comes from Debian, while signed modules are pinned to and verified against the exact Debian guest kernel before forwarding starts. The `nat` image contains no containerd, kubelet, container-runtime users, runtime images, configuration, or services.

The interface name is passed to nstance-agent as `NSTANCE_METRICS_INTERFACE`. Omitting it disables interface and conntrack collection. Each report contains one current metrics observation. Cumulative byte and packet counters remain available for observability and rate derivation, while nstance-server derives the packet-drop rate used for scaling. Collection failures are reported explicitly rather than represented as zero values.

## Vertical scaling

Each NAT instance scales vertically using A/B deployments. Nstance does not add parallel active NAT instances for the same subnet, as AWS does not support this.

Nstance-server evaluates:

- `nf_conntrack_count` against `nf_conntrack_max`;
- CPU utilization;
- packet drops per second derived from cumulative drop counters.

Scale up when any configured threshold remains exceeded for `scale_up_window`, which defaults to three minutes and must be between two and five minutes. Scale down only when all metrics remain at or below their configured thresholds for `scale_down_window`, which defaults to 25 minutes and must be between 20 and 30 minutes. `cooldown` defaults to ten minutes and cannot be shorter; `replacement_timeout` defaults to ten minutes and must be positive. Utilization thresholds default to 80% for scale-up and 30% for scale-down. Packet-drop thresholds default to one and zero drops per second respectively.

Byte and packet rates do not influence scaling because neither provider exposes a
portable, dependable capacity for every instance type. Network-limit scaling is
deferred until Nstance has a trustworthy provider-specific signal, such as AWS
ENA allowance-exceeded metrics, or maintained instance-capability metadata.

Replacement is create-before-destroy:

1. create the new instance at the selected size on its ordinary service-subnet interface;
2. wait for the Nstance agent to register and submit a successful health report;
3. move the subnet's optional fixed public IPv4 attachment and switch its route where required;
4. terminate the old instance.

Agent registration and a successful health report establish NAT-instance readiness. Forwarding readiness is not a separate reported metric. The instance type field on the group defines the default/starting point in its tenant's ordered `instance_type_ladder`.

Missing, stale, or failed health must not remove the existing translation route.
Nstance replaces a failed or externally deleted NAT VM for the same node subnet
and reuses its optional fixed IPv4, switching routing only after the replacement
is freshly healthy. If a replacement fails during cutover, Nstance restores the
previous NAT's route and fixed address only when that instance is freshly
healthy. Otherwise it leaves the route unchanged and records that routing needs
reconciliation before releasing the failed replacement's assignment. A later
fresh health report must allow reconciliation to restore the surviving NAT's
route and fixed address without creating another workload. This recovery intent
survives server restart and leadership changes; health observations must be
refreshed rather than inferred from durable assignment state. Retiring the old
NAT must never delete the healthy replacement's route. Routes are removed when
NAT is no longer needed, not simply because health is unavailable.

If every configured fixed address is assigned, Nstance keeps existing NAT VMs
unchanged and blocks creating NAT for another node subnet. It should prefer
populated subnets with remaining capacity before blocking node scale-up.
Replacement is unaffected because it reuses the address already assigned to
that subnet.

## Connection behavior

A route switch does not transfer conntrack state between NAT VMs, so vertical replacement resets established SNAT connections. This is an accepted tradeoff and must be documented as disruptive.

## Implementation areas

- **Nstance:** NAT-group configuration, subnet dependencies, provider route APIs, metrics, vertical replacement, tenant sleep/wake, and least-privilege permissions.
- **vmconfig:** `nat` kind.
- **Podplane CLI:** NAT mode/config validation, manifest selection, and generated OpenTofu/Terraform.
- **Nstance OpenTofu/Terraform modules:** stable subnet model plus provider-specific Cloud Managed and Podplane Managed NAT resources.

Tests must cover omitted and invalid NAT configuration, route changes, subnet
capacity, cross-tenant subnet conflicts, concurrent node creation, NAT-first
startup, last-node grace periods while awake, bypassing grace during sleep only
after the final dependent workload is removed, sleep/wake, failed NAT health, fixed-address
exhaustion, replacement with and without fixed IPv4, replacement cooldown, and
provider-specific cutover. Recovery tests must cover external VM deletion,
interrupted cutover, stale and missing predecessor health after restart, later
fresh health and retryable restoration failures without new workloads. Route
tests must verify correction of the owned translation destination and
preservation of peering, native IPv6, and cloud-managed NAT Gateway routes.
