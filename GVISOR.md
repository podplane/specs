# gVisor by Default

> **STATUS**: In review

Related specification: [RBAC](./RBAC.md).

This specification targets fresh clusters, and the implementation does not need to
support backwards compatability/migrating existing Podplane clusters.

## Goal

Run tenant workloads in gVisor without requiring application configuration. Allow
administrators to grant a namespace permission to run selected workloads with
runc when compatibility or measured performance requires it.

Granting permission does not change the namespace's default runtime. A deployer
must explicitly select runc for each workload that needs it. A gVisor failure must
never trigger an automatic fallback to runc.

Podplane's runc permission is explicitly a trusted shared-host-kernel exception, 
not a grant of host-root privileges.

## Non-goals

- A runc default for tenant namespaces, or automatic runtime detection. The only
  runc default is the administrator-owned platform allowlist described in
  [Platform workloads and bootstrap](#platform-workloads-and-bootstrap).
- Per-application approval resources, expiring grants, or an approval controller.
- Changing Podplane's Pod Security levels or standard RBAC group bindings.
- Treating root inside a container as a reason to automatically select runc.
- Adding another sandbox technology or dedicated native-runtime node pools.
- Protecting the cluster from its own cluster administrators or node administrators.
- Existing-cluster migration, staged fleet activation, legacy-Pod inventory, or
  backward-compatible upgrade and rollback paths for this feature.

## Historical baseline

The following describes the pre-gVisor baseline, not the current implementation.
Kubernetes nodes now configure both runtimes and probe gVisor before starting
kubelet. Components now owns `platform-runtime`, and its bootstrap installs and
checks runtime admission before Cilium. An additional seed-based managed
runtime-admission readiness gate is deferred, not an MVP implementation gap;
runtime configuration and chart presence alone do not prove admission behavior.

- `podplane/vmconfig` configures containerd with runc as its default and only
  configured runtime. The shared container-runtime template is used by `knd`,
  `knc`, and `nst`; `nat` has no container runtime.
- `podplane/components` owns admission and cluster-wide RBAC resources through
  `charts/platform-rbac`, and component namespaces through
  `charts/platform-components`.
- The OIDC administrator group is `podplane:admins`.
- Unlabelled namespaces enforce baseline Pod Security, warn at baseline, and
  audit against restricted. A mutating policy fills omitted security settings
  with restricted defaults without overriding explicit settings.
- Platform namespaces have component-specific Pod Security levels. Some require
  host access. The VM-managed tunnel static Pod runs in `kube-system`, which is
  exempt from Pod Security Admission.
- Bootstrap installs Cilium, CoreDNS, and Flux before Flux reconciles
  `platform-rbac`. Changing the runtime default therefore affects workloads that
  currently start before platform admission policies exist.

Podplane must retain runc for approved workloads and host-integrated platform
services.

## Customer-facing contract

### Namespace permission

The namespace label is:

```yaml
metadata:
  labels:
    runtime.podplane.dev/runc: allowed
```

Only the exact string `allowed` grants permission. An absent label or any other
value does not grant permission. Removing the label is the documented way to
revoke permission.

Human administrators grant or revoke permission through `podplane:admins`.
Existing trusted cluster-administration identities must retain an administrative
path, including `system:masters` and deliberately authorized platform automation.
Namespace managers and workload service accounts do not receive this permission
merely because they can manage resources inside a namespace.

The entitlement label must be protected on namespace creation and updates,
including patches and server-side apply. Granting namespace write access for
unrelated purposes must not inadvertently delegate runtime-exception approval.

### Workload selection

Expose Kubernetes RuntimeClasses named `gvisor` and `runc`. Deployer-facing names
are stable even if the internal containerd handler for gVisor is named `runsc`.

| Namespace permission | Requested runtime | Outcome for a new tenant Pod |
| --- | --- | --- |
| Not granted | Omitted or `gvisor` | gVisor |
| Not granted | `runc` | Admission rejects the Pod |
| Granted | Omitted or `gvisor` | gVisor |
| Granted | `runc` | runc, subject to unchanged Pod Security checks |
| Either | Explicit empty string | Admission rejects the Pod |
| Either | Any other RuntimeClass | Admission rejects the Pod |

Omit `runtimeClassName` to use the default. Explicit `runtimeClassName: ""` is
rejected, not normalized or defaulted. Neither omission nor an empty string can
be used to obtain the node's native runtime. An invalid RuntimeClass name may be
rejected by Kubernetes before Podplane's runtime policy is evaluated.

Kubernetes selects the runtime for the entire Pod, including application, init,
and ephemeral containers. If any container requires runc, the Pod must explicitly
select runc in a namespace with permission to use it.
For a controller-managed workload, selection belongs in its Pod template.

Illustrative Deployment fragment:

```yaml
spec:
  template:
    spec:
      runtimeClassName: runc
```

Administrators also use explicit selection and namespace permission for their
tenant workloads. Being an administrator is not an implicit per-Pod bypass.

### Web and worker templates

In `github.com/podplane/templates`, both the `web` and `worker` charts must expose
a top-level `runtimeClassName` value defaulting to `gvisor` and render it into
`Deployment.spec.template.spec.runtimeClassName`. Update each chart's values,
values schema, documentation, and rendering tests together. Accept `gvisor` and
`runc`; reject empty or unsupported values during chart validation.

Deployers can explicitly select runc with `--set runtimeClassName=runc`, but the
chart setting does not grant namespace permission. Admission remains responsible
for authorization and for defaulting workloads submitted outside these charts.

These templates target managed installations with RuntimeClasses installed. They
must not silently omit their runtime selection or fall back to runc when deployed
into a bare environment without those resources.

Template behavior must work under gVisor, which does not deliver inotify events
for files changed outside the sandbox (see
[gVisor compatibility](#gvisor-compatibility)). Certificate and configuration
reloads must not wait for inotify events on projected volumes. Under gVisor, the
`web` chart's Envoy sidecar adopts rotated serving certificates through a
reloader sidecar that polls the projected certificate and, when it changes,
renames a marker file inside the sandbox into a Pod-shared volume that Envoy
watches. The reloader never copies certificate or key material. Under runc,
Envoy watches the projected certificate directly and the chart omits the
reloader.

### Grant scope

Permission applies to everyone who can create workloads in that namespace, not
one application or the person who requested the exception. Workload names,
labels, and service accounts controlled by those deployers are not independent
approval boundaries.

Podplane's standard non-admin groups normally have access across tenant
namespaces. Recommend a dedicated exception namespace with
`rbac.podplane.dev/restricted: "true"` and explicit RoleBindings when the exception
should be available only to selected users or automation. That RBAC label is
independent of the Pod Security level named `restricted`.

Illustrative namespace for a narrowly delegated exception:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: native-workloads
  labels:
    runtime.podplane.dev/runc: allowed
    rbac.podplane.dev/restricted: "true"
```

This grants no non-admin access by itself. An administrator must also create the
intended RoleBindings.

### Revocation and runtime changes

Removing permission blocks subsequent runc Pod creations once admission observes
the updated namespace. It does not evict existing Pods or change their runtime.
In-flight admission requests may race with a namespace update; revocation is not
an atomic termination mechanism.

Replicas created by scaling, rescheduling, Jobs, or controller replacement are new
Pods and must pass the current policy. They may be rejected after revocation even
when the workload object predates it. Restarting a container within an existing
Pod is not creation of a new Pod.

Do not block unrelated updates, status reporting, or deletion of existing Pods
solely because permission was revoked. Changing a workload's runtime requires
replacing its Pods; changing a namespace label does not migrate them.

### Errors and visibility

- A denied runc request must identify the namespace permission and explain that
  an administrator must grant it, or the workload must use gVisor.
- An unsupported runtime request must identify the supported RuntimeClass names.
- Controller-created Pod rejection must remain visible through Kubernetes events
  and workload conditions. Early validation of workload templates may improve
  feedback, but cannot replace enforcement on actual Pods.
- Operators must be able to identify the selected runtime for running workloads
  and list namespaces with runc permission. Admission must make tenant runtime
  selection explicit on admitted Pods.
- gVisor startup or compatibility errors must remain errors, not cause a retry
  under runc. Runtime changes and permission changes should be observable through
  the cluster's configured Kubernetes audit logging.

## Pod Security remains independent

Keep the existing baseline enforcement and restricted defaults. The runc
permission must not disable security-context defaulting or relax Pod Security.

- `runAsNonRoot: false` can be used with gVisor where Pod Security permits it.
- runc does not imply `privileged: true`, host namespaces, hostPath mounts, extra
  capabilities, or privilege escalation.
- Host-access workloads need both an appropriate runtime and a separately
  authorized Pod Security configuration.
- Setting a namespace's Pod Security level to `privileged` does not itself grant
  runc permission or change its runtime default.
- Do not add a Pod Security RuntimeClass exemption for either tenant runtime.

Admission acceptance does not prove identical enforcement of every seccomp or
Linux security feature across runtimes. Verify the actual gVisor version against
Podplane's security defaults and document material differences.

## Enforcement and trust boundaries

RuntimeClass selection is not protected by a Kubernetes RBAC `use` permission.
Admission must enforce the namespace entitlement and runtime allowlist; restricting
writes to RuntimeClass objects alone is insufficient.

Use the native MutatingAdmissionPolicy and ValidatingAdmissionPolicy mechanisms
already used by `platform-rbac`, in a separate `platform-runtime` chart that owns
the RuntimeClasses and runtime policies. Policies must fail closed for evaluation errors
and must be installed with enforcing bindings. RuntimeClass definitions, runtime
configuration, admission policies, and their bindings remain administrator-owned.

Enforce policy on actual Pod creation regardless of the submitting identity.
Deployments and other controllers submit Pods using their own credentials, not
the original human's OIDC groups. Do not exempt controllers or infer approval from
the Pod's `serviceAccountName`.

Defaulting and validation must compose so that an omitted or explicitly empty
runtime cannot reach runc accidentally. Reject alternate RuntimeClass names even
if an administrator has defined another class backed by runc. A policy change is
required to expand the supported tenant runtime set.

Tenants must not gain direct access to containerd, node files, static-Pod
manifests, or the platform's privileged deployment automation. Those interfaces
can bypass API admission and remain outside the tenant security boundary.

The platform runc default allowlist is part of the administrator-owned policy
configuration. It may name only `platform-*` namespaces, is never derived from the
runc permission label, and does not authorize runc: Pods it defaults to runc are
still rejected unless their namespace has permission.

## Runtime defaulting architecture

Containerd retains runc as its low-level default. vmconfig installs both runtime
handlers on `knd` and `knc` Kubernetes nodes, but not on `nst` or `nat` instances.
Trusted VM-managed static Pods continue to use the node default without depending
on RuntimeClass objects or API admission.

For API-created workloads, a single MutatingAdmissionPolicy sets an omitted
`spec.runtimeClassName`: to `runc` in namespaces on the platform runc default
allowlist, and to `gvisor` everywhere else. One policy makes both choices so two
defaulting policies cannot race for the same field. A ValidatingAdmissionPolicy
checks the final selection: `gvisor`, or `runc` with namespace permission. It
rejects a missing or empty final selection, so a defaulting failure cannot
silently reach containerd's runc default.

Policy presence is part of the managed platform's security guarantee. Removing
enforcement is an administrative security change, not a supported runtime
selection mechanism. Platform workloads follow the same namespace-permission
model, with the platform allowlist and the static-Pod mirror exception below.

### Kubernetes admission behavior

Research against Podplane's pinned Kubernetes v1.37.1 establishes the following:

1. RuntimeClass mutating admission runs before MutatingAdmissionPolicy. For an
   omitted runtime field, it initially does nothing.
2. A mutating policy that adds `gvisor` requests admission-chain reinvocation.
   RuntimeClass runs again, resolves the class, applies its scheduling constraints
   and overhead, and rejects conflicting node selectors.
3. RuntimeClass validation resolves the final class and validates its overhead.
   Podplane's separate validating policy enforces runtime authorization.
4. An explicit empty string causes RuntimeClass lookup to fail before the
   mutating policy runs. Podplane accepts this rejection; it does not patch
   Kubernetes or add normalization machinery to support empty strings.

Source evidence:

- [Admission plugin ordering, v1.37.1](https://github.com/kubernetes/kubernetes/blob/v1.37.1/pkg/kubeapiserver/options/plugins.go)
- [RuntimeClass admission and validation, v1.37.1](https://github.com/kubernetes/kubernetes/blob/v1.37.1/plugin/pkg/admission/runtimeclass/admission.go)
- [Mutating policy requests reinvocation, v1.37.1](https://github.com/kubernetes/kubernetes/blob/v1.37.1/staging/src/k8s.io/apiserver/pkg/admission/plugin/policy/mutating/dispatcher.go)
- [Admission-chain reinvocation, v1.37.1](https://github.com/kubernetes/kubernetes/blob/v1.37.1/staging/src/k8s.io/apiserver/pkg/admission/reinvocation.go)

The architecture decision is settled by this behavior. Implementation tests
verify correct policy wiring, not whether native admission can support the design.

## Platform workloads and bootstrap

The tenant contract does not require every host-integrated platform process to
run inside gVisor. A platform workload uses runc only when it needs host
integration gVisor cannot provide, as established in
[gVisor compatibility](#gvisor-compatibility): the host network namespace, host
Unix sockets in either direction, mount propagation, or host devices and kernel
interfaces such as `/dev/kmsg` and BPF. A privileged Pod Security level is not
itself a reason: host ports and read-only host files work under gVisor.

| Component | Host integration | Runtime |
| --- | --- | --- |
| Cilium agent and operator | Host networking, BPF, bidirectional mount propagation | runc |
| AWS EBS CSI driver node plugin | Bidirectional mount propagation, kubelet plugin sockets, devices | runc |
| AWS EBS CSI driver controller | None, but its containers share a socket at a mount path ending in `/` (see below) | runc until the chart or gVisor is fixed |
| Secrets Store CSI driver | Bidirectional mount propagation, kubelet plugin and provider sockets | runc |
| Secrets Store providers (AWS, Google, OpenBao, Vault) | Serve a Unix socket to the CSI driver from a host directory | runc |
| Node Problem Detector | `/dev/kmsg`, the containerd socket | runc |
| CoreDNS | Read-only host `resolv.conf` | gVisor |
| Envoy Gateway proxies | Host ports, polled read-only host certificates, a sidecar socket shared through an emptyDir with a gVisor mount hint | gVisor |
| Other platform components | None | gVisor |

Upstream charts for the runc components do not expose a RuntimeClass setting, so
`platform-runtime` holds a platform runc default allowlist naming their namespaces,
and those namespaces receive `runtime.podplane.dev/runc: allowed` through trusted
platform configuration. Both are required: the allowlist selects runc and the
label permits it. Components keep the two lists identical. Every Pod in an
allowlisted namespace defaults to runc, so a namespace is allowlisted only when
all of its Pods need runc. Charts may instead select a RuntimeClass explicitly in
their Pod templates. Do not post-render or patch upstream charts to select a runtime.
Neither namespace permission nor a privileged Pod Security level changes the
default for any namespace outside the allowlist.

Preserve existing restrictions on creating and accessing `kube-*` and
`platform-*` namespaces. Trusted platform automation may manage platform namespace
permissions without delegating that authority to tenants.

Static-Pod mirror requests need a narrowly scoped exception from runtime
defaulting and validation so their API representation does not claim a runtime
different from the locally launched Pod. Restrict this handling to authenticated
trusted kubelets, the expected platform namespaces, and mirror-Pod metadata;
tenant-supplied metadata alone must never grant an exemption. Preserve Kubernetes
node authorization and NodeRestriction protections.

Fresh managed-cluster bootstrap must proceed in this order:

1. vmconfig installs both handlers, retaining the runc default for trusted static
   infrastructure. All Kubernetes nodes, including newly provisioned nodes, must
   have gVisor available before accepting workload scheduling.
2. Bootstrap creates required platform namespaces and their native-runtime
   permissions, then installs the `platform-runtime` RuntimeClasses and policies
   with enforcing bindings before starting API-managed platform workloads.
   `platform-rbac` continues to be installed by Flux.
3. Platform workloads start in their defaulted runtimes: runc in allowlisted
   namespaces and gVisor elsewhere. Bootstrap must not depend on Cilium or Flux to
   install policies that need to precede them. Flux subsequently reconciles the
   same platform policy resources rather than a second policy implementation.
4. Runtime defaulting and enforcement must be active before standard tenant
   access. An additional managed runtime-admission readiness gate is deferred;
   vmconfig already probes gVisor before starting kubelet.

No temporary tenant bypass, automatic mass restart, or legacy migration phase is
required for fresh-cluster bootstrap.

## gVisor compatibility

Podplane's pinned gVisor release `release-20261005.0`, using systrap on
Kubernetes 1.37 with containerd and the systemd cgroup driver, behaves as follows
for Pods running in the `gvisor` RuntimeClass:

| Capability | Behavior |
| --- | --- |
| Host networking (`hostNetwork`) | Does not work; listeners are unreachable, including over loopback. |
| Host ports (`hostPort`) | Work; the host forwards to the Pod network. |
| Reading host paths, ConfigMaps, Secrets, and projected volumes | Reads observe changes made outside the sandbox, including atomic replacement. |
| inotify for changes made outside the sandbox | No events are delivered. |
| Host Unix sockets | Unavailable in both directions: sockets bound in the sandbox are not created on the host, and host sockets cannot be connected to. |
| Unix sockets on an emptyDir shared by containers | Fail by default because each container sees a separate view of the volume; work with gVisor mount hints. |
| Shared mounts at a path ending in `/` | Fail: the second container cannot be created (`failed to create directory mountpoint ...: file exists`). |
| Bidirectional mount propagation | Refused at container creation. |
| `/dev/kmsg` | Unreadable, even for privileged containers. |
| Pod resource limits | Kubelet's Pod cgroup limits apply to the sandbox, including memory OOM kills. |

vmconfig's containerd configuration forwards Pod annotations matching
`dev.gvisor.spec.mount.*` to runsc. A Pod shares an emptyDir across its
containers with `dev.gvisor.spec.mount.<volume>.share: pod` and
`dev.gvisor.spec.mount.<volume>.type: tmpfs`. The shim resolves hints against the
Pod's own volumes, so a hint cannot add a mount or expose another host path.
Charts using hints must mount the shared volume at a path without a trailing
`/`. The AWS EBS CSI chart hardcodes `/var/lib/csi/sockets/pluginproxy/` for its
controller's shared socket volume, so the controller stays in runc until the
chart drops the trailing slash or gVisor accepts it.
Placing the sandbox in the Pod's systemd slice does not require runsc's
experimental `systemd-cgroup` option, which Podplane does not set.

User documentation must describe these differences, including how to share
sidecar sockets and that file reloads must poll.

## Bare and managed installations

`--components=none` is an explicitly bare, admin-only development/bootstrap
environment, not a managed platform with the tenant runtime guarantee.

| Installation | Runtime behavior |
| --- | --- |
| `--components=none` | Both node runtimes installed; containerd defaults to runc. No platform-provided RuntimeClasses or runtime admission policies. |
| Minimal components | Full gVisor defaulting and runc-exception enforcement. |
| Platform/agentic/all components | The same enforcement, with additional services. |

Keep RuntimeClasses and runtime admission policies owned by components; do not
install a duplicate policy lifecycle through vmconfig. Minimal installations must
provide this security foundation without requiring `podplane-operator`.

Bare readiness means the VM and Kubernetes API are ready, not that managed
platform security policies are active. Documentation must state that bare
API-created Pods default to runc without platform runtime restrictions. Existing
API-server Pod Security defaults still apply.

Installing components on a bare environment establishes runtime enforcement
before enabling standard tenant access. It does not retroactively sandbox Pods
created during experimentation; recreate those workloads to acquire the gVisor
default. Administrators who manually grant tenant access before installing
components operate outside the managed security guarantee.

## Ownership and implementation direction

| Repository | Responsibility |
| --- | --- |
| `github.com/podplane/vmconfig` | Verified gVisor dependencies, installation, node runtime handlers including mount-hint forwarding, architecture support, static-Pod compatibility, and runtime health checks |
| `github.com/podplane/components` | RuntimeClasses, namespace permission protection, Pod runtime defaulting and validation, the platform runc default allowlist, and bootstrap ordering |
| `github.com/podplane/templates` | Explicit gVisor defaults and configurable runtime selection in the web and worker charts, including values schemas, documentation, rendering tests, and gVisor-compatible certificate reloading |
| `github.com/podplane/podplane` | User documentation and any CLI or readiness integration needed to expose the agreed contract |

No operator controller is required merely to authorize runtime selection.

Reuse Podmin's complete-payload installation approach through vmconfig's existing
dependency and verification machinery. Pin and verify artifacts for both amd64
and arm64; do not copy Podmin's installer or remove runc. Prefer gVisor's systrap
platform for VM-hosted nodes so nested virtualization is not a prerequisite.
Verify this choice on supported host architectures rather than assuming parity.

### VM-kind scope

| VM kind | Installed runtimes |
| --- | --- |
| `knd`, `knc` | runc and gVisor; containerd defaults to runc, managed API workloads default to gVisor through admission |
| `nst` | runc only; no gVisor binaries, sidecars, or handler configuration |
| `nat` | No container runtime |

`nst` is not a Kubernetes cluster worker. Its standalone kubelet runs with
`--register-node=false` and watches local static-Pod manifests. The Nstance server,
optional sleep proxy, and tunnel supervisor run as host services. The supervisor
generates tunnel static Pods whose images, arguments, and files depend on tunnel
configuration; these are not API-scheduled tenant workloads.

Tunnel Pods use host networking and read-only hostPath file mounts. Their
containers run non-root, with a read-only root filesystem, no privilege
escalation, and all capabilities dropped; they are not privileged containers.
Keep these trusted host-integrated Pods on runc. Excluding gVisor from `nst`
avoids an unused dependency and does not assert that every tunnel implementation
is incompatible with gVisor.

Scope gVisor dependency manifests, installation, and handler configuration to
`knd` and `knc`. Do not add it indiscriminately to the shared container-runtime
template or dependency kind set, which also covers `nst`.

## Implementation plan

1. Add verified gVisor packaging and both runtime handlers for vmconfig's `knd`
   and `knc` kinds, retaining the runc default; validate both architectures and
   preserve `nst` standalone/static-Pod behavior and its runc-only installation.
2. Add RuntimeClasses and fail-closed admission policy in components, including
   omitted-field defaulting, explicit-empty rejection, namespace-label protection,
   and the runtime allowlist.
3. Integrate early bootstrap installation, the platform runc default allowlist,
   and narrowly scoped static-Pod mirror handling. Verify a clean cluster reaches
   readiness without any temporary tenant bypass.
4. Add `runtimeClassName: gvisor` defaults and explicit runc overrides to the web
   and worker charts in `podplane/templates`, with schemas and rendering tests,
   and replace the `web` chart's inotify-based certificate reloading.
   Update Pod Security, RBAC, deployment, and troubleshooting documentation.
   Distinguish bare infrastructure readiness from managed platform readiness.
5. Verify the complete contract on disposable fresh clusters, including bare and
   minimal installations. Record evidence before marking this spec implemented.

Use each repository's existing checks: vmconfig `make lint`, `make test`, package
and approved install/integration checks; components `make lint`, `make render`,
`make validate`, and `make test`. Rendered YAML alone does not establish admission
or runtime behavior. Run the templates repository's chart checks for both web
and worker, exercising default and overridden runtime values.

## Acceptance criteria

- Omitted and explicit `gvisor` runtime selections actually execute under gVisor,
  including after node reboot. Admitted workload Pods carry an explicit runtime.
  Inspect runtime evidence, not only YAML.
- Explicit empty runtime selection is rejected. A missing final runtime after
  defaulting is rejected rather than reaching containerd's runc default.
- Defaulted RuntimeClass selection applies scheduling constraints and configured
  overhead through admission reinvocation; conflicting node selectors fail.
- Explicit runc is denied without permission and executes under runc with it.
  Permission does not change the runtime of workloads that omit selection.
- Both web and worker charts render `runtimeClassName: gvisor` by default and
  `runtimeClassName: runc` when explicitly overridden. Their schemas reject empty
  and unsupported runtime values. Deployed chart workloads remain subject to
  namespace permission checks.
- gVisor artifacts and handler configuration are present on `knd` and `knc`,
  absent from `nst`, and absent along with all container-runtime machinery from
  `nat`. Nstance tunnel static Pods continue to run under runc.
- Non-admin identities cannot grant themselves permission through namespace
  creation, update, patch, or server-side apply, even with separately delegated
  namespace metadata access. Unrelated permitted metadata edits still work.
- Administrators, authorized administrative automation, ordinary deployers, and
  controller-created Pods follow their respective contracts without blanket
  controller exemptions.
- An absent label or any value other than exact `allowed`, including `"true"`,
  `"false"`, and `Allowed`, does not grant permission.
  Unknown RuntimeClasses cannot bypass the runtime allowlist.
- Revocation leaves existing Pods manageable and running while rejecting new
  runc replicas. Regranting permission permits controller recovery.
- Existing Pod Security restrictions remain effective under both runtimes. A
  privileged namespace alone does not grant native runtime access; runc permission
  alone does not permit privileged Pods.
- Application, init, and ephemeral containers use the Pod's selected runtime.
- A missing or failing gVisor handler causes a visible failure with no runc
  fallback. Admission errors fail closed.
- Cilium networking, DNS, Flux, supported CSI/storage paths, ingress, and VM tunnel
  static Pods work from clean bootstrap and after restart, with each platform
  component in the runtime the platform workloads table assigns it.
- The platform runc default allowlist accepts only `platform-*` namespaces, and
  allowlisted Pods without namespace permission are rejected.
- Containers sharing a Unix socket through an emptyDir work under gVisor with
  mount hints, including the platform Envoy proxies and their SDS sidecar.
- The `web` chart's sidecar adopts rotated serving certificates under gVisor.
- Static-Pod mirrors preserve the locally selected runtime; tenant requests with
  forged mirror metadata cannot bypass runtime policy.
- Representative restricted-default applications work on amd64 and arm64,
  including network access, persistent storage, resource limits, logs, and shell
  operations allowed by existing RBAC. Document unsupported operations.
- A fresh minimal managed cluster enforces the full runtime contract without
  requiring `podplane-operator`. Additional managed runtime-admission readiness
  integration is deferred rather than an MVP acceptance gate.
- A bare cluster retains runc defaults and API-server Pod Security, with no
  platform runtime policy installed. Installing components activates enforcement
  without claiming to sandbox pre-existing experimental Pods.

## References

- [Kubernetes RuntimeClass](https://kubernetes.io/docs/concepts/containers/runtime-class/)
- [Kubernetes ValidatingAdmissionPolicy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)
- [Kubernetes Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [gVisor platform selection](https://gvisor.dev/docs/architecture_guide/platforms/)
- [gVisor compatibility](https://gvisor.dev/docs/user_guide/compatibility/)
