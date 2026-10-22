# Persistent Volumes

> **STATUS**: Ready for implementation

Related specifications: [GVISOR](./GVISOR.md), [REGISTRY](./REGISTRY.md)
(Google workload identity federation).

Targets fresh clusters only; no migration of existing clusters or volumes.

## Goal

Every Podplane cluster (AWS, Google Cloud, and the local VM) has exactly one
default StorageClass. A PVC that omits `storageClassName` therefore binds
everywhere without per-environment values. Source Gateway is the first platform consumer.

## Non-goals

- VolumeSnapshotClasses and snapshot support. These are future work; snapshot
  sidecars are omitted where possible.
- Hyperdisk-only Google machine families (C4, C4A, C4D, N4, N4A, N4D).
  Administrators add their own `hyperdisk-*` StorageClass.
- Regional, shared, block-mode, or encrypted-with-CMEK volumes.
- Kustomize. Components remains Helm-only.
- Installing packages with `apt-get`.

## Contract

| Environment | Component (`platform-<name>` namespace) | Driver | Default StorageClass | Key parameters |
|---|---|---|---|---|
| AWS | `csi-aws-ebs` (exists) | `ebs.csi.aws.com` | `platform-aws-ebs-default` (exists) | `encrypted: "true"`, `reclaimPolicy: Retain` |
| Google | `csi-gce-pd` (new) | `pd.csi.storage.gke.io` | `platform-gce-pd-default` | `type: pd-balanced`, `csi.storage.k8s.io/fstype: ext4`, `reclaimPolicy: Retain` |
| Local | `csi-openebs-lvm` + `csi-openebs-lvm-crds` (new) | `local.csi.openebs.io` | `platform-openebs-lvm-default` | `storage: lvm`, `volgroup: podplane`, `fsType: ext4`, `mountOptions: [discard]`, `reclaimPolicy: Delete` |

All three classes set `storageclass.kubernetes.io/is-default-class: "true"`,
`volumeBindingMode: WaitForFirstConsumer`, and `allowVolumeExpansion: true`.

Each driver namespace has these settings, matching `csi-aws-ebs`:
- `podSecurity: privileged`;
- `runcAllowed: true`;
- an entry in `platform-runtime`'s `runcDefaultNamespaces`. Node plugins need
  mount propagation, devices, and kubelet plugin sockets.

## AWS: EBS credentials

Today the driver has no credentials: the Nstance agent role only grants
`ec2:DescribeInstances`. Follow zot's kube2iam pattern
(`internal/tfgen/aws.go`, `internal/tfgen/runtime.go`):

- **IAM role:** Terraform creates `${name_prefix}-ebs-csi`. Its trust policy is
  `assume_from_knc`, and it has `arn:<partition>:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy`
  attached.
- **Controller Pods:** seed values set `aws-ebs-csi-driver.controller.podAnnotations`
  to `iam.amazonaws.com/role: <role ARN>`.
- **Namespace:** the `apps.csi-aws-ebs.namespaceAnnotations` entry sets
  `iam.amazonaws.com/allowed-roles` to `[<role ARN>]`.
- **Node plugin:** gets no role. It needs only instance metadata, which kube2iam
  passes through.

## Google: PD CSI driver

### Packaging

Upstream publishes no Helm chart. Its `deploy/kubernetes/base` is plain YAML,
and the `stable-master` overlay only sets images and adds two arguments.

`charts/csi-gce-pd` is a Podplane-owned chart. `templates/external/` is
generated from `base/` at the pinned tag. The generator extends `scripts/crds`
through its existing app-manifest path (`appTargetDir`, as used by
`cluster-api`) and supports `--check`.

**Pin:** v1.27.1, the newest tag published to registry.k8s.io. Use the driver
image of that same tag; upstream's overlay lags behind it.

**Generator transforms:**
1. Keep:
   - the controller Deployment and node DaemonSet;
   - ServiceAccounts and RBAC;
   - the leader-election Role;
   - the CSIDriver;
   - the taint-controller RBAC;
   - the PriorityClasses.
