# EBS — Elastic Block Store

## What it is

Elastic Block Store provides **persistent, block-level storage volumes** for use with EC2 instances. Unlike instance store (which is ephemeral and physically attached), EBS volumes are network-attached and survive instance stop/start cycles. EBS is the default storage layer for EC2 and is the backbone of most persistent workloads: databases, file systems, application state, and boot volumes.

EBS is an **Availability Zone-scoped** service — a volume exists in a single AZ and can only be attached to instances in that same AZ. To move data between AZs or regions, you use snapshots.

---

## Key concepts

### Volume types

Understanding volume types is critical for the exam — the questions test whether you can match workload characteristics to the right type.

**SSD-based (low latency, high IOPS):**

| Type | Baseline performance | Burst | Best for | Notes |
|---|---|---|---|---|
| **gp3** | 3,000 IOPS, 125 MB/s | None needed — baseline is always available | General-purpose workloads, boot volumes, most databases | IOPS and throughput can be provisioned **independently** of volume size. Cheaper than gp2 for equivalent performance. |
| **gp2** | 3 IOPS per GB (up to 16,000), burstable to 3,000 IOPS | Credit-based bursting | Legacy general-purpose | IOPS scale with size (a 100 GB gp2 gets 300 baseline IOPS). A 1 TB gp2 gets 3,000 baseline. Being replaced by gp3. |
| **io2** | Up to 64,000 IOPS per volume | None — always provisioned | Mission-critical databases, latency-sensitive workloads | IOPS and volume size are independent. 99.999% durability. More expensive than gp3. |
| **io2 Block Express** | Up to 256,000 IOPS | None — always provisioned | Ultra-high performance databases (SAP HANA, Oracle) | Uses the Block Express protocol. Only supported on specific instance types. |

**HDD-based (high throughput, higher latency):**

| Type | Throughput | Best for | Notes |
|---|---|---|---|
| **st1** | Up to 500 MB/s | Big data, data warehouse, log processing | Cannot be a boot volume. Throughput-optimized. |
| **sc1** | Up to 250 MB/s | Infrequently accessed data, cold storage | Cannot be a boot volume. Lowest cost. |

**Exam decision framework:**
- Need low latency + high IOPS? → **gp3** (general) or **io2** (extreme)
- Need high throughput for sequential reads? → **st1**
- Need cheap storage for data you rarely access? → **sc1**
- Boot volume? → **gp3** (HDD types cannot boot)
- Database? → **gp3** for most, **io2** for latency-critical

### Key characteristics

**AZ-locked:**
- An EBS volume lives in exactly one AZ. It can only be attached to an EC2 instance in the same AZ.
- To move data to another AZ: take a snapshot → create a new volume from the snapshot in the target AZ.
- To move data to another region: copy the snapshot to the target region → create a new volume from the copied snapshot.

**Elastic Volumes (live modification):**
- You can change the size, type, and IOPS/throughput of an EBS volume **while it is in use** — without detaching it or stopping the instance.
- This works by modifying the volume metadata in the background. The OS needs to recognize the new size (e.g. by running a file system resize command), but the EBS service handles the underlying change.
- Useful for scaling up a database volume or switching from gp2 to gp3 without downtime.

