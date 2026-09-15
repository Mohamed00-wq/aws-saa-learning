# Storage Gateway — AWS Storage Gateway

## What it is

A **hybrid cloud storage service**: it gives on-premises applications access to virtually unlimited AWS cloud storage using standard protocols (NFS, SMB, iSCSI) — no application changes needed. Deployed as a **VM** (VMware ESXi, Hyper-V, KVM) or hardware appliance in your data center, connecting over the internet or **Direct Connect**. It caches frequently accessed data locally for low latency and asynchronously uploads the rest to AWS.

## The gateway types

| Gateway | Protocol | Backend | Use case |
|---|---|---|---|
| **S3 File Gateway** | NFS v3/v4.1, SMB v2/v3 | **S3** | File shares backed by S3; store files as native S3 objects |
| **FSx File Gateway** | SMB | **FSx for Windows** | Low-latency on-prem access to Windows file shares + AD |
| **Volume Gateway** | iSCSI (block) | **S3 + EBS snapshots** | Block volumes for databases, backups, DR |
| **Tape Gateway** | iSCSI (VTL) | **S3, then Glacier/Deep Archive** | Replace physical tape backup infrastructure |

## S3 File Gateway

- Presents an S3 bucket as an **NFS or SMB file share**. Files become **native S3 objects** (path = object key, POSIX metadata = object metadata).
- Objects then benefit from S3 features: lifecycle policies, cross-Region replication, Object Lock, analytics, Athena/EMR.
- Local cache keeps hot files fast; cache up to **10 TiB** per gateway.
- Use cases: migrate on-prem files to S3, back up file data, hybrid ML/big-data pipelines.

## Volume Gateway

- Presents **iSCSI block volumes** (mounted as disk devices) backed by S3. Backups become **EBS snapshots**, restorable as EBS volumes or gateway volumes.

| Mode | Primary data lives | Volumes | Cost/latency profile |
|---|---|---|---|
| **Cached volumes** | S3 (local cache of hot data) | up to **32 TiB** each, 1 PB per gateway | Minimal on-prem storage; low local cost |
| **Stored volumes** | On-premises (async backup to S3 as EBS snapshots) | up to **16 TiB** each, 512 TB per gateway | Max local performance; expensive footprint |

- Best for database backups and disaster recovery (snapshots → EBS volumes in AWS).

## Tape Gateway

- Emulates a **virtual tape library (VTL)**: virtual media changer + tape drives, presented to your existing backup software (Veeam, NetBackup, Commvault, etc.) over iSCSI — keeps the tape workflow, drops the hardware.
- Active tapes stored in **S3** (immediate access); archived tapes move to **S3 Glacier / Glacier Deep Archive** for long-term retention.
- Eliminates physical tape, media management, and off-site shipping costs.

## How it works

- Download/install the VM, activate it, then create shares/volumes/tapes on the AWS side.
- Data written locally → upload buffer → **encrypted transfer (TLS)** → stored in AWS, encrypted at rest (SSE-S3 or SSE-KMS).
- Only changed data is transferred (block-level / async), keeping bandwidth use low.

## Exam domains

- [x] **Secure (30%)** — TLS in transit, SSE-S3/SSE-KMS at rest, IAM + vault access policies
- [x] **Resilient (26%)** — EBS snapshot DR, stored volumes for local durability, AWS Backup integration
- [x] **High-Performing (24%)** — local caching for low latency, cached vs stored volume choice, Direct Connect
- [x] **Cost-Optimized (20%)** — cached volumes minimize on-prem storage, tape tiering to Glacier/Deep Archive

## Key gotchas

1. **"NFS/SMB file share → S3" ⇒ S3 File Gateway**; **"iSCSI block volume" ⇒ Volume Gateway**; **"tape"/backup app ⇒ Tape Gateway**
2. **File Gateway is file protocol only** — no block/iSCSI. **Volume Gateway is block only** — no file protocol.
3. **Cached = data in cloud, minimal on-prem storage. Stored = data on-prem, lower latency for the whole dataset.**
4. **Volume Gateway volumes are not directly S3-accessible** — only via iSCSI; snapshots become EBS snapshots
5. **Cannot be a boot volume for EC2** — data volumes only
6. **Tape Gateway keeps your backup software unchanged** — the point is VTL emulation
7. **Local cache (≥150 GiB) is required** — a gateway without cache can't run
8. **Use DataSync when there's nothing to cache** — full one-time migration to AWS, no on-prem footprint

## Related services

- **S3 / S3 Glacier / Glacier Deep Archive** — backing storage for files and tapes
- **EBS** — snapshots created by Volume Gateway, restored as block storage in AWS
- **FSx for Windows** — backing store for FSx File Gateway
- **DataSync** — the alternative for simple one-time migrations without a persistent gateway
- **AWS Backup** — centralizes backup of Volume Gateway volumes and File Gateway data