2. Drop:
   - `node_windows`;
   - the PSP-only `*-deploy` roles;
   - the `csi-snapshotter` container and its RBAC;
   - the `cloud-sa` volume, mounts, and `GOOGLE_APPLICATION_CREDENTIALS` in
     every container.
3. Replace namespace `gce-pd-csi-driver` with the release namespace.
4. Rename the ServiceAccounts to `platform-csi-gce-pd-controller` and
   `platform-csi-gce-pd-node`.
5. Template every image as `{{ .Values.images.<name>.repository }}:<tag>` so
   `_image-mirror-values.tpl` can mirror them.
6. Apply the overlay's attacher arguments (`--max-grpc-log-length=10000`,
   `--default-fstype=ext4`).
7. Remove the node DaemonSet's control-plane anti-affinity so that every
   schedulable node can mount volumes.

**Chart-owned templates:** the StorageClass, and the controller's
workload-identity wiring.

**Network and metadata** (verified against v1.27.1 source):
- **Controller:** the transform removes `hostNetwork`, so Cilium's
  `blockGoogleMetadata` applies to it. Its only metadata reads are the project
  and zone at startup, and both are supplied by
  `--cloud-config=/etc/gce/cloud.conf`. That is a chart-rendered ConfigMap
  with `[Global] project-id` and `zone` (any cluster zone; per-volume zones
  come from `WaitForFirstConsumer` topology). Never set `token-url`: it
  authenticates with a metadata token.
- **taint-controller:** makes no Google calls, so it gets no credentials and no
  metadata access.
- **Node plugin:** keeps `hostNetwork: true`. It reads `instance/zone`,
  `project/project-id`, `instance/name` and `instance/machine-type` once at
  startup and has no flag or config to supply them. Its node ID must name the
  real instance because the controller attaches disks to it. It makes no
  credentialed Google API calls.

### Credentials

The controller authenticates with the existing federation pattern
(`google-workload-identity.yaml`):
- value `platform.csiGcePd.googleWorkloadIdentity.provider`;
- a projected token with audience `https://iam.googleapis.com/<provider>`;
- an `external_account` credential config at `/var/run/google/identity`, used
  through `GOOGLE_APPLICATION_CREDENTIALS`.

Only the controller's `gce-pd-driver` container gets the credentials, with
`--fail-close-on-auth-error=true`. A file credential returns before the
metadata fallback, and metadata is unreachable from the controller anyway, so it
can never use the node service account. The node plugin needs no Google
credentials.

Terraform (`internal/tfgen/google.go`) grants these to
`principal://iam.googleapis.com/<pool>/subject/system:serviceaccount:platform-csi-gce-pd:platform-csi-gce-pd-controller`:
- `roles/compute.storageAdmin` on the project;
- a custom role `pp_<hash>_csi_gce_pd` with `compute.instances.get`,
  `compute.instances.attachDisk`, and `compute.instances.detachDisk`;
- `roles/iam.serviceAccountUser` on the Nstance agent service account
  (`agent_iam_role_arn`) only.

`seedValues` sets the provider value, project ID, and default zone.
`applyProviderComponents` enables `csi-gce-pd` for `google`. The manifest
generator's `componentMetadata` sets `Providers: ["google"]`.

## Local: OpenEBS LVM LocalPV

### Disk (`github.com/podplane/podplane`)

`internal/vm/qemu` creates `podplane-local-<id>.<arch>.data.qcow2` next to the
root image. It is a sparse qcow2 created with `qemu-img create -f qcow2`.
Every `local start` creates it if it is missing, not only first VM creation, so
VMs created before this change gain it. The root disk stays a fixed 128G.

`local start --data-disk-size SIZE` sets its size, in `qemu-img` size syntax
(`256G`, `1T`). The default is `256G`.

| Situation | Behavior |
|---|---|
| Disk missing | Create it at `SIZE`, or the default. |
| Flag omitted, disk exists | Leave it unchanged. |
| Flag larger than the disk | `qemu-img resize` before QEMU starts; the boot unit grows the PV. |
| Flag equal to the disk | No change. |
| Flag smaller than the disk | Error; shrinking is unsupported. Use `local stop --rm`. |
| Flag set while the VM is running | Error unless it equals the current size. |

