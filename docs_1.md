# Approach 3: Filesystem-Consistent Multi-Disk Snapshot Using Azure VM Restore Points

## Overview

This approach provides filesystem-consistent snapshots for a group of
Kubernetes PersistentVolumes backed by Azure managed disks.

The design combines two mechanisms:

1. CSI node-side filesystem freeze and unfreeze ensures that pending filesystem
   writes and metadata are committed and that new writes are blocked.
2. One Azure VM Restore Point captures all selected disks under one Azure
   multi-disk consistency operation.

This approach is intended for Kubernetes `VolumeGroupSnapshot`. Existing
single-volume `CreateSnapshot` workflows can continue to use the normal Azure
managed-disk snapshot path.

The approach applies only when:

- the PersistentVolumes use `Filesystem` volume mode;
- all selected disks are Azure managed data disks;
- all selected disks are attached to the same supported Azure VM;
- the VM and disk types support Azure VM Restore Points;
- the filesystems can be frozen through the CSI node component.

Application consistency is not provided by this approach. Filesystem freeze
flushes filesystem-managed data, but it does not force a database or other
application to reach a transaction boundary.

## Problem with independent member snapshots

Approaches 1 and 2 coordinate filesystem freeze, but the group snapshot still
consists of separate Azure disk snapshot operations:

```text
Freeze volume 1
Freeze volume 2
Freeze volume 3
        |
        v
Create snapshot for disk 1
Create snapshot for disk 2
Create snapshot for disk 3
        |
        v
Unfreeze all volumes
```

Keeping every filesystem frozen ensures that the disk contents do not change
while the individual snapshot requests are submitted. However, each disk
snapshot remains a separate Azure resource operation with its own snapshot
time.

Azure VM Restore Points provide a better storage primitive for the same-VM
case. One Restore Point contains one Disk Restore Point for every included
managed disk:

```text
Freeze all selected filesystems
        |
        v
Create one Azure VM Restore Point
        |
        +-- Disk Restore Point for volume 1
        +-- Disk Restore Point for volume 2
        +-- Disk Restore Point for volume 3
        |
        v
Unfreeze all selected filesystems
```

The filesystem freeze provides filesystem cleanliness. The VM Restore Point
provides the common multi-disk storage capture.

## Consistency guarantee

A successful operation guarantees that:

- every selected filesystem was frozen before Restore Point creation;
- dirty filesystem data and metadata were flushed;
- new filesystem writes were blocked during the snapshot boundary;
- all member Disk Restore Points belong to the same VM Restore Point;
- the set of captured data disks exactly matches the requested volume group;
- all filesystems frozen by this operation were scheduled for unfreeze after
  Azure established the Restore Point snapshot boundary.

This is a **filesystem-consistent multi-disk snapshot**.

It does not guarantee:

- that data held only in an application's memory was flushed;
- that a database completed or rolled back active transactions;
- consistency across disks attached to different VMs;
- atomic provisioning or attachment of restored Kubernetes volumes.

## Design

### Kubernetes and Azure resource mapping

The Kubernetes group snapshot maps to Azure resources as follows:

| Kubernetes/CSI object | Azure resource |
| --- | --- |
| VolumeGroupSnapshot | VM Restore Point |
| Group snapshot handle | Restore Point ARM resource ID |
| Member VolumeSnapshot | Disk Restore Point |
| Member snapshot handle | Disk Restore Point ARM resource ID |
| Source volume ID | Managed disk ARM resource ID |

The Restore Point Collection is scoped to one source VM. The driver may reuse a
compatible collection for later group snapshots of the same VM.

### Controller-to-node coordination

This approach reuses the VolumeAttachment annotation protocol described in
Approach 2.

The controller requests a freeze generation by setting:

```text
csi.azure.com/freeze-required: "<group-request-id>"
```

The node reports:

```text
csi.azure.com/freeze-state: "skipped|user-frozen|freezing|frozen|unfrozen|failed"
```

The request ID identifies one group-snapshot generation. All member volumes
must be associated with the same group operation, while each VolumeAttachment
retains its own freeze state.

The node keeps a durable local ownership record for every filesystem it
successfully freezes. A filesystem is unfrozen only when the node can establish
that the current group operation owns that freeze.

### Consistency policy

Filesystem-consistent group snapshots use strict all-member behavior.

If any required member cannot be frozen:

1. do not create the Restore Point;
2. remove the freeze request from all members;
3. unfreeze every filesystem already frozen by this operation;
4. fail the group snapshot with an actionable error.

