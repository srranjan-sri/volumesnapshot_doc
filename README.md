# Approach 3: Filesystem-Consistent Multi-Disk Snapshots Using Azure VM Restore Points

## Status

Revised design proposal based on the Azure VM Restore Point documentation and
linked supporting resources available on October 12, 2026.

This document defines behavior and product boundaries. It does not define code
structure or an implementation plan.

## Executive decision

Provide filesystem-consistent Kubernetes volume group snapshots by combining:

1. CSI-managed freeze and unfreeze of every selected filesystem.
2. One explicitly requested Azure `CrashConsistent` VM Restore Point.
3. One Disk Restore Point for each selected Azure managed disk.

```text
Validate one-VM topology
        |
        v
Freeze every selected filesystem
        |
        v
Create one CrashConsistent VM Restore Point
        |
        v
Verify one Disk Restore Point per selected disk
        |
        v
Unfreeze every driver-owned filesystem
```

Azure labels the Restore Point `CrashConsistent` because guest coordination is
not performed by the Azure VMSnapshot extension. The end-to-end Kubernetes
workflow is nevertheless filesystem-consistent because the CSI layer freezes
all selected filesystems before Azure establishes the multi-disk storage
snapshot.

The achieved guarantee must be described as:

> CSI-coordinated filesystem-consistent volume group snapshot backed by one
> Azure crash-consistent VM Restore Point.

The design must not simultaneously request Azure application consistency.
Doing so would introduce a second guest-quiesce owner through the Azure
VMSnapshot extension and could cause double freeze, ambiguous ownership, and
unsafe thaw behavior.

## 1. Azure product facts that shape this design

### 1.1 Restore Point resource model

Azure documents the following hierarchy:

```text
Restore Point Collection
    |
    +-- VM Restore Point
            |
            +-- Disk Restore Point for included disk 1
            +-- Disk Restore Point for included disk 2
            +-- Disk Restore Point for included disk N
```

A Restore Point Collection:

- is an Azure Resource Manager resource;
- is scoped to one source VM;
- contains Restore Points for that VM.

A VM Restore Point stores:

- the VM configuration at capture time;
- point-in-time snapshots of included attached managed disks;
- one Disk Restore Point for each included disk.

Restore Points are incremental. The first point stores a full copy, while later
points store changes relative to the preceding snapshot chain.

Azure supports excluding individual disks from a Restore Point.

### 1.2 Azure consistency modes

Azure exposes two Restore Point request modes:

| Azure mode | Request behavior |
| --- | --- |
| Application-consistent | Omit `consistencyMode`; Azure uses VSS on Windows or the VMSnapshot Linux extension and pre/post scripts on Linux |
| Crash-consistent | Set `consistencyMode` to `CrashConsistent` |

Azure does not expose `FileSystemConsistent` as an explicit Restore Point
request value.

Azure can report or produce a filesystem-consistent result as a fallback when
application-consistency processing fails. That fallback is not the contract
used by Approach 3 because:

- it is controlled by Azure extension behavior rather than the CSI group
  operation;
- it does not provide CSI with freeze ownership;
- fallback behavior can vary by OS and failure mode;
- it cannot safely coordinate with an additional CSI-managed freeze.

### 1.3 VMSnapshot extension behavior

For an application-consistent Restore Point:

- Azure automatically installs the VMSnapshot extension when required;
- on Windows, the extension coordinates through VSS;
- on Linux, it runs application pre- and post-scripts;
- Azure waits for every VM extension to finish provisioning before beginning
  Restore Point creation;
- VM Agent, extension, network, script, and VSS failures can prevent or degrade
  the requested consistency.

On Windows, Microsoft documents that a VSS writer failure causes fallback to a
filesystem-consistent Restore Point for the next three attempts. Azure resumes
application-consistent attempts from the fourth attempt.

The Linux application-consistency script framework supports:

