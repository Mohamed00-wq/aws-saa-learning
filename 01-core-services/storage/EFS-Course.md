# EFS Course  Elastic File System

## 1. Purpose

EFS is AWS's **managed, serverless shared file system** using **NFS** (Network File System). It grows and shrinks automatically to **petabyte scale** with zero capacity provisioning, and lets **many Linux instances across many AZs mount the same files simultaneously**. It's the shared-POSIX-storage answer when EBS's single-instance block model doesn't fit. PowerShell/SMB/Windows needs go to FSx, not EFS.

EBS → Block Storage → EC2 Disk
EFS → File Storage → Shared Files → NFS → Multiple EC2
S3 → Object Storage → Objects/Buckets

## 2. How it works

- **Protocol**: NFSv4.0 / NFSv4.1 mount at `/mnt/efs` via a DNS name (`fs-xxxx.efs.region.amazonaws.com`)
- **File system type**:
  - **Regional**  data replicated across **3+ AZs** (11 nines durability)  recommended for production
  - **One Zone**  stored in a single AZ (~47% cheaper, 99.9% durability)  dev/build/staging / re-creatable data
- **Mount targets**: one ENI per AZ in your VPC compute just mounts the DNS name
- **Security**: EFS Security Group must allow inbound **NFS TCP 2049** from the compute SG IAM + POSIX permissions control access
- **On-prem access**: via **Direct Connect / VPN** (from Linux servers), and via **EFS File Gateway** for hybrid POSIX
- Automatically scales capacity + throughput lifecycle management auto-tiers files between storage classes based on access

```
EC2/ECS/EKS/Lambda (AZ-A) ┐
EC2 (AZ-B)                ├── mount target per AZ ── EFS Regional (data across 3 AZs)
On-prem (DX/VPN)          ┘        → NFS 2049 → shared POSIX files
Lifecycle: Standard → (30d no access) → IA → (90d) → Archive Intelligent-Tiering back to Standard
```

## 3. When to use

- **Shared files across many EC2/ECS/EKS/Lambda/Fargate** clients at once
- **Multi-AZ shared storage**  web farms, CMS, shared home directories, container persistent volumes
- **Linux/POSIX workloads** needing file-level semantics over a mount, not object semantics
- **Automatic scaling**  unknown/unbounded growth to petabytes with no provisioning
- **Cost-tiered cold data**  lifecycle to IA/Archive Archive on Elastic throughput
- **Hybrid**  on-prem Linux access over Direct Connect/VPN EFS File Gateway for cached hybrid
- **Lambda/container persistent file storage** (shared across invocations/containers)

## 4. When NOT to use

- **Windows / SMB / Active Directory** needs → **FSx for Windows File Server**
- **HPC / Lustre sub-ms parallel** → **FSx for Lustre**
- **Single-instance block storage / boot volume** → **EBS** (or instance store for scratch)
- **Object data / static web serving over HTTP** → **S3**
- **OLTP databases needing very high sustained IOPS** → EBS io2/io1 (EFS is a file system, not a block DB device)
- **Very low-latency single-file hammering** → EBS/instance store (EFS throughput per client is limited)
- **Snapshots expected in the OS sense**  EFS uses AWS Backup, not native volume snapshots

## 5. Important features

- **Performance modes** (chosen at creation, cannot be changed later):
  - **General Purpose**  default, lowest per-op latency recommended for all workloads
  - **Max I/O**  previous-gen for highly parallelized workloads higher aggregate throughput but higher latency **not available on One Zone or Elastic throughput**
- **Throughput modes** (can be changed later):
  - **Elastic** (default, recommended)  auto-scales with workload pay for what you use scales up to **3 GiB/s read / 1 GiB/s write** (1,500 MiB/s combined for v2 EFS clients/CSI driver) supports Archive
  - **Provisioned**  set a fixed MiB/s independent of size good for steady high-throughput changing has a 24-hour cooldown to lower it
  - **Bursting**  throughput ≈ 50 MiB/s per TiB, burst to 100 MiB/s/TiB using **burst credits** watch `BurstCreditBalance` (throttles to baseline when exhausted)
- **Storage classes** (auto-tiered via lifecycle):
  | Class | Use | Latency | Min |
  |---|---|---|---|
  | **Standard** | Active, frequently accessed (first write destination) | Sub-ms |  |
  | **Infrequent Access (IA)** | Accessed a few times/quarter  up to **95% cheaper** | Tens of ms | 30 days, 128 KiB/file |
  | **Archive** | Few times/year or less  up to **~72% cheaper than IA** | Tens of ms | 90 days, 128 KiB/file **Elastic throughput only** |