The driver must not silently create a crash-consistent Restore Point and report
it as filesystem-consistent.

A separate best-effort policy may explicitly allow fallback to crash
consistency, but the resulting snapshot must be reported as crash-consistent
and must not satisfy a strict filesystem-consistent request.

## Preconditions and topology validation

Topology validation happens before any filesystem is frozen.

For every requested source volume, the controller validates:

- the volume ID identifies an Azure managed disk;
- the disk exists;
- the disk is currently attached;
- the disk is attached to only one VM;
- the disk is not an unsupported shared disk;
- the disk and VM support Restore Points;
- all requested disks resolve to the same VM;
- the VM orchestration model supports Restore Points.

The operation fails before freeze if:

- one or more disks are unattached;
- selected disks are attached to different VMs;
- a selected disk is attached to multiple VMs;
- the VM is unsupported, including an unsupported scale-set orchestration
  model;
- a selected disk type cannot participate in the requested Restore Point;
- a member has no usable filesystem attachment information.

This same-VM restriction is fundamental. One Azure VM Restore Point cannot
provide one storage consistency boundary across multiple VMs.

## Disk selection

A VM may contain disks that are not part of the Kubernetes volume group:

- the VM OS disk;
- unrelated data disks used by other workloads;
- temporary or unsupported disks.

The Restore Point request must exclude:

- the OS disk;
- every attached data disk not selected by the VolumeGroupSnapshot;
- every disk that Azure requires to be excluded and that is not a requested
  member.

The controller builds the exclusion set from the VM's current storage profile.
It must not create a Restore Point of every VM disk and expose only some of the
resulting Disk Restore Points. Capturing unrelated disks would weaken workload
isolation and increase storage use.

After Restore Point creation, the controller compares the captured source disk
set with the requested source disk set. The two sets must match exactly.

If the sets differ, the operation is not accepted as a valid group snapshot.
The controller initiates cleanup of the incorrect Restore Point and returns a
retryable failure. This protects against a disk attach or detach race between
topology validation and Restore Point creation.

## Workflow

### CSI controller

1. Receive a `CreateVolumeGroupSnapshot` request containing the source volume
   IDs.
2. Validate the request and remove duplicate volume IDs.
3. Resolve every managed disk and verify that all disks are attached to one
   supported VM.
4. Read the VM storage profile and calculate the excluded disk set.
5. Resolve the VolumeAttachment for every member filesystem.
6. Verify that the responsible CSI node components are available.
7. Generate one unique group freeze request ID.
8. Set `freeze-required` with that request ID on every member
   VolumeAttachment.
9. Return or continue reporting the group as not ready while waiting for all
   required members to report `frozen`.
10. If any member reports `failed`, `skipped`, `user-frozen`, disappears, or
    exceeds the strict timeout:
    - do not create the Restore Point;
    - remove all group freeze requests;
    - wait for or continue reconciling unfreeze;
    - fail the group operation.
11. When every required member reports `frozen`, revalidate that the disks are
    still attached to the same source VM.
12. Create or reuse the Restore Point Collection associated with that VM.
13. Create one Restore Point with:
    - the source VM;
    - explicit crash-consistent Azure storage mode;
    - the OS disk and all unselected disks in `excludeDisks`.
14. Wait until Azure reports the state that proves the Restore Point snapshot
    boundary has been established.
15. Read the Restore Point metadata and map every requested source disk to its
    Disk Restore Point.
16. Verify that the captured source set exactly equals the requested source
    set.
17. Remove `freeze-required` from every member VolumeAttachment.
18. The node unfreezes every filesystem owned by this group generation.
19. Return:
    - the Restore Point ID as the group snapshot handle;
    - one Disk Restore Point ID for each member snapshot;
    - one common group creation time;
    - readiness based on the Restore Point and all Disk Restore Points.

The exact Azure state at step 14 must be validated. It must indicate that the
storage consistency point has been established, not merely that Azure received
the initial request.

### CSI node

1. Watch VolumeAttachments for the node.
2. On a new `freeze-required` group request:
   - validate the request ID and deadline;
   - resolve the global mount point for the volume;
   - recover any durable local freeze record;
   - mark the operation as `freezing`;
   - freeze the filesystem;
   - persist ownership and result before publishing `frozen`.
3. If the filesystem is not mounted, report `skipped`. In strict mode this
   prevents Restore Point creation unless the design explicitly treats an
   unmounted volume as already safe.