**Snapshots:**
- Point-in-time backups stored in S3 (managed by AWS — you don't access them directly).
- **Incremental:** the first snapshot is a full copy. Subsequent snapshots store only the blocks that changed since the last snapshot. This makes frequent snapshots fast and storage-efficient.
- Snapshots are **region-scoped** — they exist in a specific region but can be copied to other regions.
- When you create a volume from a snapshot, the volume is created in the same region as the snapshot (not necessarily the same AZ — you choose the AZ).
- **Snapshot lifecycle:** you can set lifecycle policies to auto-delete snapshots after a retention period, or auto-copy snapshots to another region for DR.
- **Fast Snapshot Restore (FSR):** normally, a volume created from a snapshot is lazily initialized (blocks are loaded on first access, causing initial latency). FSR pre-initializes all blocks in the snapshot so the resulting volume has full performance from the start.

**Encryption:**
- EBS encryption uses **AWS KMS** (by default the `aws/ebs` managed key, or a customer-managed KMS key).
- Encrypts: data at rest on the volume, data in transit between the instance and EBS, all snapshots created from the volume, and volumes created from those snapshots.
- Encryption is set at volume creation time — you cannot retroactively encrypt an existing unencrypted volume. To encrypt an existing volume, create an encrypted snapshot and then create a new volume from it.
- All snapshots of an encrypted volume are also encrypted, and all volumes created from those snapshots are also encrypted — encryption propagates through the snapshot chain.

### EBS vs Instance Store

| Property | EBS | Instance Store |
|---|---|---|
| **Persistence** | Survives stop/start/terminate (if not deleted) | Data lost on stop/terminate |
| **Attachment** | Network-attached | Physically attached to the host |
| **Performance** | High IOPS, but network latency | Very high IOPS, no network latency |
| **Billing** | Billed separately per GB-month | Included in instance price |
| **Use cases** | Boot volumes, databases, general storage | Temporary scratch, caches, buffers |
| **Availability** | Any instance type | Only specific instance types with local NVMe |

**Exam note:** instance store is sometimes called "ephemeral storage." If a question says "the data was lost when the instance was stopped," the answer is almost always "use EBS instead."

### Multi-Attach

- io1 and io2 volumes support **Multi-Attach** — a single volume can be attached to multiple instances in the **same AZ** simultaneously.
- All instances must use a clustered file system (e.g. GFS2) that supports concurrent multi-writer access. Standard ext4/XFS do NOT support this.
- Use case: clustered databases (Oracle RAC), clustered applications that need concurrent access to shared storage.
- Multi-Attach is **not** supported for gp2, gp3, st1, or sc1.

---

## Architecture deep dive

### How EBS works under the hood

1. When you create an EBS volume, AWS allocates storage capacity in the specified AZ and presents it to your instance as a block device.
2. The instance sees the volume as a raw block device (e.g. `/dev/xvda` for root, `/dev/xvdf` for additional volumes).
3. The OS must format the volume with a file system (ext4, XFS, NTFS, etc.) before it can store files — unless you're using it as raw block storage for a database.
4. Communication between the instance and EBS happens over the AWS network backbone, not over the internet. EBS traffic uses a dedicated network separate from instance public traffic.
5. EBS volumes are replicated within an AZ for durability. io2 volumes offer 99.999% durability; gp3 offers 99.8%–99.9% (varies by size).

### Snapshot mechanics

Snapshots are incremental, but the "increment" is at the **block level**, not the file level. AWS tracks which 512 KB blocks have changed between snapshots. This means:
- A 1 TB volume with only 1 GB changed will produce a ~1 GB snapshot (after the first full snapshot).
- Deleting a snapshot only removes the blocks unique to that snapshot — if a block is referenced by a newer snapshot, it is kept.
- The first snapshot is the only full copy; all subsequent snapshots are deltas.

### Volume initialization and pre-warming

When you restore an EBS volume from a snapshot, the volume is created instantly but the data is loaded lazily. Until each block is accessed and loaded, reads to that block have higher latency.

For gp2/gp3/io1/io2: AWS automatically loads blocks from the snapshot in the background after creation. This process can take up to 12 hours for large volumes. Until it completes, the volume has inconsistent performance.

For full performance immediately: use **Fast Snapshot Restore (FSR)** or force initialization by reading all blocks (e.g. `dd` or reading every file).

### Root volume behavior

- The root EBS volume is the boot volume — it contains the OS.
- **DeleteOnTermination:** by default, the root volume is deleted when the instance is terminated (`true`). You can change this to `false` to preserve the data after termination.
- Non-root EBS volumes default to `DeleteOnTermination = false` — they persist after termination and must be deleted manually.
- To preserve the root volume: set `DeleteOnTermination = false` before terminating, or snapshot it first.

---

## Exam domain(s)

- [ ] Design Secure Architectures (30%)
- [x] **Design Resilient Architectures (26%)** — snapshots for backup/DR, cross-AZ/region volume recovery, Multi-Attach for HA
- [x] **Design High-Performing Architectures (24%)** — volume type selection, IOPS/throughput tuning, FSR for fast restore
- [x] **Design Cost-Optimized Architectures (20%)** — choosing gp3 over io2 when possible, using st1/sc1 for cold data, snapshot lifecycle policies

---

## Advanced gotchas & edge cases

1. **EBS volumes are AZ-scoped.** You cannot attach a volume to an instance in a different AZ. This is the #1 EBS constraint on the exam.

2. **gp3 decouples IOPS from size.** Unlike gp2 (where IOPS = 3 × size in GB), gp3 gives you 3,000 IOPS regardless of volume size. You can add up to 16,000 IOPS and 1,000 MB/s throughput independently for additional cost.

3. **Encryption cannot be added retroactively.** You must create an encrypted snapshot of an unencrypted volume, then create a new encrypted volume from that snapshot.

4. **Snapshots of encrypted volumes are always encrypted.** You cannot create an unencrypted snapshot from an encrypted volume.

5. **Deleting a snapshot of an encrypted volume does not compromise data.** AWS handles the key management; deleting one snapshot does not expose unencrypted data.

6. **HDD types (st1, sc1) cannot be root volumes.** If a question asks about a cold data volume or a throughput-optimized big-data volume, st1 or sc1 is the answer. If it needs to boot, it must be SSD.

7. **io2 vs io2 Block Express:** io2 Block Express supports up to 256,000 IOPS and is only available on specific instance types (e.g. z1d, r5b, m5d). Regular io2 supports up to 64,000 IOPS.

8. **Multi-Attach requires a clustered file system.** Using Multi-Attach with a standard file system (ext4, XFS without GFS2) will cause data corruption. The exam tests this nuance.

9. **Fast Snapshot Restore is per-AZ.** You enable FSR for a snapshot in a specific AZ. Volumes created from that snapshot in that AZ get full performance immediately. Other AZs are unaffected.

10. **EBS does not automatically back up data.** Snapshots are opt-in. If you don't create snapshots or enable lifecycle policies, there is no automatic backup.

---

## Exam-style questions

**Q1.** A company wants to migrate an EBS volume's data from us-east-1a to us-east-1b. What is the correct approach?
- A) Detach the volume and reattach it in us-east-1b
- B) Create a snapshot, then create a new volume from the snapshot in us-east-1b
- C) Use EBS replication to mirror the volume cross-AZ
- D) Create a clone of the volume using EBS Multi-Attach

