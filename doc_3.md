# Multi-Disk VolumeGroupSnapshot Using Azure VM Restore Points

**Implementation design: crash-consistent and Azure application-consistent modes**  
**Status:** Proposed for review

## 1. Decision summary

One Kubernetes `VolumeGroupSnapshot` will be implemented as one Azure VM Restore Point.

```text
VolumeGroupSnapshot
└── VM Restore Point
    ├── Disk Restore Point for PVC A
    ├── Disk Restore Point for PVC B
    └── Disk Restore Point for PVC C
```

The feature exposes two modes through `VolumeGroupSnapshotClass`:

| Mode | Azure request | Filesystem/application coordination |
|---|---|---|
| Crash | Set `consistencyMode=CrashConsistent` | None |
| Application | Omit `consistencyMode` | Azure VMSnapshot extension |

### Direct answer about filesystem freeze

The crash-consistent path does **not** freeze filesystems. It captures persisted disk state as if the VM lost power.

The Azure application-consistent path performs guest coordination before capture:

- Windows: VMSnapshot invokes VSS writers.
- Linux: VMSnapshot invokes configured pre- and post-scripts.

The CSI driver does not run `fsfreeze` in either mode.

On Linux, Azure application consistency does not automatically make every containerized application consistent. The configured script must correctly flush and pause the workload. If the script only calls `sync` or freezes a filesystem, the result may be filesystem-consistent but not transactionally application-consistent.

The implementation is strict:

- A crash request succeeds only when Azure returns `CrashConsistent`.
- An application request succeeds only when Azure returns `ApplicationConsistent`.
- `FileSystemConsistent` is not reported as application-consistent.

### Recommendation

Ship crash mode first. Keep application mode opt-in until AKS node images, VM Agent/VMSnapshot behavior, Linux scripts or Windows VSS writers, and workload recovery are qualified.

---

## 2. Existing driver behavior

The current driver:

- Registers CSI Identity, Controller, and Node services.
- Implements individual `CreateSnapshot`, `DeleteSnapshot`, and `ListSnapshots`.
- Creates one `Microsoft.Compute/snapshots` resource per request.
- Restores a managed disk from that snapshot using `DiskCreateOptionCopy`.

The current driver does not:

- Register the CSI `GroupController` service.
- Implement the group create/get/delete RPCs.
- Create Restore Point Collections or Restore Points.
- Restore a disk from a Disk Restore Point.

The vendored dependencies already contain:

- CSI v1.11 group-controller messages and service definitions.
- Azure Compute SDK clients for Restore Point Collections, Restore Points, and Disk Restore Points.

The cloud-provider client factory does not expose the Restore Point clients, so the driver needs a small local client wrapper.

The deployed external-snapshotter is v8.6.0, but its group-snapshot feature gate, CRDs, and RBAC are not enabled.

---

## 3. Scope

### Included

- Same-VM `VolumeGroupSnapshot`.
- Crash and Azure-native application consistency selection.
- Restore Point create, get, and delete.
- One Disk Restore Point handle per source volume.
- Exact requested-disk validation.
- Restore of member PVCs from Disk Restore Points.
- Idempotency and restart recovery.
- Required CRDs, RBAC, Helm values, examples, and tests.

### Excluded

- Groups spanning VMs or nodes.
- CSI-owned application hooks.
- CSI node filesystem freeze/thaw.
- Hook or freeze CRDs.
- Cross-region Restore Point copy.
- Instant Access.
- Atomic group restore.

PVC-scoped application hooks or CSI-owned freeze require a separate Kubernetes coordinator. They are not added to this feature.

---

## 4. Consistency contract

### 4.1 Crash mode

Class parameter:

```yaml
parameters:
  consistencyMode: crash
```

Azure request:

```go
Properties: &armcompute.RestorePointProperties{
    ConsistencyMode: ptr.To(armcompute.ConsistencyModeTypesCrashConsistent),
    ExcludeDisks: excludedDisks,
}
```

Guarantee:

- One write-order-consistent capture across the included disks.
- Filesystems and applications recover as after sudden power loss.

Not guaranteed:

- Page-cache flush.
- Clean filesystem state.
- Completed application transactions.
- Application-specific logical consistency.

### 4.2 Application mode

Class parameter:

```yaml
parameters:
  consistencyMode: application
```

Azure request:

```go
Properties: &armcompute.RestorePointProperties{
    ConsistencyMode: nil,
    ExcludeDisks: excludedDisks,
}
```

Application consistency is requested by omitting the Azure field. The driver must not send an `ApplicationConsistent` literal as input.

Azure owns this sequence:

```text
Provision/check VMSnapshot extension
  -> Quiesce applications through VSS or Linux pre-script
  -> Capture included disks
  -> Resume applications through VSS or Linux post-script
```