- **EFS Lifecycle Management**  default policy: Standard → IA after **30 days** no access, → Archive after **90 days**. **Intelligent-Tiering** also moves files **back to Standard** on access
- **AWS Backup**  automated backup/versioning (EFS has no native snapshots)
- **Cross-Region replication** for DR/compliance
- **Access points**  app-specific entry points with own POSIX user/group + root directory great for Lambda/containers
- **Works with** EC2, ECS, EKS, Lambda, Fargate encryption at rest (KMS) + in transit (TLS)

## 6. Limitations

- **Linux only**  NFS/POSIX no SMB/Windows (→ FSx for Windows)
- **Performance mode fixed at creation**  can't switch General Purpose ↔ Max I/O
- **Encryption at rest can't be enabled later**  must re-create and migrate (encryption at creation only)
- **File system type fixed**  Regional vs One Zone chosen at creation
- **Burst credits exhaust**  sustained high throughput throttles to baseline move to Elastic/Provisioned
- **IA/Archive access charges**  cheap storage, but every read of tiered data costs min billing per file 128 KiB
- **Archive restrictions**  only on Regional + Elastic throughput can't switch to Bursting/Provisioned once data is in Archive
- **One Zone ≠ 11 nines**  99.9% durability, data can be lost if the AZ fails
- **Elastic metering minimums**  each I/O metered at 32 KiB minimum many tiny ops cost more than expected
- **No native snapshots**  use AWS Backup per-client throughput limits can bottleneck single-file workloads

## 7. Trade-offs

- **Regional vs One Zone**  multi-AZ 11-nines vs ~47% cheaper single-AZ (use only for re-creatable/dev data)
- **Elastic vs Provisioned vs Bursting**  pay-for-use & auto (spiky/unknown) vs fixed for steady high throughput vs credits that throttle when exhausted
- **General Purpose vs Max I/O**  low latency, broad use vs high parallel throughput at higher latency (legacy)
- **Standard vs IA vs Archive**  speed/first-write vs cheap-rare-access vs coldest access fees apply when reading tiered files
- **EFS vs EBS**  shared multi-AZ POSIX (file) vs single-instance low-latency block EFS scales automatically, EBS caps per volume
- **EFS vs S3**  POSIX mount/latency vs object/HTTP/archive S3 is cheaper at scale for static data
- **EFS vs FSx**  generic Linux NFS vs purpose-built (Windows SMB, Lustre HPC, NetApp ONTAP, OpenZFS)
- **Lifecycle tiering vs always-Standard**  lower storage cost vs access charges + retrieval latency for cold files

## 8. Architecture

Reference shared-storage patterns:

```
Shared web tier (multi-AZ):
  ALB → ASG [EC2 AZ-A, AZ-B] ── mount EFS Regional /mnt/efs (NFS 2049, SG→SG)
     → content/uploads shared across all instances & AZs
     → lifecycle: Standard → IA(30d) → Archive(90d) + AWS Backup

Containers/serverless:
  ECS/EKS tasks & Lambda → EFS access points (POSIX per app)
     → persistent shared state across tasks/invocations

Hybrid:
  On-prem Linux farm ── DX/VPN ── EFS mount targets (NFS)
  (or EFS File Gateway for cached local + cloud)
```

## 9. SAA-C03 Perspective

EFS shows up in storage/resilience/cost questions  know the decision rules:

- **"Shared file storage for Linux EC2 across AZs"** → **EFS** (never EBS)
- **"Windows shared drive / SMB / AD"** → **FSx for Windows**, not EFS
- **"HPC / Lustre / sub-ms parallel"** → **FSx for Lustre**
- **Performance mode**: General Purpose default Max I/O only for massive parallelism (and not on One Zone/Elastic)
- **Throughput modes**: Elastic (default, spiky) / Provisioned (steady high) / Bursting (credits → throttling)
- **Storage classes + lifecycle**: Standard → IA (30d) → Archive (90d), Intelligent-Tiering back to Standard IA up to 95% cheaper
- **Port 2049** security group rule mount targets per AZ
- **Regional vs One Zone** for durability/cost trade-off
- **Encryption at creation only** AWS Backup (no native snapshots) cross-region replication for DR
- Access points for per-app POSIX contexts (containers/Lambda)

Exam trap: "shared storage among multiple Linux web servers in different AZs" → **EFS Regional**. "Cheapest shared dev environment" → **EFS One Zone**. "Windows file share" → **FSx for Windows**. "Cannot enable encryption later" is true for EFS  a regular gotcha.