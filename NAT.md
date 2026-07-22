# Podplane NAT

## Goal

Podplane can use either Cloud Managed NAT or Podplane Managed NAT VMs without changing the cluster subnet model. AWS and Google Cloud use provider-native routing, but expose the same Podplane configuration and lifecycle.

## Network model

- Node IPv4 subnets are `/26`, providing 59 usable addresses on AWS and 60 on Google Cloud.
    - Nstance conservatively calculates usable capacity as the subnet size minus AWS's five reserved addresses on both providers. It fills each sorted subnet to 90% of that capacity before using the next, targeting 53 nodes for `/26`.
    - Once all configured subnets are at 90%, it should round-robin until it finds an available slot.
    - As you will learn, this will reduce the total number of required Podplane Managed NAT instances.
- Each populated node subnet has one NAT next hop. One configured subnet may optionally use the elected shard leader; other subnets use dedicated NAT VMs in a public/NAT service subnet in the same zone.
- One generated Terraform variable selects `cloud-managed` or `podplane-managed` NAT. Reapplying changes Cloud Managed NAT resources, route configuration, Nstance configuration, and required NAT VMs; node-subnet definitions remain unchanged.
    - This means cluster operators can switch between Cloud Managed NAT and Podplane Managed NAT without any cluster node changes or rotations. Cloud Managed to Podplane Managed causes a temporary IPv4 egress outage while Nstance reconciles.

Podplane Managed NAT always places `nst` VMs in public service subnets on AWS and gives them external IPv4 addresses on Google Cloud. Inbound access remains denied except where explicitly required. Enabling Podplane Managed NAT may rotate Nstance-server VMs but not Kubernetes nodes.

The cluster administrator configures the maximum number of dedicated Podplane Managed NAT instances per zone/shard. Terraform pre-provisions that many stable NAT identities and passes their provider resource IDs to the zone's Nstance-server:

- AWS uses an ENI with an associated Elastic IP for each identity.
- Google Cloud uses a reserved internal IPv4 and regional external IPv4 pair for each identity.

Nstance claims an unassigned identity when it creates a NAT instance, tracks the assignment in its state, and releases it when the instance is removed. During vertical replacement, the replacement VM claims another available identity so both instances can overlap. The subnet's egress address changes within the administrator's stable, pre-allowlisted set.

AWS gives each node subnet its own route table. Its default IPv4 route targets that subnet's AWS NAT Gateway in Cloud Managed mode or the selected forwarding network interface in Podplane Managed mode.

Google Cloud has no subnet-associated route tables. In Podplane Managed mode, Nstance assigns a subnet-specific network tag to each node and maintains a tagged default route to the selected forwarding-enabled VM. In Cloud Managed mode, Terraform configures Cloud NAT for the node-subnet ranges instead.

Terraform creates the route tables, Cloud Managed NAT resources, IAM permissions, and other static network configuration. Nstance updates routes only while they point to Podplane Managed NAT next hops, because it knows which shard leader or dedicated NAT VM is healthy.

Changing modes works as follows:

- **Cloud Managed to Podplane Managed:** Terraform disables Cloud Managed NAT and updates Nstance configuration in one apply. Nstance then establishes the configured small-cluster or dedicated NAT next hops and installs the Podplane Managed routes. IPv4 egress is temporarily unavailable between the Terraform change and Nstance reconciliation.
- **Podplane Managed to Cloud Managed:** Terraform creates and validates Cloud Managed NAT, then changes each node subnet to use it. Nstance subsequently observes that Podplane Managed NAT is no longer configured, disables small-cluster NAT, and terminates any tracked dedicated NAT VMs and releases their identities.

Nstance-server must retain provider API and object-storage access during the first transition. AWS uses its S3 gateway endpoint and an EC2 interface endpoint; Google Cloud uses Private Google Access, which is already enabled on managed subnets. These paths do not provide general internet access.

IPv6 is not translated and retains its existing provider-specific routing.

## Nstance NAT group

Each shard's Nstance configuration identifies at most one NAT group per tenant through a top-level `nat` map:

```jsonc
"groups": {
  "default": {
    "nat": {
      // NAT template and configuration
    }
  }
},
"nat": {
  "default": {
    "group": "nat",
    "small_cluster": {
      "initial_subnet": "nodes-0",
      "max_instances": 10
    }
  }
}
```

The map key is the tenant. Optional `group` references the dedicated NAT group under `groups[tenant]`; `small_cluster` configures small-cluster NAT. A missing tenant entry disables Podplane Managed NAT for that tenant. At least one mechanism must be configured, and invalid references are configuration errors. The singular `group` reference makes multiple dedicated NAT groups for one tenant unrepresentable. Tenant-level options such as last-node grace period and identity-pool reference belong in this object; template, starting instance type, and vertical-scaling policy remain on the referenced group.