- application-native pre-scripts to quiesce I/O and flush memory;
- post-scripts to resume application I/O;
- optional filesystem freeze;
- per-script timeouts up to 1,800 seconds;
- optional continuation with filesystem/crash consistency after hook failure.

That framework is relevant to a future application-consistent approach, but it
is not part of this filesystem-consistent Approach 3.

### 1.4 Restore Point support restrictions

The official support matrix establishes the following:

| Scenario | Support relevant to Approach 3 |
| --- | --- |
| Managed disks | Supported |
| Unmanaged disks | Unsupported |
| Standard HDD/SSD, Premium SSD, ZRS | Supported, subject to VM support |
| Ultra Disk | Application consistency only; crash consistency unsupported |
| Premium SSD v2 | Application consistency only; crash consistency unsupported |
| Write-accelerated disk | Crash consistency unsupported |
| Shared disk | Unsupported in both modes; must be excluded |
| Ephemeral OS disk | Unsupported; must be excluded |
| Temporary disk | VM can have one, but its data is not included |
| VMSS Uniform | Unsupported |
| VMSS Flexible | Supported per individual VM |
| Availability set | Supported per individual VM |
| Azure Boost VM size | Application consistency only; crash consistency unsupported |
| Confidential VM | Application consistency only in the documented preview matrix |
| Crash-consistent OS support | All operating systems |

Approach 3 therefore cannot support a requested group containing:

- Ultra Disks;
- Premium SSD v2 disks;
- write-accelerated disks;
- shared disks;
- unmanaged disks;
- disks attached to a VM for which crash-consistent Restore Points are
  unsupported.

Unsupported disks attached to the same VM but not selected by the group must be
excluded from the Restore Point.

### 1.5 Frequency, concurrency, and quota limits

Azure documents:

- minimum application-consistent Restore Point frequency: three hours;
- minimum crash-consistent Restore Point frequency: one hour;
- maximum retained Restore Points per VM: 500 across all collections and
  consistency modes;
- maximum Restore Point Collections: 10,000 per subscription per region;
- concurrent Restore Point creation for the same VM is unsupported;
- Restore Point create operations are throttled and return HTTP 429 with a
  `Retry-After` value when the limit is exceeded.

The overview also lists up to three Restore Point PUT operations per VM per
hour. This does not override the separately documented minimum one-hour
frequency for crash-consistent Restore Points. Until Azure clarifies the
difference, the design uses the stricter product boundary:

> At most one filesystem-consistent Approach 3 snapshot per source VM per hour.

This is a significant product limitation for Kubernetes snapshot schedules.

### 1.6 VM mutation and movement

Azure documents that:

- Restore Point creation can fail if the VM changes while creation is in
  progress;
- a badly allocated attached disk can fail the complete Restore Point unless
  excluded;
- source VM movement between resource groups or subscriptions can invalidate
  the source reference held by an existing collection.

The Microsoft pages use slightly different wording about whether movement is
"supported." Both agree operationally that old source references do not follow
the VM and old collections may require cleanup. This design therefore treats
VM resource ID stability as an invariant for a collection.

### 1.7 Encryption behavior

The support matrix states that source disk encryption is not automatically
enabled on the Restore Point for:

- service-managed keys;
- customer-managed keys;
- double encryption;
- host-based encryption;
- Azure Disk Encryption.

This requires explicit security review before production adoption. Restored
managed disks must be created with the intended target encryption policy; the
design must not assume that source encryption configuration is inherited.

### 1.8 Restore and readiness

For local Restore Points:

- `provisioningState` transitions from `Creating` to `Succeeded` or `Failed`;
- Microsoft states that local points normally complete within seconds.

For cross-region points:

- creation/copy is long-running;
- readiness requires all Disk Restore Point copies to complete;
- per-disk progress is available through instance view.

Restore is not in-place. A Disk Restore Point is used to create a new managed
disk with restore semantics. Kubernetes then uses those new disks for restored
PersistentVolumes.

Approach 3 is a local-region snapshot design. Cross-region copy is a separate
disaster-recovery workflow and is out of scope.

