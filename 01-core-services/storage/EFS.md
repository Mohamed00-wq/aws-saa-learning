# EFS — Amazon Elastic File System

## What it is

EFS is AWS's **managed shared file system** (NFS). Serverless — it grows and shrinks automatically to **petabyte scale** with no capacity provisioning. Built for multiple Linux EC2 instances across multiple AZs to mount the same files simultaneously. **Linux/POSIX only** — if the scenario needs Windows, it's FSx, not EFS.

## Key characteristics

- **Protocol**: NFSv4.0 / NFSv4.1. Mounted at `/mnt/efs` via DNS name.
- **File system types**: **Regional** (data replicated across 3+ AZs — recommended) vs **One Zone** (single AZ, ~45% cheaper, lower durability).
- **EBS vs EFS**: EBS = block storage attached to one EC2 in one AZ; EFS = shared NFS accessible from many instances across AZs.
- **Mount targets**: one per AZ in the VPC; EFS Security Group must allow inbound NFS on **TCP 2049** from the EC2 Security Group. On-prem access via **Direct Connect / VPN**.
- **Encryption at rest**: enabled at creation only — cannot turn it on for an existing unencrypted file system (recreate and migrate).

## Performance modes (set at creation, cannot change)

| Mode | Use case |
|---|---|
| **General Purpose** | Default. Lowest per-operation latency. Web serving, CMS, home dirs. **Recommended for all workloads.** |
| **Max I/O** | Previous-gen, for highly parallelized big-data workloads. Higher throughput/IOPS but higher latency. Not supported on One Zone or with Elastic throughput. |

## Throughput modes (can be changed later)

| Mode | Use case |
|---|---|
| **Elastic** | Default & recommended. Auto-scales throughput up/down; pay per GiB transferred. Best for spiky/unpredictable workloads. |
| **Provisioned** | Set a fixed throughput independent of size. For steady, predictable workloads ≥5% average-to-peak usage. 1 MiBps increments. |
| **Bursting** | Throughput scales with data stored: baseline 50 MiBps per TiB (Standard), burst up to 100 MiBps/TiB using burst credits. Watch `BurstCreditBalance` — when credits hit zero, you throttle to baseline. |

## Storage classes

| Class | Use case | Notes |
|---|---|---|
| **Standard** | Active data, sub-ms latency | SSD, first write destination |
| **Infrequent Access (IA)** | Accessed a few times per quarter | Up to 92% cheaper; min 128 KiB billing per file |
| **Archive** | Accessed a few times per year | Up to 50% cheaper than IA; Elastic throughput only |

- **Lifecycle management**: automatically moves files to IA/Archive after N days (e.g. 14/30/60/90). Reading IA/Archive data incurs access charges.

## EFS ↔ compute

- Works with **EC2, ECS, EKS, Lambda, Fargate** — multiple types of compute can share one file system.
- **AWS Backup** integration for automated backups and versioning.
- **Replication**: cross-Region replication possible for DR/compliance.

## Exam domains

- [x] **Secure (30%)** — encryption at rest, Security Groups (TCP 2049), IAM permissions, mount target placement
- [x] **Resilient (26%)** — Regional (Multi-AZ) vs One Zone, AWS Backup, cross-Region replication, lifecycle tiering
- [x] **High-Performing (24%)** — General Purpose vs Max I/O, Elastic/Provisioned/Bursting throughput, burst credits
- [x] **Cost-Optimized (20%)** — IA & Archive classes, lifecycle policies, One Zone, Provisioned sizing

## Key gotchas

1. **Linux-only** — NFS + POSIX. Windows shared drive → **FSx for Windows**, not EFS
2. **Encryption at rest can't be enabled later** — must recreate and migrate
3. **Performance mode fixed at creation** — cannot switch General Purpose ↔ Max I/O
4. **Port 2049** — EFS Security Group must allow NFS inbound from compute
5. **Burst credits exhaust** — sustained high throughput throttles to baseline; move to Elastic/Provisioned
6. **IA/Archive access fees** — cheap storage, but every read costs
7. **One Zone not 11-nines** — data can be lost if the AZ fails
8. **Elastic throughput min-metering** — each I/O metered at 32 KiB minimum; many tiny ops cost more than expected

## Related services

- **EC2/ECS/EKS/Lambda** — compute clients mounting the same file system
- **AWS Backup** — automated backup & restore
- **FSx** — use when you need Windows SMB, HPC Lustre, ONTAP, or ZFS instead of generic Linux NFS
- **DataSync** — migrate on-prem NFS data into EFS