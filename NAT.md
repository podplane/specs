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
- One generated Terraform variable selects `cloud-managed` or `podplane-managed` NAT. Reapplying changes Cloud Managed NAT resources, route configuration, Nstance configuration, and required NAT VMs; node-subnet definitions remain unchanged.
    - This means cluster operators can switch between Cloud Managed NAT and Podplane Managed NAT without any cluster node changes or rotations. Cloud Managed to Podplane Managed causes a temporary IPv4 egress outage while Nstance reconciles.

Podplane Managed NAT always places `nst` VMs in public service subnets on AWS and gives them external IPv4 addresses on Google Cloud. Inbound access remains denied except where explicitly required. Enabling Podplane Managed NAT may rotate Nstance-server VMs but not Kubernetes nodes.

The cluster administrator configures the maximum number of dedicated Podplane Managed NAT instances per zone/shard. Terraform pre-provisions that many stable NAT identities and passes their provider resource IDs to the zone's Nstance-server:

- AWS uses an ENI with an associated Elastic IP for each identity.
- Google Cloud uses a reserved internal IPv4 and regional external IPv4 pair for each identity.

Nstance claims an unassigned identity when it creates a NAT instance, tracks the assignment in its state, and releases it when the instance is removed. During vertical replacement, the replacement VM claims another available identity so both instances can overlap. The subnet's egress address changes within the administrator's stable, pre-allowlisted set.

AWS gives each node subnet its own route table. Its default IPv4 route targets that subnet's AWS NAT Gateway in Cloud Managed mode or the selected forwarding network interface in Podplane Managed mode.

Google Cloud has no subnet-associated route tables. In Podplane Managed mode, Nstance assigns a subnet-specific network tag to each node and maintains a tagged default route to the selected forwarding-enabled VM. In Cloud Managed mode, Terraform configures Cloud NAT for the node-subnet ranges instead.

Terraform creates the route tables, Cloud Managed NAT resources, IAM permissions, and other static network configuration. Nstance updates routes only while they point to Podplane Managed NAT next hops, because it knows which dedicated NAT VM is healthy.

Changing modes works as follows:

- **Cloud Managed to Podplane Managed:** Terraform disables Cloud Managed NAT and updates Nstance configuration in one apply. Nstance then establishes the configured NAT next hops and installs the Podplane Managed routes. IPv4 egress is temporarily unavailable between the Terraform change and Nstance reconciliation.
- **Podplane Managed to Cloud Managed:** Terraform creates and validates Cloud Managed NAT, then changes each node subnet to use it. Nstance subsequently observes that Podplane Managed NAT is no longer configured, terminates any tracked NAT VMs, and releases their identities.

Nstance-server must retain provider API and object-storage access during the first transition. AWS uses its S3 gateway endpoint and an EC2 interface endpoint; Google Cloud uses Private Google Access, which is already enabled on managed subnets. These paths do not provide general internet access.

Native IPv6 retains its existing provider-specific routing. The `nat` VM may additionally provide stateful NAT64 for explicitly allowed IPv6 source ranges; this does not replace native IPv6 routing.

## Nstance NAT group

Each shard's Nstance configuration identifies at most one NAT group per tenant through a top-level `nat` map:

```jsonc
"groups": {
  "default": {
    "nat": {
      "template": "nat",
      "instance_type": "t4g.small"
    }
  }
},
"nat": {
  "default": {
    "group": "nat",
    "network_identity_pool": "default-nat-identities",
    "last_node_grace_period": "10m",
    "instance_type_ladder": ["t4g.small", "t4g.medium", "t4g.large"],
    "scale_up_thresholds": {
      "throughput_percent": 80,
      "packets_per_second_percent": 80,
      "conntrack_percent": 80,
      "cpu_percent": 80,
      "packet_drops_per_second": 1
    },
    "scale_down_thresholds": {
      "throughput_percent": 30,
      "packets_per_second_percent": 30,
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

`nat.<tenant>` owns `network_identity_pool`, `last_node_grace_period`, `instance_type_ladder`, scale-up and scale-down thresholds and windows, `cooldown`, and `replacement_timeout`. The referenced group owns only deployment settings such as its template and starting `instance_type`. The ladder is ordered from lowest to highest capacity and must contain that starting instance type. `network_identity_pool` names the provider-neutral stable network identities available to dedicated NAT replacements: ENIs and their addresses on AWS, or reserved internal and external addresses on Google Cloud.

`last_node_grace_period` defaults to ten minutes and must be positive. `network_identity_pool` is required. The ladder must be nonempty and cannot contain empty or duplicate instance types. Omitted threshold objects receive their complete defaults; a supplied object must specify valid values for every threshold. Utilization percentages must be greater than zero and no more than 100, with each scale-down threshold below its scale-up threshold. Packet-drop thresholds must be non-negative, with the scale-down threshold below the scale-up threshold.

When configured, the referenced group has one derived instance per populated tenant node subnet, rather than a fixed desired count. It is excluded from MachinePool import and replica distribution. Each derived member stores its subnet, instance type, route state, and lifecycle. Group `size` is rejected; `instance_type` is the starting/default size for each derived member.

When placing a node, Nstance must:

1. select its subnet;
2. ensure that subnet's NAT next hop is healthy;
3. ensure the subnet route targets it;
4. only then create the node.

Groups may span multiple subnets, so this dependency is evaluated from actual instance placement, not group configuration. Subnets using tenant-owned NAT cannot be shared with another tenant's NAT group because their routes would conflict. When the final dependent node leaves a non-NAT subnet, Nstance removes its NAT instance after the tenant's configured grace period. Putting a tenant to sleep in a shard includes its local NAT group, but Nstance terminates nodes before their NAT dependencies. Reconciling that shard back to awake uses the reverse order.

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

The interface name is passed to nstance-agent as `NSTANCE_METRICS_INTERFACE`. Omitting it disables interface and conntrack collection. Each report contains one current metrics observation; the server derives throughput, packet, and drop rates from consecutive timestamped reports. Collection failures are reported explicitly rather than represented as zero values.

## Vertical scaling

Each NAT instance scales vertically using A/B deployments. Nstance does not add parallel active NAT instances for the same subnet, as AWS does not support this.

Nstance-server evaluates:

- throughput derived from cumulative byte counters against provider instance bandwidth;
- packets per second derived from cumulative packet counters against instance capability;
- `nf_conntrack_count` against `nf_conntrack_max`;
- CPU utilization;
- packet drops per second derived from cumulative drop counters.

Scale up when any configured threshold remains exceeded for `scale_up_window`, which defaults to three minutes and must be between two and five minutes. Scale down only when all metrics remain at or below their configured thresholds for `scale_down_window`, which defaults to 25 minutes and must be between 20 and 30 minutes. `cooldown` defaults to ten minutes and cannot be shorter; `replacement_timeout` defaults to ten minutes and must be positive. Utilization thresholds default to 80% for scale-up and 30% for scale-down. Packet-drop thresholds default to one and zero drops per second respectively.

Replacement is create-before-destroy:

1. claim an unused identity and create the new instance at the selected size;
2. wait for the Nstance agent to register and submit a successful health report;
3. switch the subnet route;
4. terminate the old instance and release its identity.

Agent registration and a successful health report establish NAT-instance readiness. Forwarding readiness is not a separate reported metric. The instance type field on the group defines the default/starting point in its tenant's ordered `instance_type_ladder`.

If no identity is available, Nstance keeps the current healthy NAT VM, reports an `identity_pool_exhausted` condition/metric, and retries later. It must not attempt a disruptive same-identity replacement. Pool exhaustion also blocks creating NAT for another subnet; Nstance should prefer populated subnets with remaining capacity before blocking node scale-up.

## Connection behavior

A route switch does not transfer conntrack state between NAT VMs, so vertical replacement resets established SNAT connections. This is an accepted tradeoff and must be documented as disruptive.

## Implementation areas

- **Nstance:** NAT-group configuration, subnet dependencies, provider route APIs, metrics, vertical replacement, tenant sleep/wake, and least-privilege permissions.
- **vmconfig:** `nat` kind.
- **Podplane CLI:** NAT mode/config validation, manifest selection, and generated Terraform.
- **Nstance Terraform modules:** stable subnet model plus provider-specific Cloud Managed and Podplane Managed NAT resources.

Tests must cover omitted and invalid NAT configuration, route changes, subnet capacity, cross-tenant subnet conflicts, concurrent node creation, NAT-first startup, last-node grace periods, sleep/wake, failed NAT health, identity-pool exhaustion, replacement cooldown, and provider-specific cutover.