## 2. Why Approach 3 uses CSI freeze plus CrashConsistent Restore Point

There are two possible guest-consistency owners:

### Option A: Azure-owned guest coordination

```text
Request application-consistent Restore Point
        |
        v
Azure installs/invokes VMSnapshot extension
        |
        v
VSS or Linux pre/post scripts coordinate guest state
```

This is appropriate for VM-level application consistency, but it has several
problems for the proposed Kubernetes filesystem-consistency feature:

- Azure coordinates the entire VM, not a Kubernetes-selected PVC group.
- CSI cannot prove which selected filesystems Azure froze.
- Kubernetes does not own or directly observe extension freeze state.
- users must configure application scripts on the VM for Linux application
  consistency;
- extension failures may cause a different achieved mode;
- combining it with CSI `fsfreeze` creates two freeze owners.

### Option B: CSI-owned filesystem coordination

```text
CSI freezes exactly the selected Kubernetes filesystems
        |
        v
CSI requests a CrashConsistent Restore Point
        |
        v
Azure captures included disks as one storage group
        |
        v
CSI thaws only filesystems it froze
```

This is the recommended basis for Approach 3 because:

- freeze ownership is explicit;
- the selected Kubernetes volumes are known;
- strict all-member success can be enforced;
- CSI can recover ownership after restart;
- Azure does not run a competing application-consistency extension workflow;
- the Azure Restore Point supplies the common storage snapshot boundary.

The tradeoff is a smaller support matrix because crash-consistent Restore
Points do not support all disk and VM types.

## 3. Scope

### In scope

- Kubernetes PersistentVolumes provisioned from supported Azure managed data
  disks.
- `Filesystem` volume mode.
- Kubernetes volume groups whose selected disks are attached to one supported
  VM.
- Local-region Restore Points.
- Strict filesystem consistency.
- Creation, status, deletion, and per-member restore semantics.
- Recovery after CSI controller or node restart.

### Out of scope

- application transaction consistency;
- raw block volumes;
- cross-VM or cross-node volume groups;
- VMSS Uniform;
- shared disks;
- unsupported crash-consistency disk and VM types;
- cross-region Restore Point creation or copy;
- Instant Access;
- in-place rollback of existing disks;
- atomic creation or attachment of all restored Kubernetes volumes;
- arbitrary user commands executed by the CSI driver.

## 4. Consistency contract

A successful Approach 3 group snapshot means:

1. Every requested source disk passed topology and support validation.
2. Every selected filesystem was successfully frozen by this group operation.
3. Every selected filesystem remained frozen until Azure reported the Restore
   Point as `Succeeded`.
4. Every member Disk Restore Point belongs to the same parent Restore Point.
5. The set of captured source data disks exactly equals the requested set.
6. CSI initiated unfreeze for every filesystem it owned.

The Restore Point cannot be reported ready until:

- its `provisioningState` is `Succeeded`;
- every requested source disk has one Disk Restore Point ID;
- the captured member set exactly matches the request;
- no unexpected disk is included as a member;
- each Disk Restore Point is usable according to the applicable Azure state.

### Safe thaw boundary

The public Azure documentation does not identify an earlier intermediate state
that guarantees the storage snapshot boundary is durable.

Therefore, the conservative design is:

> Keep filesystems frozen until the local Restore Point reaches
> `provisioningState=Succeeded` and member mapping is validated.

This may block application writes for several seconds. An earlier thaw point
can be adopted only after Azure Compute provides a documented contract for it.

### Timeout behavior

If Azure does not reach `Succeeded` within the configured freeze safety
deadline:

1. unfreeze every filesystem owned by the operation;
2. do not report filesystem-consistent success;
3. reconcile whether Azure eventually created the Restore Point;
4. delete or quarantine that point because CSI cannot prove its snapshot
   boundary occurred while every member remained frozen.

The design prioritizes avoiding indefinite application I/O blockage over
retaining an unproven filesystem-consistent snapshot.

## 5. Preconditions