The driver validates the returned `RestorePoint.Properties.ConsistencyMode`. Only `ApplicationConsistent` is accepted.

If Azure returns `FileSystemConsistent` or `CrashConsistent`, the driver:

1. Does not publish the group as ready.
2. Attempts to delete the unusable Restore Point.
3. Returns `FailedPrecondition` with the requested and achieved modes.

### 4.3 AKS limitation

Azure guest coordination is VM-scoped, not Pod- or PVC-scoped.

`excludeDisks` controls which disks are captured, but it does not limit VMSnapshot guest coordination to selected Pods. Unrelated workloads on the same node may be affected.

Application mode should initially be supported only on dedicated or qualified nodes. Linux scripts must be installed, secured, tested, and aware of the intended workloads.

The driver must never combine Azure VMSnapshot coordination with CSI-owned `fsfreeze`; that would create two freeze owners and ambiguous recovery.

---

## 5. Azure support constraints

The driver should perform checks it can determine from VM metadata and let Azure remain authoritative for the complete support matrix.

| Area | Crash mode | Application mode |
|---|---|---|
| Minimum documented frequency | One hour | Three hours |
| NVMe controller | Unsupported | Supported where listed by Azure |
| Azure Boost | Unsupported | Supported where listed by Azure |
| Premium SSD v2 and Ultra multi-disk | Public preview on qualifying SCSI VMs | Supported subject to Azure matrix |
| Guest OS requirement | No guest coordination | Supported OS required |
| VM Agent/VMSnapshot | Not used | Required |

Common restrictions:

- All selected disks must be attached to one VM.
- VMSS Uniform is unsupported.
- Shared disks are unsupported.
- Ephemeral OS disks must be excluded.
- Concurrent Restore Point creation for one VM is unsupported.
- Maximum 500 Restore Points per VM.
- Maximum 10,000 Restore Point Collections per subscription per region.

Azure error codes must be preserved in the user-facing failure message.

---

## 6. CSI and Azure resource mapping

| CSI value | Azure value |
|---|---|
| `group_snapshot_id` | VM Restore Point ARM ID |
| Member `snapshot_id` | Disk Restore Point ARM ID |
| Member `source_volume_id` | Managed disk ARM ID |
| Group creation time | Restore Point `TimeCreated` |
| Member size | Captured source metadata `DiskSizeGB` |

Every returned member `csi.Snapshot` sets `group_snapshot_id` to the parent Restore Point ID. This prevents the member from being treated as an independent snapshot.

The group is ready only when:

- The parent Restore Point succeeded.
- Azure returned the requested consistency mode.
- The captured disk set exactly matches the request.
- Every source disk has one Disk Restore Point.

---

## 7. Driver implementation

### 7.1 New files

#### `pkg/azuredisk/groupcontrollerserver.go`

Implement:

```text
GroupControllerGetCapabilities
CreateVolumeGroupSnapshot
GetVolumeGroupSnapshot
DeleteVolumeGroupSnapshot
```

#### `pkg/azuredisk/restorepoint.go`

Add a small driver-owned Azure wrapper for:

```text
Ensure Restore Point Collection
Create/Get/Delete Restore Point
List/Get Disk Restore Points
Get source VM
```

The wrapper owns Azure SDK pollers, per-subscription clients, and Azure error translation.

#### `pkg/azureutils/azure_restorepoint_utils.go`

Add pure helpers for:

- Restore Point ARM ID parsing.
- Disk Restore Point ARM ID parsing.
- Case-insensitive disk-set comparison.
- CSI group/member response construction.

### 7.2 Existing files

| File | Required change |
|---|---|
| `pkg/azuredisk/features.go` | Add alpha `VolumeGroupSnapshot` feature gate |
| `pkg/azuredisk/azuredisk.go` | Embed/register GroupController and initialize Restore Point client |
| `pkg/azuredisk/controllerserver.go` | Restore from Disk Restore Point and reject standalone member delete |
| `pkg/azuredisk/azure_controller_common.go` | Support `DiskCreateOptionRestore` |
| `pkg/azuredisk/fake_azuredisk.go` | Inject fake Restore Point service |

Do not modify vendored Azure or cloud-provider code.

---

## 8. Driver initialization

Add:

```text
VolumeGroupSnapshot=false
```

When disabled:

- Do not register GroupController.
- Existing snapshot behavior is unchanged.

When enabled:

- Register `csi.RegisterGroupControllerServer`.
- Return `CREATE_DELETE_GET_VOLUME_GROUP_SNAPSHOT` from `GroupControllerGetCapabilities`.

This capability is separate from `ControllerGetCapabilities`.

Construct Azure clients with the existing cloud credential, cloud endpoint, user agent, retry configuration, and additional compute client options. Cache one client bundle per subscription.

---

## 9. Naming and idempotency

### Restore Point Collection

Create one collection per source VM:

```text
azuredisk-csi-<hash of normalized VM ARM ID>
```

Place it in the source VM resource group and location. If it already exists, verify that its source VM ID matches.

The collection remains after individual group deletion so later groups for the same VM can reuse it.

### Restore Point

Encode the requested mode in the deterministic name:

```text
cc-<normalized CSI request name>
ac-<normalized CSI request name>
```

This is necessary because `GetVolumeGroupSnapshot` does not receive class parameters. After restart, the driver derives the requested mode from the controlled prefix and revalidates the Azure result.

Before creating one mode, check whether the opposite-mode name already exists. Reusing the same CSI request name with different parameters returns `AlreadyExists`.

---

## 10. CreateVolumeGroupSnapshot

### 10.1 Validate request

Return `InvalidArgument` for:

- Empty name.
- Empty volume list.
- Duplicate volume ID.
- Malformed or non-disk ARM ID.
- Unsupported `consistencyMode`.
- Unknown class parameter, except external-snapshotter metadata keys.

`consistencyMode` values are case-insensitive. Default is `crash`.

### 10.2 Resolve source VM

For each source volume:

1. Use the existing managed-disk client to read the disk.
2. Require `ManagedBy`.
3. Require every `ManagedBy` ID to match.
4. Retain the source disk size.

Different or detached VMs return `FailedPrecondition`.

Parse the common VM ID and read its storage profile.

### 10.3 Build the exact disk set

From the VM storage profile:

1. Confirm every requested disk is attached.
2. Exclude the OS disk.
3. Exclude every unselected data disk.
4. Re-read the VM immediately before capture.

If attachment membership changed, return `Aborted` and let external-snapshotter retry.

### 10.4 Serialize per VM

Use the existing `volumeLocks` with normalized VM ARM ID as the key.

Azure also rejects concurrent Restore Point creation for the same VM. Translate this conflict to `Aborted`, not `Internal`.

### 10.5 Create or observe

Use the deterministic collection and Restore Point names.

- Missing Restore Point: create it.
- Creating: return `ready_to_use=false`.
- Succeeded: validate and return it.
- Failed: return the Azure terminal failure.

Do not hold the gRPC request for the entire Azure operation. After Azure accepts and the resource is readable, return not ready and let `GetVolumeGroupSnapshot` reconcile it.

### 10.6 Validate result

After success:

1. Validate the returned consistency mode.
2. Read `SourceMetadata.StorageProfile.DataDisks`.
3. Compare captured managed-disk IDs with requested IDs.
4. Require exact equality.
5. Extract each Disk Restore Point ID.

Create returns members in `source_volume_ids` order.

If consistency or disk membership is wrong, do not publish the group. Attempt cleanup and return an explicit failure.

---

## 11. Get and delete

### GetVolumeGroupSnapshot

Validate:

- Parent Restore Point ID.
- Member Disk Restore Point IDs.
- Every member belongs to the parent.
- Mode prefix is `cc-` or `ac-`.

Reconstruct the response only from Azure:

- `Creating` returns not ready.
- `Succeeded` revalidates mode, exact disk set, and members.
- Azure 404 returns `NotFound`.
- Terminal failure returns `Internal`.

Get returns members in the request's `snapshot_ids` order.

### DeleteVolumeGroupSnapshot

Before deletion:

1. Get the parent.
2. List its Disk Restore Points.
3. Compare them with `snapshot_ids`.

A detectable mismatch returns `FailedPrecondition`.

Delete the parent Restore Point. Missing parent is success. Do not delete the shared collection.

---

## 12. Member restore

This change is required; otherwise group creation works but restored PVCs fail.

In the `CreateVolume` snapshot source path:

1. Parse the ARM resource type with `arm.ParseResourceID`.
2. Keep the current path for `Microsoft.Compute/snapshots`.
3. Detect nested Disk Restore Point IDs.
4. Read the child and parent Restore Point metadata.
5. Validate the child-to-source-disk mapping.

Extend the internal source type and `getValidCreationData`:

```go
case sourceDiskRestorePoint:
    return armcompute.CreationData{
        CreateOption:     ptr.To(armcompute.DiskCreateOptionRestore),
        SourceResourceID: &sourceResourceID,
    }, nil
```

The target StorageClass remains authoritative for target SKU, zone, encryption, and network settings.

Update `DeleteSnapshot`:

- Normal managed snapshot: existing behavior.
- Disk Restore Point: return `FailedPrecondition`; deletion must use `DeleteVolumeGroupSnapshot`.

---

## 13. Deployment

### Kubernetes resources

Add the upstream v8.6.0 CRDs:

```text
VolumeGroupSnapshot
VolumeGroupSnapshotContent
VolumeGroupSnapshotClass
```

Enable on both sidecars:

```text
--feature-gates=CSIVolumeGroupSnapshot=true
```