"Flag set" means the user passed it (cobra `Changed`), so a plain `local start`
never resizes.

Both boot paths (direct kernel boot and the `-hda` firmware fallback) attach it
before cidata:

```text
-drive file=<data>,if=none,id=data0,format=qcow2,discard=unmap,detect-zeroes=unmap
-device virtio-blk-pci,drive=data0,serial=podplane-data
```

The guest sees `/dev/disk/by-id/virtio-podplane-data`. Device order (`vdb`) is
not part of the contract. `local stop` keeps the data disk, so PVs survive
restarts; `local stop --rm` deletes it.

### Host LVM (`github.com/podplane/vmconfig`)

Everything here is local-only, following kube2iam's AWS-only pattern. Cloud
and Proxmox VMs neither download the debs nor configure LVM.

**Packages:**
- Add trixie debs for knd/knc with `providers: ["local"]` through
  `scripts/manifests/sources.go`: `lvm2`, `dmeventd`, `liblvm2cmd2.03`, and
  `libdevmapper-event1.02.1` (epoch `2:`). `debianPackage` gains a providers
  option, and the manifest's existing per-provider `case "${PROVIDER_KIND}"`
  selection limits download and `dpkg -i` to `PROVIDER_KIND=local`. Debian's
  genericcloud image already ships their other dependencies.
- `lvm2` hard-depends on `dmeventd`, which depends on `liblvm2cmd2.03`, which
  depends back on `dmeventd`. `validateDependencyGraph` therefore permits this
  cycle. Cycles stay allowed only when every member is a deb installed in the
  single `dpkg -i` batch.
- `podplane local start` passes `local` in its deps-download providers, so the
  local server caches these debs.

**Host lvm2 is required.** The driver image bundles LVM, but the host provides
boot-time LV autoactivation (udev and `pvscan`), which the driver does not.

**Gating:** `features.sh` adds `lvm_enabled` (`PROVIDER_KIND = local`).
- `configure.sh` runs `lib/configure/lvm.sh` only when it is true. That script
  writes `/etc/modules-load.d/podplane-lvm.conf` (`dm_mod`), and the file is
  removed otherwise.
- `restart.sh` enables `podplane-lvm.service` only when it is true, and
  otherwise disables and stops it, as it does for kube2iam.

**Boot unit:** `podplane-lvm.service` is a knd oneshot:
- `ConditionPathExists=/dev/disk/by-id/virtio-podplane-data`;
- `Before=kubelet.service`;
- it runs on every boot, and every step is idempotent.

Its steps:
1. `modprobe dm_mod`. First boot installs vmconfig after `systemd-modules-load`
   has already run.
2. If the disk has a non-LVM signature (`blkid`), fail without writing.
3. `pvcreate` the disk unless it is already a PV.
4. `vgcreate podplane` unless the VG exists, so an interrupted first run
   completes on the next boot.
5. `pvresize` the disk, so a disk grown by `--data-disk-size` adds its new
   space to the VG (a no-op otherwise).
6. `vgchange -ay podplane`.

If the disk is absent, the condition skips the unit and PVCs stay Pending.

The driver creates and removes one thick (linear) LV per PVC in `podplane`.

Host LVM commands run only in this unit, before kubelet starts. They must not
race the driver's in-container LVM, which uses separate locks. Keep
`use_devicesfile = 0` (the default on both Debian and Alpine).

**Space reclaim:**
- The `discard` mount option passes space freed inside live volumes through
  dm-linear to the qcow2.
- Deleting a PVC frees its extents for reuse but does not shrink the qcow2.
  The driver runs `lvremove` with its own lvm.conf (`issue_discards = 0`).
- `local stop --rm` resets all space.
- Total requested PVC capacity cannot exceed the VG (no overprovisioning).

### Chart (`github.com/podplane/components`)

**`charts/csi-openebs-lvm`:**
- Helm dependency on the standalone `lvm-localpv` chart from
  `https://openebs.github.io/lvm-localpv`, pinned to 1.10.1. Never use a version
  range or the umbrella `openebs` chart.