All preconditions are checked before freezing application I/O.

### 5.1 Source-volume validation

For every member:

- the source is an Azure managed-disk resource;
- the disk exists;
- the PersistentVolume uses `Filesystem` mode;
- the filesystem is mounted through the responsible CSI node;
- the disk type supports crash-consistent VM Restore Points;
- the disk is not shared;
- the disk is attached to exactly one source VM.

### 5.2 Group topology

All selected disks must:

- be attached to the same VM resource ID;
- be in the same source VM storage profile;
- be in the same Azure region as that VM;
- be managed under a topology supported by VM Restore Points.

The request fails without freezing any filesystem if the selected disks span
VMs.

### 5.3 Source VM health

The source VM must:

- be running and in a healthy provisioning state;
- not be in a conflicting update operation;
- support crash-consistent Restore Points;
- not be a VMSS Uniform instance;
- have no unresolved disk allocation failure that would affect the request.

Crash-consistent Restore Points do not require application-consistency scripts,
but VM health remains relevant to stable disk membership and Azure operation
success.

### 5.4 Frequency and quota

Before freeze, reject or defer the request if:

- a Restore Point operation for the VM is already active;
- the one-hour crash-consistency interval has not elapsed;
- the VM is at the 500-point retention limit;
- subscription/region collection quotas prevent creation.

Applications must not be frozen while waiting for an Azure quota or frequency
window.

## 6. Exact disk-selection contract

A Kubernetes group normally selects only data disks. The VM also has:

- an OS disk;
- potentially unrelated data disks;
- potentially unsupported disks;
- temporary disks whose data Azure does not capture.

The Restore Point request includes `excludeDisks` for:

- the OS disk;
- every attached managed data disk not selected by the group;
- every unsupported attached disk not selected by the group.

The ability to exclude the OS disk for all targeted VM configurations must be
validated in the Azure qualification phase. If Azure cannot create a Restore
Point containing exactly the requested data disks, that topology is
unsupported.

### Membership race protection

VM disk membership can change after validation but before Azure captures the
VM configuration.

Therefore:

1. read and retain the pre-freeze VM storage profile;
2. revalidate membership after all filesystems are frozen;
3. create the Restore Point;
4. read `sourceMetadata.storageProfile` after creation;
5. map each included data disk's managed-disk ID to its Disk Restore Point ID;
6. compare the captured source set with the requested source set.

The two sets must be identical.

If they differ:

- do not publish the group snapshot;
- unfreeze all filesystems;
- initiate deletion of the incorrect Restore Point;
- report a retryable topology-change failure.

## 7. Resource mapping and lifecycle

| Kubernetes concept | Azure concept |
| --- | --- |
| VolumeGroupSnapshot | One VM Restore Point |
| Group snapshot handle | Restore Point ARM ID |
| Member VolumeSnapshot | One Disk Restore Point |
| Member snapshot handle | Disk Restore Point ARM ID |
| Source volume | Azure managed data disk |

### Restore Point Collection

The collection is associated with one immutable source VM resource ID.

The design should reuse a compatible collection for that VM to:

- preserve incremental Restore Point behavior;
- avoid unnecessary collection growth;
- simplify quota and lifecycle management.

A collection must not be reused if its source VM ID or region differs.

### Restore Point

Each successful Kubernetes group snapshot maps to one Restore Point.

The Restore Point is created:

- in the source VM region;
- with `consistencyMode=CrashConsistent`;
- with the exact exclusion set;
- only after every filesystem is frozen.

### Disk Restore Points

Disk Restore Point IDs are discovered from:

```text
sourceMetadata.storageProfile.dataDisks[*].diskRestorePoint.id
```

Each returned member is associated with its source managed-disk ID. Azure array
ordering is not treated as identity.

### Delete

Deleting the Kubernetes group snapshot deletes the parent Restore Point.
Azure Restore Point deletion can be asynchronous:

- `200` means deleted;
- `202` means deletion accepted and must be polled;
- `204` means the resource does not exist and is an idempotent success.

