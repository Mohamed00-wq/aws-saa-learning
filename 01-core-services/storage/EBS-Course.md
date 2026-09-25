# EBS Course  Elastic Block Store

## 1. Purpose

EBS provides **persistent block-level storage volumes** for EC2  like virtual hard drives. A network-attached SSD/implicit-HDD volume you attach to an EC2 instance, format with a filesystem, and use like a local disk. It survives instance stop/start and (if configured) termination, so your data persists independent of the VM. It's the storage engine behind boot volumes and databases on EC2.

## 2. How it works

- Volumes are **AZ-scoped**  created in an AZ and attachable only to instances **in that same AZ**
- Attach via the network (ENI/Nitro) the instance formats it (ext4/XFS/NTFS)
- **Elastic Volumes**: resize, change type, or adjust IOPS **live** without detaching/rebooting
- **Snapshots**: point-in-time backups  **incremental** (only changed blocks), stored in S3, region-scoped can be copied cross-region, shared cross-account, or archived (Snapshot Archive, ~24–72 h restore)
- **DeleteOnTermination**: root volume defaults true (wiped on instance termination) non-root defaults false (kept)  an exam classic
- **Encryption**: KMS-backed at rest + in transit + embedded in snapshots can only be enabled at **creation** (add retroactively via an encrypted snapshot clone)

```
Create volume (AZ-A, gp3, 100 GiB, encrypted) → attach to EC2 (same AZ)
  → format → mount → read/write blocks over network
  → snapshot (incremental → S3) → copy/restore/FSR for DR
Retention: volume continues to bill even when instance is stopped (not terminated)
```

## 3. When to use