- Values:
  - `analytics.enabled: false`;
  - the chart's snapshot controller and all of its CRDs disabled
    (`crds.enabled: false`);
  - `lvmPlugin.allowedTopologies` left at its default.
- The StorageClass follows the contract table.

**`charts/csi-openebs-lvm-crds`:** vendors the three `local.openebs.io/v1alpha1`
CRDs (`lvmnodes`, `lvmvolumes`, `lvmsnapshots`) with `renderHelmCRDs`. All three
are required: the controller needs `LVMNode` to start, and the node plugin
watches all three, including `LVMSnapshot`.

**Snapshot CRDs:** the chart always renders a `csi-snapshotter` sidecar (its
`snapshotter.enabled` value is ignored in v1.10.1). It needs the
`snapshot.storage.k8s.io` CRDs, or the controller Pod never becomes Ready.
`csi-openebs-lvm` therefore `dependsOn` the existing `snapshot-crds` chart,
which stays their only owner. This installs CRDs only: no snapshot controller
and no VolumeSnapshotClass.

**Wiring:**
- Add both components to `componentMetadata` with `Providers: ["local"]`.
  The same `local` deps-download provider fetches their images.
- The local seed overlay (`internal/local/netsy_seed.go`) enables both
  components and `snapshot-crds`.
- The components repo's `_local-bootstrap` enables all three for the
  `platform`, `agentic`, and `all` tiers.

## Implementation plan

1. **`github.com/podplane/components`:**
   - charts `csi-gce-pd`, `csi-openebs-lvm`, and `csi-openebs-lvm-crds`, plus
     the generator entry;
   - namespace entries, `runcDefaultNamespaces`, image-mirror blocks, and
     `componentMetadata`;
   - Source Gateway's `persistence.enabled` restored to the upstream default.
2. **`github.com/podplane/vmconfig`:** the local-only debs, the dependency-cycle
   rule, `lvm_enabled` gating, and `podplane-lvm.service`.
3. **`github.com/podplane/podplane`:**
   - the QEMU data disk, `--data-disk-size`, and the local seed and deps
     providers;
   - the AWS EBS role and values;
   - Google IAM, values, and enablement;
   - docs: components reference and pod-security guide.
4. Integrated validation, as below.

## Verification

**Chart tests:**
- each environment renders exactly one default class;
- driver namespaces match `runcDefaultNamespaces`;
- every image is mirrored;
- the `csi-gce-pd` render contains no `cloud-sa`, Windows node, or
  `csi-snapshotter`, and passes the existing workload-identity assertions;
- generator `--check` is clean.

**vmconfig:**
- `make dependencies` passes;
- an `aws` or `google` install downloads none of the LVM debs and leaves
  `podplane-lvm.service` disabled;
- running `podplane-lvm.service` twice changes nothing;
- the unit refuses a disk that carries a filesystem;
- a PV left without a VG completes on the next run.

**Local, end to end:**
- a PVC without `storageClassName` binds, and its data survives Pod rescheduling
  and a VM reboot;
- a VM created before this change gains the data disk on its next `local start`;
- `--data-disk-size` grows a stopped VM's disk and VG, rejects shrinking and
  changes to a running VM, and is ignored when omitted;
- the OpenEBS controller Pod is Ready with no snapshot controller installed;
- expansion works;
- deleting a PVC removes its LV;
- writing then deleting a file in a volume shrinks the qcow2;
- Source Gateway becomes Ready with a PVC.

**AWS:**
- provision, attach, expand, and delete work;
- the controller gets kube2iam credentials;
- the node plugin reads IMDSv2 through kube2iam.

**Google (release gate):**
- on `e2-medium`, provision, attach, expand, and delete work;
- Cloud Audit Logs show the federated principal;
- the controller runs without `hostNetwork` and cannot reach metadata;
- the controller fails at startup when its token is removed.

v1.27.x's `external_account` path is untested upstream. If it fails, block
`csi-gce-pd` rather than weakening the identity model.

## Future work

- VolumeSnapshotClasses for all three drivers, once snapshots ship outside the
  `all` tier.
- A Hyperdisk default for newer Google families.
- Narrowing `roles/compute.storageAdmin` to a custom role.