Member Disk Restore Points follow the parent lifecycle and are not independently
deleted as unrelated snapshots.

An empty collection can be removed according to retention policy. Failure to
remove an already empty collection does not invalidate successful deletion of
the group snapshot.

### Restore

Each member Disk Restore Point creates a new managed disk using restore
semantics.

Restored disks must explicitly receive the intended:

- disk SKU;
- encryption configuration;
- network access policy;
- zone and region placement;
- resource group and tags.

The design must not assume that source disk SKU or encryption settings are
automatically inherited.

Kubernetes restores member PVCs separately. The workload must not start until
all required restored PVCs are bound and ready.

## 8. End-to-end workflow

### Phase A: Validate without disrupting I/O

1. Resolve all selected Kubernetes volumes and their Azure disk IDs.
2. Validate filesystem volume mode.
3. Validate disk type, attachment, sharing, and crash-consistency support.
4. Prove that all selected disks belong to one supported VM.
5. Verify VM health and stable provisioning state.
6. Verify frequency, concurrency, and retention limits.
7. Read the VM storage profile and calculate `excludeDisks`.
8. Verify that every selected filesystem can be coordinated through its CSI
   node.

Any failure in this phase ends the request without freezing application I/O.

### Phase B: Freeze as one group

1. Create one unique group freeze generation.
2. Request freeze for every selected member.
3. Each node persists freeze ownership before reporting success.
4. Wait until every required member reports `frozen`.
5. If any member fails, is skipped, is already frozen by another owner,
   disappears, or times out:
   - cancel the group;
   - thaw every filesystem owned by the group;
   - do not create the Restore Point.

The filesystem-consistent mode is strict and all-or-nothing.

### Phase C: Revalidate

After all members are frozen:

1. reread disk attachment and VM storage profile;
2. confirm the requested group is unchanged;
3. confirm the exclusion set is unchanged;
4. confirm no conflicting VM operation started.

If anything changed, thaw and fail before creating Azure resources.

### Phase D: Establish the Azure snapshot boundary

1. Create or reuse the source VM's Restore Point Collection.
2. Create one local Restore Point with:
   - `consistencyMode=CrashConsistent`;
   - the calculated `excludeDisks`.
3. Hold all selected filesystems frozen.
4. Poll until:
   - `provisioningState=Succeeded`; or
   - the safety deadline or terminal failure is reached.
5. On success, read source metadata and Disk Restore Point membership.
6. Verify exact source-set equality.

### Phase E: Thaw and publish

1. Remove the freeze requirement for all members.
2. Nodes thaw only filesystems owned by this generation.
3. Publish the group and member handles only after the Azure membership and
   readiness checks succeed.
4. Report thaw failures separately and urgently.

A Restore Point can be valid even if post-snapshot thaw fails. In that case:

- preserve the snapshot;
- mark the cleanup/application state as degraded;
- continue retrying thaw;
- emit an urgent operator-facing event.

## 9. Controller-to-node consistency protocol

Approach 3 adopts the annotation-based protocol from the senior design because
it avoids application Pod mutation and additional public CRDs.

Conceptually:

```text
csi.azure.com/freeze-required: "<group-generation-id>"
csi.azure.com/freeze-state: "freezing|frozen|skipped|user-frozen|failed|unfrozen"
```

### Ownership rules

- The controller owns the desired freeze generation.
- The node owns the observed freeze state.
- A node thaws only a filesystem it successfully froze.
- A group remains in freeze-required state until the Azure boundary is proven
  or the operation is aborted.
- An old attachment's state cannot prove that a recreated attachment is frozen.
- In-memory state is never the sole ownership record.

### Durable node state

The node persists, per volume:

- group generation ID;
- intended operation;
- freeze ownership;
- mount identity;
- current state;
- deadline.

This state survives CSI node process restart. It is removed only after
successful or safely reconciled thaw.

### Already-frozen filesystem

If the filesystem was frozen by another system:

- report `user-frozen`;
- do not claim ownership;
- do not thaw it;
- fail strict filesystem-consistent group creation.

This prevents CSI from unfreezing a filesystem owned by AKS Backup or another
operator.

## 10. Failure model

### Member fails before the group is frozen

- no Restore Point is created;
- already frozen members are thawed;
- the group fails.

### Restore Point creation returns a terminal failure

- thaw every owned filesystem immediately;
- preserve Azure failure details in bounded diagnostics;
- do not publish member snapshots.

### Azure response is ambiguous

- thaw at the safety deadline;
- reconcile the deterministic Restore Point identity;
- do not create a second point blindly;
- do not claim filesystem consistency unless Azure timing proves the boundary
  occurred before thaw.

### VM changes during Restore Point creation

Azure can reject the operation. If creation succeeds, exact post-validation
still determines whether the result represents the requested group.

### Controller restart

- rebuild group state from durable Kubernetes state;
- recover the original freeze generation and deadline;
- reconcile the Azure Restore Point;
- either finish validation or thaw and fail.

### Node restart

- recover durable local freeze ownership;
- restore observed state;
- thaw if the controller request disappeared or expired;
- never silently discard ownership.

### Node becomes unreachable while frozen

- the controller cannot guarantee immediate thaw;
- the node safety reconciler thaws after its bounded deadline;
- the request cannot be reported as successful until Azure timing and member
  state prove the required overlap.

### Thaw fails

- keep the ownership record;
- retry with bounded backoff;
- surface an urgent event;
- instruct the operator to inspect filesystem and storage health.

### Quota or throttling

- respect Azure `Retry-After`;
- never keep filesystems frozen while waiting for an hour-scale retry window;
- end the current attempt, thaw, and retry later from the beginning.

## 11. Interaction with other backup systems

AKS Backup can already run `sync` and `fsfreeze` through BackupHook.

Running both BackupHook filesystem freeze and CSI filesystem freeze creates:

- `EBUSY`/already-frozen results;
- unclear freeze ownership;
- risk of the wrong system thawing the filesystem;
- inconsistent timeout behavior.

The supported ownership rule is:

> Exactly one layer owns filesystem freeze and thaw for a snapshot operation.

When Approach 3 is selected:

- CSI owns filesystem freeze/thaw;
- AKS Backup application hooks may quiesce the application;
- AKS Backup must not separately execute `fsfreeze` for the same volumes.

## 12. Why Instant Access is out of scope

Instant Access applies only to application-consistent Restore Points for VMs
with Premium SSD v2 or Ultra data disks. It is not supported for
CrashConsistent Restore Points.

Because Approach 3 deliberately requests `CrashConsistent`, it cannot use
Instant Access.

If future requirements include Premium SSD v2, Ultra Disk, or Instant Access,
that requires a separate Azure-owned application-consistency design. It must
not be presented as a minor option on Approach 3.

## 13. Observability

Report at least:

- requested consistency: filesystem;
- Azure Restore Point consistency mode: crash;
- achieved end-to-end consistency: filesystem or not proven;
- source VM identity and orchestration type;
- selected and excluded disk counts;
- member freeze and thaw outcomes;
- time spent with application I/O blocked;
- Restore Point provisioning duration;
- Azure throttling and frequency-limit rejection;
- requested-versus-captured membership mismatch;
- valid snapshot with failed thaw;
- retention and quota pressure.

Avoid a single boolean named `consistent`. Record the layer:

```text
guestFilesystemConsistency: Succeeded|Failed|NotProven
azureStorageConsistency: CrashConsistent
groupMembershipConsistency: Succeeded|Failed
cleanupState: Succeeded|Degraded
```

## 14. Security and isolation

- Capture only requested disks.
- Exclude the node OS disk and unrelated data disks.
- Do not accept arbitrary freeze commands.
- Do not thaw a filesystem without durable ownership.
- Do not log application data, hook output, or credentials.
- Treat Restore Point encryption behavior as a launch-blocking security review.
- Require restored disks to specify the intended encryption configuration.
- Limit Azure permissions to VM/disk reads and Restore Point lifecycle
  operations.