- **Boot volumes** for EC2 instances (root OS)
- **Databases and transactional workloads** needing consistent, low-latency block I/O (RDS runs on EBS underneath for self-managed DBs on EC2 it's EBS too)
- **Stateful data that must survive instance stop/start or replacement** (ASG leaves volumes, ephemeral state is elsewhere)
- **File server / apps needing local-latency storage** vs network object/file services
- Where performance/latency matters and **snapshot backup/DR** is the recovery mechanism

## 4. When NOT to use

- **Shared access across many instances**  EBS is single-instance (except io1/io2 Multi-Attach) → use **EFS** (NFS) or **FSx**
- **Serving files to many clients over HTTP**  that's **S3** (object storage)
- **Cross-AZ/region access**  EBS volumes are AZ-locked use S3/EFS (regional) or snapshots + re-create
- **Ephemeral, scratch, high-throughput local caching**  instance store is faster (physically attached, but data lost on stop/terminate)
- **Mass unstructured archive**  S3 + Glacier tiers are far cheaper
- Very large capacities in one file system spanning AZs → EFS scales to petabytes, EBS caps at 64 TiB

## 5. Important features

- **Volume types** (the exam's table):
  | Type | Max IOPS (Input/Output Operations Per Second) | Max throughput | Best for |
  |---|---|---|---|
  | **gp3** (default) | 16,000 | 1,000 MB/s | Boot, most apps, dev/test  IOPS decoupled from size (baseline 3,000 IOPS + 125 MB/s, scale independently) |
  | **gp2** (legacy) | 16,000 | 250 MB/s | Legacy  IOPS tied to GB (3 IOPS/GB) |
  | **io1** | 64,000 | 1,000 MB/s | Provisioned-IOPS DBs (99.8–99.9% durability) |
  | **io2 / io2 Block Express** | 64k / **256,000** | 1,000 / **4,000 MB/s** | Mission-critical (Oracle, SAP HANA) **99.999% durability**, sub-ms latency (Nitro) Multi-Attach |
  | **st1** (throughput HDD) | 500 | 500 MB/s | Big data, log processing  **can't boot** |
  | **sc1** (cold HDD) | 250 | 250 MB/s | Cold bulk data  **can't boot**, cheapest |
- **Snapshots**: incremental, low-cost backups **Fast Snapshot Restore (FSR)** = immediate full performance on restore (per-AZ, billed hourly) **Recycle Bin** retains deleted snapshots
- **Multi-Attach**  io1/io2 to up to **16 Nitro instances in the same AZ**, needs a **cluster-aware filesystem** (ext4/XFS won't do  e.g. GFS2/OCFS2/NVMe reservations)
- **Elastic Volumes**  resize/increase IOPS online, no downtime
- **Data Lifecycle Manager / AWS Backup**  automate snapshots with retention
- **Dedicated throughput**: EBS-optimized instances deliver provisioned IOPS (Nitro needed for top end)

## 6. Limitations

- **AZ-locked**  Number‑1 constraint: a volume can only attach to instances in the same AZ (cross-AZ requires snapshot→recreate)
- **Billed while stopped**  GB-month (and IOPS for io1/io2) continues even when the instance isn't running (terminate or snapshot to save)
- **Encryption can't be added after the fact**  must create new via encrypted snapshot and migrate
- **HDD types can't be boot volumes** | **No automatic backups by default**  snapshots are opt-in
- **16 TiB standard / 64 TiB max** (io2 Block Express) per volume
- **Multi-Attach is narrow**  io1/io2 only, same AZ, clustered FS, concurrency handled at app level
- **Root volume wiped on termination by default**  delete-on-termination=true is easy to brick data with
- Initial lazy-load from snapshots degrades performance until blocks are read (mitigate with FSR)

## 7. Trade-offs

- **gp3 vs io2**  cheaper/default and good enough (16k IOPS) vs premium latency/IOPS/durability when the workload demands it
- **gp3 vs gp2**  IOPS/throughput independent of size vs size-proportional gp3 replaces gp2
- **SSD (gp/io) vs HDD (st/sc)**  transactional low-latency IOPS vs bulk sequential throughput/cost HDD can't boot
- **EBS vs Instance Store**  persistent, re-attachable, snapshot-able vs faster physically-attached but temporary (lost on stop/terminate)
- **EBS vs EFS vs S3**  block (1 instance, A-Z, low latency) vs shared file (many instances/AZs, POSIX) vs object (global/http, archiving)
- **More snapshots vs FSR**  cheap incremental backups with restore latency vs immediate-full-perf restore billed per-AZ-hour
- **Single volume (scale up) vs RAID/EBS multi-attach**  deep capacity in one volume vs bandwidth/redundancy splitting

## 8. Architecture

Reference common EBS patterns:

```
Stateless app + stateful data:
  Web tier: EC2 boot volumes (gp3), root delete-on-termination=true, stateless app
  DB tier (self-managed): EC2 + io2 Multi-Attach (cluster-aware) OR single gp3/io2
   → snapshot via Data Lifecycle Manager/AWS Backup → cross-region copy (DR)
   → FSR enabled in standby AZ for fast failover restore

ASG-friendly state:
  App writes state → EBS snapshot → lifecyle to S3/Glacier for archive
  (or put state in EFS for multi-AZ shared access instead of EBS)
```

## 9. SAA-C03 Perspective

EBS is core exam material (Resilient + High-Performing + Cost domains):

- **AZ-locked**  the recurring constraint cross-AZ = snapshot + recreate (or EFS/S3 for shared)
- **Volume types**: pick gp3 (default), io2/io2-BE (top DB performance/durability), st1/sc1 (throughput/cold, **can't boot**), Multi-Attach → **only io1/io2, same AZ, clustered FS**
- **Snapshots**: incremental → S3 **cross-region copy** for DR **FSR** for fast restore Recycle Bin for delete protection
- **DeleteOnTermination**  root vs non-root default trap questions
- **Encryption at creation only**  "encrypt existing unencrypted volume" → snapshot → new encrypted volume
- **Elastic Volumes**  online resize as cost/performance flexibility answer
- **Instance stop still bills EBS**  cost optimization: terminate + snapshot, or use instance store for scratch
- Works with AMIs, EBS-backed golden images, and multi-AZ failover via snapshots

Exam trap: "move EBS volume to another AZ quickly" → **snapshot + restore in new AZ** (never "attach cross-AZ"). "Shared storage between web servers" → EFS, not EBS. "Attach a volume to multiple instances" → only io1/io2 Multi-Attach with cluster FS  gp3 can't.