4. If the filesystem is already frozen by another owner, report
   `user-frozen`. Do not unfreeze it.
5. When `freeze-required` is removed:
   - unfreeze only if the durable record proves this group generation owns the
     freeze;
   - treat an already-unfrozen or unmounted filesystem as successful cleanup;
   - remove the freeze-state annotation and durable record after successful
     cleanup.
6. Periodically reconcile local records with VolumeAttachment state so that
   filesystems are eventually unfrozen after controller or node restarts.

## Idempotency and retry behavior

The CSI group snapshot name must map deterministically to one Restore Point.

For repeated requests:

- the same name and same source volume set returns the existing group snapshot;
- the same name with a different source volume set fails rather than replacing
  the existing Restore Point;
- an in-progress Restore Point is returned as not ready;
- an ambiguous Azure response is reconciled by reading the deterministic
  Restore Point instead of creating another resource.

Restore Point creation must be serialized for the same source VM because Azure
may reject concurrent Restore Point operations for one VM.

No in-memory cache is the sole source of truth. Kubernetes annotations, durable
node ownership records, and Azure Restore Point resources allow reconciliation
after process restarts.

## Concurrent snapshot behavior

Multiple snapshot requests may overlap on one or more volumes.

- A filesystem must remain frozen while any active operation still requires
  the same compatible freeze generation.
- One operation must not remove another operation's freeze request.
- Restore Point creation for the same VM is serialized.
- Requests with different source sets or consistency policies must not
  accidentally share completion state.
- A new operation must not interpret a stale `frozen` state from a previous
  VolumeAttachment instance as proof that the current attachment is frozen.

The initial version may serialize all filesystem-consistent group snapshots per
VM. This is simpler and safer than attempting to merge partially overlapping
groups.

## Failure handling

### Failure before all filesystems are frozen

- Do not create a Restore Point.
- Remove all freeze requests.
- Unfreeze members owned by the operation.
- Report the member and reason that prevented consistency.

### Controller restart while members are frozen

- Rebuild the active group state from Kubernetes annotations.
- Recover the original request ID and deadline.
- Continue Restore Point creation only if every member and attachment still
  satisfies the original topology.
- Otherwise initiate unfreeze and fail the operation.

### Node restart after freeze

- Recover freeze ownership from the host-persistent record.
- Restore the published freeze state or conservatively reconcile and unfreeze
  if ownership is uncertain.
- Never assume that a missing in-memory record means the filesystem is safe.

### Attachment changes during the operation

- Before Restore Point creation, abort and unfreeze.
- After Restore Point creation, compare the captured and requested source sets.
- Do not report success when membership differs.

### Restore Point creation fails

- Remove freeze requests immediately.
- Continue unfreeze reconciliation even if the snapshot request has already
  failed.
- Preserve enough state to determine whether Azure created a Restore Point
  despite an ambiguous response.

### Unfreeze fails

- Retain the ownership record.
- Retry with bounded backoff.
- Raise a high-severity event because application writes may remain blocked.
- Distinguish this from snapshot validity: the Restore Point can be valid even
  though post-snapshot recovery failed.

### Controller loses connectivity after Restore Point creation

- The node-side deadline prevents an indefinite freeze.
- The controller reconciles the deterministic Restore Point on retry.
- If the node unfreezes because its safety deadline expires before the
  controller confirms the snapshot boundary, the operation must not claim
  filesystem consistency unless Azure evidence proves the boundary occurred
  while all members were frozen.

## Delete behavior

Deleting the VolumeGroupSnapshot deletes the parent Restore Point. Its member
Disk Restore Points are children of that Restore Point and are not treated as
independent snapshots.

Deleting one member snapshot must not implicitly delete the entire group.
Member lifecycle follows the Kubernetes group-snapshot deletion policy and the
parent Restore Point lifecycle.

An empty driver-owned Restore Point Collection may be deleted after its final
Restore Point is removed. Collection cleanup is not allowed to invalidate an
already successful group deletion.

## Restore behavior

Each Disk Restore Point can be used to create a new managed disk. Kubernetes
restores each member PVC from the corresponding generated member
VolumeSnapshot.

The data on all restored disks comes from one common Restore Point, but
Kubernetes may provision the restored PVCs at different times. The workload
must start only after every required restored PVC is ready.

This design does not provide in-place rollback of the original disks.

## Interaction with AKS Backup hooks

Only one component should own filesystem freeze and unfreeze.

If AKS Backup or another tool has already frozen a filesystem:

- the CSI node reports `user-frozen`;
- it does not unfreeze that filesystem;
- strict CSI-managed filesystem consistency does not proceed unless ownership
  and coordination are explicitly defined.

Application-specific backup hooks may still be used to quiesce a database, but
they should not independently run `fsfreeze` when this approach owns filesystem
coordination.

A later application-consistent design can use:

```text
Application pre-hooks
        |
        v
Approach 3 filesystem freeze
        |
        v
One Azure VM Restore Point
        |
        v
Filesystem unfreeze
        |
        v
Application post-hooks
```

Application hooks are outside the scope of this approach.

## Observability

Events and metrics should report:

- requested and achieved consistency mode;
- source VM and member count;
- freeze success, failure, skip, and timeout per member;
- time waiting for all members to freeze;
- Restore Point creation time and provisioning state;
- requested versus captured disk-set mismatch;
- unsupported topology or VM type;
- unfreeze success and failure;
- valid snapshot with failed post-snapshot cleanup.

Recommended events include:

- `VolumeGroupFreezeStarted`
- `VolumeGroupFreezeSucceeded`
- `VolumeGroupFreezeFailed`
- `RestorePointCreationStarted`
- `RestorePointCreationSucceeded`
- `RestorePointDiskSetMismatch`
- `VolumeGroupThawSucceeded`
- `VolumeGroupThawFailed`
- `FilesystemConsistencyNotGuaranteed`

The system must not emit a filesystem-consistent success event unless every
required member was frozen at the established Restore Point boundary.

## Security

- The node performs only built-in freeze and unfreeze operations; no arbitrary
  commands are accepted by this approach.
- The node unfreezes only filesystems for which it has a durable ownership
  record.
- The controller captures only requested data disks and explicitly excludes the
  OS disk and unrelated data disks.
- Azure permissions are limited to reading the source VM and disks and managing
  Restore Point Collections, Restore Points, and Disk Restore Points.
- Resource IDs, annotation values, and request identifiers are validated and
  bounded.
- Credentials and application data are never recorded in events or logs.

## Advantages

- Provides one Azure multi-disk storage capture for all selected same-VM disks.
- Reuses the durable freeze and recovery protocol from Approach 2.
- Does not require a privileged sidecar container in every application Pod.
- Applies consistently to any Kubernetes workflow using
  `VolumeGroupSnapshot`.
- Avoids exposing unrelated VM disks by using explicit exclusions.
- Supports restart-safe reconciliation through Kubernetes and Azure state.

## Limitations

- All selected disks must be attached to the same supported VM.
- Cross-node and cross-VM groups are not supported.
- VM Scale Sets with unsupported orchestration modes cannot use this approach.
- Shared and unattached disks are not supported.
- Filesystem freeze can block application I/O and may hang under unhealthy
  storage or filesystem conditions.
- Restore Points alone do not provide application transaction consistency.
- Kubernetes restores member PVCs independently even though their data belongs
  to one common Restore Point.
- The exact Azure state that permits safe unfreeze must be confirmed through
  Azure API documentation and validation.

## Open questions

1. Which AKS VM and VM Scale Set orchestration models are in the required
   support matrix?
2. Is an unmounted selected volume considered safely quiesced, or should strict
   mode reject it?
3. Which Azure Restore Point state proves that the snapshot boundary has been
   established and filesystems can be unfrozen?
4. Can the OS disk and every unrelated data disk always be excluded for all
   supported VM and disk configurations?
5. Which managed disk SKUs, encryption modes, zones, and shared-disk settings
   are supported?
6. Where should Restore Point Collections be created and how long should they
   be retained?
7. Should all group snapshot operations be serialized per VM in the first
   release?
8. How should the achieved consistency level be represented if best-effort
   fallback is supported?
9. How should the system expose a valid Restore Point when unfreeze cleanup
   fails?
10. How will application startup be gated until every restored member PVC is
    ready?

## Summary

Approach 3 keeps the node-side filesystem consistency mechanism from Approach
2 and replaces sequential member snapshots with one Azure VM Restore Point.

```text
Validate one-VM topology
        |
        v
Freeze every selected filesystem
        |
        v
Create one VM Restore Point
        |
        v
Validate one Disk Restore Point per selected disk
        |
        v
Unfreeze every driver-owned filesystem
```

This provides filesystem-consistent multi-disk snapshots for the supported
same-VM topology. It is stronger than independently snapshotting each member
disk, while remaining separate from application-specific transaction
consistency.