- Prevent one namespace from selecting or coordinating another namespace's
  volumes without Kubernetes authorization.

## 15. Validation plan

Before approving the design, validate the following against Azure:

### Azure API validation

- OS disk exclusion works for every targeted VM topology.
- A local CrashConsistent Restore Point reaches `Succeeded` within an acceptable
  filesystem-freeze window.
- `Succeeded` is a safe thaw boundary.
- Restore Point source metadata contains stable source-disk-to-Disk-Restore-
  Point mapping.
- A VM storage profile change during creation produces detectable failure or
  membership mismatch.
- Delete ordering and eventual consistency are understood.

### Support-matrix validation

- standalone VM;
- VMSS Flexible instance used by AKS;
- VMSS Uniform rejection;
- Standard HDD/SSD, Premium SSD, and ZRS;
- encryption combinations intended for launch;
- maximum selected-disk count and resulting freeze duration.

### Consistency validation

For three filesystems:

1. continuously write one cross-disk sequence with checksums;
2. freeze all members;
3. create one Restore Point;
4. overwrite all source data;
5. restore all Disk Restore Points to new managed disks;
6. mount and verify:
   - filesystem cleanliness;
   - equal cross-disk sequence;
   - matching checksums;
   - no post-snapshot values.

Repeat with:

- controller restart;
- node restart;
- one member freeze failure;
- one member thaw failure;
- disk attach/detach attempt;
- Azure timeout/429;
- Restore Point quota/frequency rejection.

## 16. Alternatives considered

### Independent managed-disk snapshots while filesystems are frozen

This is the old group approach. It can preserve stable filesystem contents
while requests are submitted, but each disk uses a separate Azure snapshot
operation. Approach 3 uses one Restore Point instead.

### Azure application-consistent Restore Point without CSI freeze

This delegates guest consistency to VMSnapshot/VSS/Linux scripts. It is a valid
future application-consistency architecture, but it does not give CSI explicit
per-PVC freeze ownership and has different support, fallback, and operational
semantics.

### CSI freeze plus Azure application-consistent Restore Point

Rejected because both CSI and VMSnapshot can freeze or quiesce guest I/O.
Ownership and thaw behavior become ambiguous.

### Best-effort fallback to crash consistency

Rejected for the strict filesystem-consistent policy. A separate explicitly
named crash-consistent policy may create a Restore Point without freeze.

### Multiple Restore Points for a cross-VM group

Rejected because the result would not have one Azure multi-disk consistency
boundary.

## 17. Principal product risks

### 17.1 AKS topology compatibility

VMSS Uniform is unsupported. The design is useful only for AKS nodes represented
as supported individual VMs, such as qualifying VMSS Flexible instances.

### 17.2 One-hour minimum frequency

The documented crash-consistent minimum interval can be much longer than users
expect for Kubernetes snapshots. This can make Approach 3 unsuitable for
high-frequency backup objectives.

### 17.3 Freeze duration

The conservative safe boundary is Restore Point `Succeeded`. If actual Azure
latency exceeds the application's tolerable I/O pause, the design is not viable
without a documented earlier thaw point.

### 17.4 Reduced disk support

Premium SSD v2, Ultra Disk, write-accelerated disks, Azure Boost, and other
application-consistency-only configurations cannot use this approach.

### 17.5 Encryption

Azure documentation states that source encryption is not enabled on the
Restore Point. Security and compliance approval is required before launch.

### 17.6 VM-scoped primitive

A Kubernetes selector can choose PVCs across nodes, but one Restore Point cannot
capture multiple VMs. The product must make same-VM placement a documented
precondition.

## 18. Open design decisions

1. Are VMSS Flexible or standalone nodes sufficient for the target AKS use
   cases?
2. Is one Restore Point per VM per hour compatible with the required RPO?
3. What maximum I/O freeze duration can applications tolerate?
4. Can Azure document a safe thaw boundary earlier than
   `provisioningState=Succeeded`?