When configured, the referenced group has one derived instance per populated tenant node subnet not served by small-cluster NAT, rather than a fixed desired count. It is excluded from MachinePool import and replica distribution. Each derived member stores its subnet, instance type, route state, and lifecycle. Group `size` is rejected; `instance_type` is the starting/default size for each derived member.

When placing a node, Nstance must:

1. select its subnet;
2. ensure that subnet's shard-leader or dedicated NAT next hop is healthy;
3. ensure the subnet route targets it;
4. only then create the node.

Groups may span multiple subnets, so this dependency is evaluated from actual instance placement, not group configuration. Subnets using tenant-owned NAT cannot be shared with another tenant's NAT group because their routes would conflict. When the final dependent node leaves a non-NAT subnet, Nstance removes its NAT instance after the tenant's configured grace period. Putting a tenant to sleep in a shard includes its local NAT group, but Nstance terminates nodes before their NAT dependencies. Reconciling that shard back to awake uses the reverse order.

Nstance-server owns route mutation while Podplane Managed NAT is active. Provider permissions must be limited to the cluster's route and NAT resources to keep that security boundary narrow.

## Small-cluster NAT

`small_cluster.max_instances > 0` enables small-cluster NAT. `initial_subnet` is reserved for this tenant's first instances and excluded from every other placement path. Nstance fills it first up to `max_instances`. If a dedicated `group` is configured, additional populated subnets use dedicated NAT VMs; otherwise placement blocks at the limit.

Every Nstance-server candidate in the shard is configured by the `nst` vmconfig kind for IP forwarding and source NAT. AWS disables source/destination checks; Google Cloud enables `can_ip_forward`. The elected leader uses its external IPv4 for egress. On leadership change, Nstance activates forwarding on the new leader and changes the node-subnet route before retiring the old path.

Small-cluster NAT has no stable egress-IP guarantee, automatic vertical scaling, or conntrack transfer. Leadership changes may change the public IPv4 and reset established connections. It is an optional cost optimization bounded by `max_instances`.

## NAT VM configuration

`vmconfig` adds a `nat` kind for both supported architectures. Podplane `cluster create` selects its pinned manifest when Podplane Managed NAT is configured and passes the rendered userdata through the generated Nstance template.

The kind configures:

- IP forwarding and source NAT;
- Nstance agent health/config reporting;
- conntrack limits and metrics;
- packet-drop, throughput, packet-rate, and CPU metrics;
- Fluent Bit and the standard vmconfig receive/reconfigure workflow.

## Vertical scaling

Each dedicated NAT instance scales vertically using A/B deployments; small-cluster NAT does not. Nstance does not add parallel NAT instances for the same subnet, as AWS does not support this.

Nstance-server evaluates:

- throughput against provider instance bandwidth;
- packets per second against instance capability;
- `nf_conntrack_count` against `nf_conntrack_max`;
- CPU utilization;
- packet drops.

Scale up when any configured threshold remains exceeded for a configurable two-to-five-minute window. Scale down only when all metrics remain below their configured thresholds for a configurable 20-to-30-minute window. After replacement, enforce a configurable ten-minute cooldown.

Replacement is create-before-destroy:

1. claim an unused identity and create the new instance at the selected size;
2. wait for Nstance agent and forwarding health;
3. switch the subnet route;
4. terminate the old instance and release its identity.

Instance-size ladders, thresholds, windows, cooldown, and replacement timeout are configurable. The instance-size ladder is essentially an ordered slice of instance types. The instance type field on the group defines the default/starting point.

If no identity is available, Nstance keeps the current healthy NAT VM, reports an `identity_pool_exhausted` condition/metric, and retries later. It must not attempt a disruptive same-identity replacement. Pool exhaustion also blocks creating NAT for another subnet; Nstance should prefer populated subnets with remaining capacity before blocking node scale-up.

## Connection behavior

A route switch does not transfer conntrack state between NAT VMs, so vertical replacement resets established SNAT connections. This is an accepted tradeoff and must be documented as disruptive.

## Implementation areas

- **Nstance:** NAT-group configuration, subnet dependencies, provider route APIs, metrics, vertical replacement, tenant sleep/wake, and least-privilege permissions.
- **vmconfig:** `nat` kind plus optional NAT forwarding in the `nst` kind.
- **Podplane CLI:** NAT mode/config validation, manifest selection, and generated Terraform.
- **Nstance Terraform modules:** stable subnet model plus provider-specific Cloud Managed and Podplane Managed NAT resources.

Tests must cover omitted and invalid NAT mechanisms, small-cluster-only and small-cluster-plus-group modes, leader election and route changes, initial-subnet reservation/capacity, cross-tenant subnet conflicts, concurrent node creation, NAT-first startup, last-node grace periods, sleep/wake, failed NAT health, identity-pool exhaustion, replacement cooldown, and provider-specific cutover.
