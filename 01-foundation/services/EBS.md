![EBS](/images/icons/arch/Arch_Amazon-Elastic-Block-Store_64.svg)

# EBS — Elastic Block Store

## What it is

Elastic Block Store provides **persistent, block-level storage volumes** for EC2. Network-attached, survives stop/start. AZ-scoped — single AZ, attach only to instances in that AZ. Move data between AZs/regions via snapshots.

## Volume types

**SSD (low latency, high IOPS):**

| Type | Perf | Best for | Notes |
|---|---|---|---|
| **gp3** | 3,000 IOPS, 125 MB/s | General, boot, most DBs | IOPS/throughput independent of size (up to 16k IOPS / 1,000 MB/s). Cheaper than gp2. |
| **gp2** | 3 IOPS/GB, burstable to 3,000 | Legacy general | IOPS scale with size. Being replaced by gp3. |
| **io2** | up to 64,000 IOPS | Mission-critical DBs | 99.999% durability. Supports **Multi-Attach**. |
| **io2 Block Express** | up to 256,000 IOPS | SAP HANA, Oracle | Specific instance types only. |

**HDD (high throughput, higher latency):**

| Type | Throughput | Best for | Notes |
|---|---|---|---|
| **st1** | up to 500 MB/s | Big data, DWH, logs | **Cannot be a boot volume**. |
| **sc1** | up to 250 MB/s | Cold/infrequent data | Lowest cost. Cannot boot. |

## Key characteristics

- **Elastic Volumes**: resize/change type/IOPS **live** while in use.
- **Snapshots**: incremental, point-in-time, S3-backed, **region-scoped**. Auto lifecycle policies.
- **Fast Snapshot Restore (FSR)**: full performance immediately on restore (per-AZ).

## Encryption

Uses **KMS**. Encrypts at rest + in transit + snapshots. **Cannot retroactively encrypt** — make encrypted snapshot, then new volume.

## EBS vs Instance Store

| | EBS | Instance Store |
|---|---|---|
| Persistence | Survives stop/start | Lost on stop/terminate |
| Attachment | Network | Physically attached |
| Latency | Network latency | No network latency |
| Use | Boot, DBs, general | Scratch, caches, buffers |

## Multi-Attach & Root volume

- **io1/io2 only**, same AZ, multiple instances. Requires **clustered filesystem** (GFS2).
- Root volume deleted on termination by default; **non-root volumes preserved**.

## Exam domains

- [ ] Secure (30%)
- [x] **Resilient (26%)** — snapshots for backup/DR, cross-AZ/region recovery
- [x] **High-Performing (24%)** — volume type selection, IOPS tuning, FSR
- [x] **Cost-Optimized (20%)** — gp3 over io2 where possible, st1/sc1 for cold

## Key gotchas

1. **EBS is AZ-scoped** — #1 exam constraint
2. **gp3 decouples IOPS from size** (gp2 IOPS = 3×GB)
3. **Encryption can't be added retroactively** — via encrypted snapshot
4. Snapshots of encrypted volumes are **always encrypted**
5. **HDD types can't be boot volumes**
6. **Multi-Attach requires clustered filesystem** (GFS2), io1/io2 only
7. **No automatic backup** — snapshots are opt-in

## Related services

- [[EC2]] — EBS attaches as block devices
- [[AMI]] — EBS-backed AMIs from snapshots
- [[KMS]] — EBS encryption
- [[Auto-scaling]] — launch templates define EBS volumes
