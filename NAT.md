# Podplane NAT

## Goal

Podplane can use either Cloud Managed NAT or Podplane Managed NAT VMs without changing the cluster subnet model. AWS and Google Cloud use provider-native routing, but expose the same Podplane configuration and lifecycle.

## Network model

- Node IPv4 subnets are `/26`, providing 59 usable addresses on AWS and 60 on Google Cloud. 
    - Nstance's algorithm should aim to fill a sorted subnet list one-at-a-time, until that subnet reaches 90% of it's allocation, in other word it targets about 50-53 nodes per subnet. 
    - Once all configured subnets are at 90%, it should round-robin until it finds an available slot.
    - As you will learn, this will reduce the total number of required Podplane Managed NAT instances.
- Each populated node subnet has one dedicated NAT VM when Podplane Managed NAT is enabled.
    - A NAT VM runs in a public/NAT service subnet in the same zone as its node subnet.
- One generated Terraform variable selects `cloud-managed` or `podplane-managed` NAT. Reapplying changes Cloud Managed NAT resources, route configuration, Nstance configuration, and required NAT VMs; node-subnet definitions remain unchanged.
    - This means cluster operators can switch between Cloud Managed NAT and Podplane Managed NAT without any cluster node changes or rotations. Cloud Managed to Podplane Managed causes a temporary IPv4 egress outage while Nstance reconciles.

The cluster administrator configures the maximum number of Podplane Managed NAT instances per zone/shard. Terraform pre-provisions that many stable NAT identities and passes their provider resource IDs to the zone's Nstance-server:

- AWS uses an ENI with an associated Elastic IP for each identity.
- Google Cloud uses a reserved internal IPv4 and regional external IPv4 pair for each identity.

Nstance claims an unassigned identity when it creates a NAT instance, tracks the assignment in its state, and releases it when the instance is removed. During vertical replacement, the replacement VM claims another available identity so both instances can overlap. The subnet's egress address changes within the administrator's stable, pre-allowlisted set.

AWS gives each node subnet its own route table. Its default IPv4 route targets that subnet's AWS NAT Gateway in Cloud Managed mode or NAT VM network interface in Podplane Managed mode.

Google Cloud has no subnet-associated route tables. In Podplane Managed mode, Nstance assigns a subnet-specific network tag to each node and maintains a tagged default route to that subnet's forwarding-enabled NAT VM. In Cloud Managed mode, Terraform configures Cloud NAT for the node-subnet ranges instead.

Terraform creates the route tables, Cloud Managed NAT resources, IAM permissions, and other static network configuration. Nstance updates routes only while they point to Podplane Managed NAT VMs, because it knows which NAT instance is healthy.

Changing modes works as follows:

- **Cloud Managed to Podplane Managed:** Terraform disables Cloud Managed NAT and updates Nstance configuration in one apply. Nstance then claims identities, creates and health-checks NAT VMs, and installs the Podplane Managed routes. IPv4 egress is temporarily unavailable between the Terraform change and Nstance reconciliation.
- **Podplane Managed to Cloud Managed:** Terraform creates and validates Cloud Managed NAT, then changes each node subnet to use it. Nstance subsequently observes that Podplane Managed NAT is no longer configured but still has NAT instances in its state, so standard reconciliation terminates them and releases their identities.

Nstance-server must retain provider API and object-storage access during the first transition. AWS uses its S3 gateway endpoint and an EC2 interface endpoint; Google Cloud uses Private Google Access, which is already enabled on managed subnets. These paths do not provide general internet access.

IPv6 is not translated and retains its existing provider-specific routing.

## Nstance NAT group

NAT is a special Nstance group identified as the NAT group. It has one derived instance per populated node subnet, rather than a fixed desired count. It is excluded from MachinePool import and replica distribution. Each derived member stores its own subnet, instance type, route state, and lifecycle; ordinary group `size` and `instance_type` do not drive the members.

When placing a node, Nstance must:

1. select its subnet;
2. ensure that subnet's NAT instance exists and is healthy;
3. ensure the subnet route targets it;
4. only then create the node.

Groups may span multiple subnets, so this dependency is evaluated from actual instance placement, not only group configuration. When the final dependent node leaves a non-NAT subnet, Nstance removes its NAT instance after a configurable grace period. Suspension intent is recorded for all groups atomically, but Nstance terminates nodes before their NAT dependencies. Resume uses the reverse order.

Nstance-server owns route mutation while Podplane Managed NAT is active. Provider permissions must be limited to the cluster's route and NAT resources to keep that security boundary narrow.

## NAT VM configuration

`vmconfig` adds a `nat` kind for both supported architectures. Podplane `cluster create` selects its pinned manifest when Podplane Managed NAT is configured and passes the rendered userdata through the generated Nstance template.

The kind configures:

- IP forwarding and source NAT;
- Nstance agent health/config reporting;
- conntrack limits and metrics;
- packet-drop, throughput, packet-rate, and CPU metrics;
- Fluent Bit and the standard vmconfig receive/reconfigure workflow.

## Vertical scaling

Each subnet's NAT instance scales vertically using A/B deployments; Nstance does not add parallel NAT instances for the same subnet, as AWS does not support this.

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

- **Nstance:** NAT-group configuration, subnet dependencies, provider route APIs, metrics, vertical replacement, suspension, and least-privilege permissions.
- **vmconfig:** `nat` kind and forwarding/observability services.
- **Podplane CLI:** NAT mode/config validation, manifest selection, and generated Terraform.
- **Nstance Terraform modules:** stable subnet model plus provider-specific Cloud Managed and Podplane Managed NAT resources.

Tests must cover group placement across subnets, concurrent node creation, NAT-first startup, last-node grace periods, suspend/resume, failed NAT health, identity-pool exhaustion, replacement cooldown, and provider-specific cutover.