Enable on the Azure Disk driver:

```text
--feature-gates=VolumeGroupSnapshot=true
```

Add group-snapshot RBAC to:

- Snapshot-controller service account.
- CSI snapshotter service account.

### Helm

Add conditional values:

```yaml
snapshot:
  volumeGroupSnapshot:
    enabled: false
    classes:
      crash:
        enabled: false
        name: csi-azuredisk-vgsc-crash
        parameters:
          consistencyMode: crash
      application:
        enabled: false
        name: csi-azuredisk-vgsc-application
        parameters:
          consistencyMode: application
```

Application class remains disabled by default.

Update only `deploy/` and `charts/latest/`. Historical version directories are release artifacts.

### Azure permissions

The controller identity needs read access to source VMs/disks and create/read/delete access to:

```text
Microsoft.Compute/restorePointCollections
Microsoft.Compute/restorePointCollections/restorePoints
Microsoft.Compute/restorePointCollections/restorePoints/diskRestorePoints
```

Application mode may require additional VM extension permissions; confirm the exact Azure provider actions during qualification.

---

## 14. Error behavior

| Condition | gRPC code |
|---|---|
| Invalid field, ID, mode, or parameter | `InvalidArgument` |
| Detached or cross-VM disk | `FailedPrecondition` |
| Unsupported VM/disk/mode combination | `FailedPrecondition` |
| Application request achieved filesystem/crash consistency | `FailedPrecondition` |
| Same name with different source set or mode | `AlreadyExists` |
| VM operation already active or attachment changed | `Aborted` |
| Missing group in Get | `NotFound` |
| Member list does not match parent | `FailedPrecondition` |
| Azure throttling | Honor `Retry-After`; return retryable code |
| Temporary Azure transport failure | `Unavailable` |
| Azure terminal failure | `Internal` |

Do not convert every Azure error to `Internal`; sidecar retry behavior depends on the gRPC code.

---

## 15. Validation

### Unit tests

Cover:

- Request and mode parsing.
- Same-VM and cross-VM validation.
- OS and unrelated disk exclusion.
- Exact-set verification.
- Deterministic naming and retries.
- Crash field is explicit.
- Application field is omitted.
- Strict returned-mode validation.
- `FileSystemConsistent` rejection.
- Create/Get/Delete restart behavior.
- Disk Restore Point restore with `DiskCreateOptionRestore`.
- Standalone member deletion rejection.
- Existing individual snapshot behavior.

### Crash-mode end-to-end test

1. Attach two PVCs to one Pod.
2. Write ordered data to both.
3. Create a crash-mode group.
4. Restore both member PVCs.
5. Verify readable filesystems and a crash-recoverable common prefix.
6. Delete the group.

Add a cross-node negative test.

### Application-mode qualification

Run only on a dedicated supported node with a healthy VM Agent and VMSnapshot configuration.

1. Use a qualified application with two PVCs.
2. Configure audited Linux pre/post scripts or Windows VSS writers.
3. Create an application-mode group.
4. Require Azure result `ApplicationConsistent`.
5. Restore and run application-native integrity checks.
6. Inject script or VSS failure.
7. Verify `FileSystemConsistent` is not published as application success.

---

## 16. Rollout

### Phase 1

- Implement common GroupController and Restore Point lifecycle.
- Enable crash mode in private preview.
- Validate create, restore, delete, restart, and Azure limits.

### Phase 2

- Move crash mode to broader preview/GA after scale and recovery testing.

### Phase 3

- Enable application mode only for qualified node images and workloads.
- Keep the class opt-in.
- Publish VM-scoped coordination and three-hour frequency limitations.

---

## 17. Completion criteria

- One same-VM group creates one VM Restore Point.
- Every source disk maps to one Disk Restore Point.
- Crash mode performs no guest freeze or quiesce.
- Application mode delegates coordination to Azure VMSnapshot.
- A ready application group has result `ApplicationConsistent`.
- Filesystem fallback is surfaced as failure for application mode.
- Get reconstructs the group after driver restart.
- Delete is idempotent.
- Member PVC restore succeeds.
- Cross-VM input fails before Azure creation.
- Existing individual snapshot tests continue to pass.

---

## 18. References

- [Support matrix for VM restore points](https://learn.microsoft.com/azure/virtual-machines/concepts-restore-points)
- [Use virtual machine restore points](https://learn.microsoft.com/azure/virtual-machines/virtual-machines-create-restore-points)
- [VM Snapshot extension using VM restore points](https://learn.microsoft.com/azure/virtual-machines/virtual-machines-restore-points-vm-snapshot-extension)
- [Kubernetes VolumeGroupSnapshot](https://kubernetes.io/docs/concepts/storage/volume-group-snapshots/)
- [CSI specification](https://github.com/container-storage-interface/spec/blob/master/csi.proto)
