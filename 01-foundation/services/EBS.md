![EBS](/images/icons/arch/Arch_Amazon-Elastic-Block-Store_64.svg)

# EBS — Elastic Block Store

## What it is

Elastic Block Store provides **persistent, block-level storage volumes** for use with EC2 instances. Unlike instance store (which is ephemeral and physically attached), EBS volumes are network-attached and survive instance stop/start cycles. EBS is the default storage layer for EC2 and is the backbone of most persistent workloads: databases, file systems, application state, and boot volumes.

EBS is an **Availability Zone-scoped** service — a volume exists in a single AZ and can only be attached to instances in that same AZ. To move data between AZs or regions, you use snapshots.

## Volume types

**SSD (low latency, high IOPS):**

| Type | Perf | Best for | Notes |
|---|---|---|---|
| **gp3** | 3,000 IOPS, 125 MB/s | General, boot, most DBs | IOPS/throughput provisioned **independently of size** (up to 16,000 IOPS / 1,000 MB/s). Cheaper than gp2. |
| **gp2** | 3 IOPS/GB, burstable to 3,000 | Legacy general | IOPS scale with size (1 TB = 3,000). Being replaced by gp3. |
| **io2** | up to 64,000 IOPS | Mission-critical DBs, latency-sensitive | 99.999% durability. Supports **Multi-Attach**. |
| **io2 Block Express** | up to 256,000 IOPS | SAP HANA, Oracle | Only on specific instance types. |

**HDD (high throughput, higher latency):**

| Type | Throughput | Best for | Notes |
|---|---|---|---|
| **st1** | up to 500 MB/s | Big data, DWH, logs | **Cannot be a boot volume**. |
| **sc1** | up to 250 MB/s | Cold/infrequent data | Lowest cost. Cannot boot. |

**Decision:** latency+IOPS → gp3/io2 · sequential throughput → st1 · rare cold data → sc1 · boot → SSD · DB → gp3, latency-critical → io2.

## Key characteristics

- **AZ-locked**: move via snapshot → new volume in target AZ; copy snapshot cross-region.
- **Elastic Volumes**: resize/change type/IOPS **live** while in use, no downtime (OS may need filesystem resize).
- **Snapshots**: incremental, point-in-time, stored in S3; **region-scoped**. Auto lifecycle policies for delete/cross-region copy.
- **Fast Snapshot Restore (FSR)**: pre-initializes all blocks so restored volume has full performance immediately (per-AZ).

## Encryption

Uses **KMS** (default `aws/ebs` or custom key). Encrypts data at rest + in transit + all snapshots + volumes from those snapshots. **Cannot retroactively encrypt** — make an encrypted snapshot, then a new volume from it.

## EBS vs Instance Store

| | EBS | Instance Store |
|---|---|---|
| Persistence | Survives stop/start/terminate | Lost on stop/terminate |
| Attachment | Network | Physically attached |
| Latency | Network latency | No network latency |
| Billing | Per GB-month | Included in instance price |
| Use | Boot, DBs, general | Scratch, caches, buffers |

**Exam:** "data lost when stopped" → use EBS instead.

## Multi-Attach

- **io1/io2 only**, same AZ, multiple instances.
- Requires a **clustered file system** (GFS2) — ext4/XFS will corrupt.
- Use case: Oracle RAC, clustered apps.

## Root volume behavior

Root volume deleted on termination by default (`DeleteOnTermination=true`); **non-root volumes preserved by default**. To keep root: revert the flag or snapshot first.

## Exam domains

- [ ] Secure (30%)
- [x] **Resilient (26%)** — snapshots for backup/DR, cross-AZ/region recovery, Multi-Attach HA
- [x] **High-Performing (24%)** — volume type selection, IOPS tuning, FSR
- [x] **Cost-Optimized (20%)** — gp3 over io2 where possible, st1/sc1 for cold, snapshot lifecycle

## Key gotchas

1. **EBS is AZ-scoped** — the #1 exam constraint
2. **gp3 decouples IOPS from size** (3,000 IOPS regardless of size; gp2 IOPS = 3×GB)
3. **Encryption can't be added retroactively** — via encrypted snapshot
4. Snapshots of encrypted volumes are **always encrypted**
5. **HDD types (st1/sc1) can't be root/boot volumes**
6. io2 Block Express supports 256,000 IOPS (vs 64,000 for io2)
7. **Multi-Attach requires clustered filesystem** (GFS2), io1/io2 only
8. **FSR is per-AZ**
9. **No automatic backup** — snapshots are opt-in


## Related services

- [[EC2]] — EBS attaches as block devices
- [[AMI]] — EBS-backed AMIs come from snapshots; launching creates EBS volumes
- [[S3]] — snapshot storage (managed by AWS)
- [[KMS]] — EBS encryption
- [[Auto-scaling]] — launch templates define EBS volumes
- [[AWS-Backup]] — policy-based snapshot management
