# Multi-Disk VolumeGroupSnapshot Using Azure VM Restore Points

**Status:** Proposed for review

## Overview

The requirement is to create a consistent snapshot of multiple Azure Disk PVCs used by one application.

Creating managed-disk snapshots one after another does not give all disks the same capture point. Azure VM Restore Points address this by capturing the selected disks attached to a VM as one operation.

```text
VolumeGroupSnapshot
└── VM Restore Point
    ├── Disk Restore Point for PVC 1
    ├── Disk Restore Point for PVC 2
    └── Disk Restore Point for PVC 3
```

All disks in the group must be attached to the same VM.

## Approaches

Two approaches were evaluated:

1. **Crash-consistent Restore Point**
2. **Application-consistent Restore Point**

| Approach | Guest coordination | Result | Decision |
|---|---|---|---|
| Crash consistent | None | Storage-level multi-disk consistency | Not selected |
| Application consistent | Azure VMSnapshot extension | Application and filesystem consistency when coordination succeeds | Selected |

---

## Approach 1: Crash-Consistent Restore Point

### What

A crash-consistent Restore Point captures the persisted state of all selected disks at a common point.

It is equivalent to the disk state after a VM suddenly loses power. Azure preserves disk write ordering, but it does not coordinate with the operating system or application before capture.

### Why consider it

- It is the simplest Restore Point workflow.
- It does not require guest scripts, VSS writers, or application pause.
- It has fewer guest-side dependencies.
- It provides a common multi-disk capture point, unlike sequential managed-disk snapshots.
- It is suitable for workloads designed to recover after an abrupt VM failure.

### How

1. The CSI driver receives `CreateVolumeGroupSnapshot`.
2. It resolves every source volume to an Azure managed disk.
3. It verifies that all disks are attached to the same VM.
4. It creates or reuses a Restore Point Collection for that VM.
5. It excludes the OS disk and unselected data disks.
6. It creates a Restore Point with:

   ```text
   consistencyMode = CrashConsistent
   ```

7. Azure creates one Disk Restore Point for each selected disk.
8. The CSI driver returns the VM Restore Point ID as the group ID and Disk Restore Point IDs as member IDs.

The CSI driver does not run `sync`, `fsfreeze`, or application hooks.

### Why we are not using this approach

Crash consistency does not satisfy the filesystem/application-consistency requirement.

Data may still be held in application memory, database buffers, the operating-system page cache, or incomplete filesystem journal transactions.

After restore:

- The filesystem may require journal recovery.
- The application may require crash recovery.
- In-flight transactions may be lost.
- Files belonging to one logical transaction may not agree.

A common storage capture point does not automatically make the filesystem or application consistent. Therefore, this approach is not selected.

---

## Approach 2: Application-Consistent Restore Point

### What

An application-consistent Restore Point coordinates with the guest VM before capturing the selected disks.

Azure uses the VMSnapshot extension:

- Windows uses VSS writers.
- Linux uses configured pre- and post-snapshot scripts.

The application flushes pending data and temporarily pauses writes. Azure captures the selected disks and then resumes the application.

### Why use it

- Application buffers can be flushed before capture.
- Filesystem writes can complete before capture.
- New writes are paused while Azure establishes the snapshot point.
- All selected disks are captured in one Restore Point operation.
- Azure owns the quiesce, capture, and resume sequence.

Using Azure coordination avoids adding another freeze protocol to the CSI driver. A CSI-owned `fsfreeze` design would need controller-to-node coordination, restart recovery, timeouts, safe thaw, and handling for partial freeze or unfreeze failures.

### How

1. The CSI driver receives `CreateVolumeGroupSnapshot` with all source volume IDs.
2. It reads every managed disk and uses `ManagedBy` to identify the source VM.
3. It rejects the request if any disk is detached or belongs to another VM.
4. It reads the VM storage profile and excludes the OS disk and unselected data disks.
5. It creates or reuses one Restore Point Collection for the VM.
6. It creates the Restore Point with `consistencyMode` omitted.

   ```text
   consistencyMode = omitted
   ```

   Omitting the field requests Azure's application-consistent workflow. The driver must not send `ApplicationConsistent` as an input value.

7. Azure VMSnapshot performs:

   ```text
   Quiesce application
       -> Flush and pause writes
       -> Capture selected disks
       -> Resume application
   ```

8. The driver verifies:
   - Azure returned `ApplicationConsistent`.
   - The captured disk set exactly matches the requested set.
   - Every requested disk has a Disk Restore Point.
9. The driver returns:
   - VM Restore Point ID as `group_snapshot_id`.
   - Disk Restore Point IDs as member `snapshot_id` values.
10. During restore, each PVC is recreated from its Disk Restore Point using Azure disk create option `Restore`.

The CSI driver must not run `fsfreeze` while Azure VMSnapshot is coordinating the guest. There must be only one owner of quiesce and resume.

Azure may return `FileSystemConsistent` when application coordination does not fully succeed. In that case, the driver must not publish the group as application-consistent. It returns an error and attempts to delete the unusable Restore Point.

### Important limitations

- All selected disks must be attached to the same VM.
- A supported OS, healthy Azure VM Agent, and working VMSnapshot extension are required.
- Linux scripts must perform the application-specific flush or pause operation.
- `sync` or filesystem freeze alone does not guarantee application transaction consistency.
- Guest coordination is VM-scoped, not Pod-scoped, so unrelated workloads on the node may be affected.
- Kubernetes restores the member PVCs individually; it does not provide an atomic group-restore RPC.

### Decision

Use the **application-consistent Restore Point** approach.

The CSI driver will use one VM Restore Point as the group snapshot, Disk Restore Points as member snapshots, and Azure VMSnapshot for guest coordination. A snapshot is ready only when Azure reports `ApplicationConsistent` and the captured disk set exactly matches the request.

This design provides a common multi-disk capture point without introducing a second filesystem-freeze mechanism in the CSI driver.

---

## References

- [Support matrix for VM restore points](https://learn.microsoft.com/azure/virtual-machines/concepts-restore-points)
- [Use virtual machine restore points](https://learn.microsoft.com/azure/virtual-machines/virtual-machines-create-restore-points)
- [VM Snapshot extension using VM restore points](https://learn.microsoft.com/azure/virtual-machines/virtual-machines-restore-points-vm-snapshot-extension)
- [Kubernetes VolumeGroupSnapshot](https://kubernetes.io/docs/concepts/storage/volume-group-snapshots/)