<details><summary>Answer</summary>
**B** — EBS volumes cannot move between AZs directly. The standard approach is snapshot → create new volume from snapshot in the target AZ. EBS does not have built-in cross-AZ replication (C). Multi-Attach (D) keeps multiple instances in the same AZ on the same volume, not cross-AZ.
</details>

**Q2.** True or False: You must stop an EC2 instance to increase the size of its attached gp3 EBS volume.
- A) True — resizing requires downtime
- B) False — Elastic Volumes allows live modification of size, type, and IOPS
- C) True — but only for root volumes
- D) False — but the instance needs to be rebooted

<details><summary>Answer</summary>
**B** — Elastic Volumes enables on-the-fly modifications (size, type, IOPS) without stopping the instance or detaching the volume. The OS may need a file system resize to recognize additional space, but the EBS modification itself is live.
</details>

**Q3.** Which EBS volume type should a Solutions Architect choose for a NoSQL database requiring consistent sub-millisecond latency and 40,000 provisioned IOPS?
- A) gp3
- B) st1
- C) io2
- D) sc1

<details><summary>Answer</summary>
**C** — io2 is designed for the most demanding low-latency, high-IOPS workloads. gp3 maxes out at 16,000 IOPS. st1 and sc1 are HDD-based with higher latency.
</details>

**Q4.** An engineer creates an AMI from a running instance and then deregisters the AMI. They also delete all snapshots associated with the AMI. What happens to instances already running from that AMI?
- A) They are terminated immediately
- B) They continue to run normally — they are independent copies
- C) They lose their root volume
- D) They can no longer be stopped and restarted

<details><summary>Answer</summary>
**B** — once an instance is launched, it is an independent copy. Deleting the AMI and snapshots does not affect running instances. The snapshot deletion only prevents launching new instances from that AMI.
</details>

**Q5.** A Solutions Architect needs to create a 500 GB EBS volume for a cold data archive that is accessed less than once per month. The workload is not performance-sensitive. Which volume type minimizes cost?
- A) gp3
- B) io2
- C) st1
- D) sc1

<details><summary>Answer</summary>
**D** — sc1 (Cold HDD) is the lowest-cost EBS option, designed specifically for infrequently accessed data. st1 is for throughput-optimized workloads that are accessed regularly. gp3 and io2 are SSD-based and more expensive for cold data.
</details>

**Q6.** An engineer restores a 2 TB gp3 volume from a snapshot and immediately runs a latency-sensitive application. The application experiences higher-than-expected read latency for the first several hours. What is the MOST likely cause?
- A) The gp3 volume was created with the wrong IOPS setting
- B) The volume data is being lazily loaded from the snapshot (initialization)
- C) The instance type does not support gp3
- D) The snapshot was corrupted

<details><summary>Answer</summary>
**B** — volumes restored from snapshots are lazily initialized — blocks are loaded from the snapshot on first access. Until initialization completes, reads to unloaded blocks have higher latency. Fast Snapshot Restore (FSR) or forced initialization (reading all blocks) eliminates this.
</details>

---

## Related services

- [[EC2]] — EBS volumes are attached to EC2 instances as block devices
- [[AMI]] — EBS-backed AMIs are built from EBS snapshots; launching an AMI creates new EBS volumes
- [[S3]] — snapshots are stored in S3 (managed by AWS); S3 can also be used as an alternative storage tier (EFS, FSx)
- [[KMS]] — EBS encryption uses KMS keys
- [[Auto-scaling]] — launch templates define which EBS volumes are created with new instances
- [[AWS-Backup]] — provides centralized, policy-based EBS snapshot management