5. Which disk SKUs and encryption modes must be supported at launch?
6. Is excluding the OS disk supported for every target configuration?
7. What is the retention policy below Azure's 500-point-per-VM limit?
8. Where should Restore Point Collections live and who owns their lifecycle?
9. Is an unmounted selected filesystem considered already quiesced, or should
   strict mode reject it?
10. How will scheduling ensure that all selected PVCs are attached to the same
    supported VM?
11. How will users distinguish Azure `CrashConsistent` metadata from the
    end-to-end CSI filesystem-consistency guarantee?
12. Is preserving a valid snapshot acceptable when thaw fails, or must the
    snapshot be hidden until cleanup succeeds?

## 19. Recommendation

Proceed with Approach 3 only if the following feasibility gates pass:

1. The target AKS node pools use a Restore-Point-supported VM topology.
2. A one-hour minimum group-snapshot frequency meets product RPO requirements.
3. Local Restore Point completion fits within the allowed application I/O pause.
4. Required disk SKUs are supported by crash-consistent Restore Points.
5. Encryption behavior is approved by security/compliance.
6. Exact selected-disk capture, including OS-disk exclusion, is validated.

If any of these gates fail, do not weaken the consistency claim. Reconsider one
of:

- independent frozen managed-disk snapshots;
- an Azure-owned application-consistent Restore Point design;
- an application-level backup solution;
- a future Azure group-snapshot primitive with broader topology support.

## 20. References and additional resources analyzed

Primary resources:

- [Support matrix for VM restore points](https://learn.microsoft.com/en-us/azure/virtual-machines/concepts-restore-points)
- [Use virtual machine restore points](https://learn.microsoft.com/en-us/azure/virtual-machines/virtual-machines-create-restore-points)
- [Create Virtual Machine restore points with REST APIs](https://learn.microsoft.com/en-us/azure/virtual-machines/create-restore-points)

Additional linked resources:

- [VM Snapshot extension using VM restore points](https://learn.microsoft.com/en-us/azure/virtual-machines/virtual-machines-restore-points-vm-snapshot-extension)
- [Troubleshoot restore point failures](https://learn.microsoft.com/en-us/azure/virtual-machines/restore-point-troubleshooting)
- [Creating VM Restore Points using Azure CLI](https://learn.microsoft.com/en-us/azure/virtual-machines/virtual-machines-create-restore-points-cli)
- [Creating VM Restore Points using PowerShell](https://learn.microsoft.com/en-us/azure/virtual-machines/virtual-machines-create-restore-points-powershell)
- [Configure application-consistent backup for Azure Linux VMs](https://learn.microsoft.com/en-us/azure/backup/backup-azure-linux-app-consistent)
- [Restore Points Create REST API](https://learn.microsoft.com/en-us/rest/api/compute/restore-points/create?view=rest-compute-2025-04-01)
- [Restore Points Get REST API](https://learn.microsoft.com/en-us/rest/api/compute/restore-points/get?view=rest-compute-2025-04-01)
- [Restore Points Delete REST API](https://learn.microsoft.com/en-us/rest/api/compute/restore-points/delete?view=rest-compute-2025-04-01)
- [Restore Point Collections Create or Update REST API](https://learn.microsoft.com/en-us/rest/api/compute/restore-point-collections/create-or-update?view=rest-compute-2025-04-01)
- [Disk Restore Point Get REST API](https://learn.microsoft.com/en-us/rest/api/compute/disk-restore-point/get?view=rest-compute-2025-04-01)
- [Instantly access managed disk snapshots](https://learn.microsoft.com/en-us/azure/virtual-machines/disks-instant-access-snapshots)
- [Azure VM Restore Point ARM templates](https://github.com/Azure/Virtual-Machine-Restore-Points)
- [Azure VM backup and recovery options](https://learn.microsoft.com/en-us/azure/virtual-machines/backup-recovery)
