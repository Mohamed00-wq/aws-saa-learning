# FSx  Amazon FSx File Systems

## What it is

Amazon FSx is a family of **four fully managed, feature-rich file systems**, each built on a well-known commercial or open-source file system. Use it when you need a specific capability EFS doesn't offer: Windows SMB, HPC parallel I/O, NetApp ONTAP, or ZFS. AWS handles hardware, patching, backups, and replication.

## The four flavors  pick by what the scenario names

| Flavor | Protocol(s) | Best for | Exam trigger keywords |
|---|---|---|---|
| **FSx for Windows File Server** | **SMB** | Windows shares, home dirs, .NET/SQL Server, CMS | "Windows", "Active Directory", "SMB", "NTFS", "DFS" |
| **FSx for Lustre** | Lustre (parallel, POSIX) | HPC, **ML training**, video rendering, financial modeling | "HPC", "machine learning", "high throughput", "S3", parallelism |
| **FSx for NetApp ONTAP** | **NFS + SMB + iSCSI** | Migrating on-prem NetApp multi-protocol NAS dev/test clones | "NetApp", "SnapMirror", "FlexClone", "multi-protocol" |
| **FSx for OpenZFS** | NFS (v3/v4.x) | Linux/ZFS migrations, high IOPS, low latency, instant clones | "ZFS", "snapshots", "compression", "Linux NFS" |

## FSx for Windows File Server

- Built on **Windows Server** SMB 2.0–3.1.1, NTFS, Active Directory integration, DFS Namespaces + Replication, shadow copies, quotas, dedup.
- **SSD** (low latency, IOPS) or **HDD** (cost-effective throughput) storage.
- **Single-AZ** (cheaper) or **Multi-AZ** (synchronous replication + automatic failover in ~60s).

## FSx for Lustre

- High-performance **parallel file system**: hundreds of GB/s, millions of IOPS, sub-ms latency. **Linux only**.
- **Native S3 integration**: lazily hydrates objects from S3 as files and writes results back  S3 becomes the durable backing store, Lustre the high-speed compute cache.
- **Deployment types**:
  - **Scratch**  temporary, no replication, highest performance, cheapest. For ephemeral data that can be regenerated.
  - **Persistent**  durable data, replicated, self-healing, for long-term compute data.
- SSD or HDD storage.

## FSx for NetApp ONTAP

- Runs NetApp's ONTAP OS. **Only flavor that's simultaneously NAS and SAN** (NFS, SMB, iSCSI).
- Features: **SnapMirror** replication (async, RPO ~5 min), **FlexClone** instant space-efficient clones, dedup, compression, thin provisioning, automatic tiering of cold data to a cheap capacity pool.
- Single-AZ or Multi-AZ.

## FSx for OpenZFS

- Managed OpenZFS over NFS. **ZFS-native**: near-instant snapshots, instant clones, copy-on-write, compression (Zstandard up to ~75% savings).
- Very low latency (<0.5 ms cached), up to ~1 million IOPS.
- Good for migrating on-prem ZFS, dev/test environments needing fast clones, databases needing consistent snapshots.

## Deployment & protection (shared across flavors)

- **Multi-AZ** (Windows, ONTAP, OpenZFS): synchronous replication across AZs with automatic failover. Lustre is single-AZ.
- **Backups**: automatic daily backups + snapshots, stored in S3, managed via **AWS Backup** cross-Region / cross-account copy for DR.
- **Encryption**: at rest via KMS on all flavors in transit (SMB encryption / Kerberos, NFS over TLS, Lustre internal channels).
- **One-way doors**: deployment type and storage class are fixed at file system creation  changing either requires a new file system.

## Choosing: EFS vs FSx

- Vanilla **Linux NFS shared storage** → **EFS** (simpler, serverless, elastic).
- FSx only when the scenario names a capability EFS lacks: **SMB/AD** → Windows **HPC/ML/parallel + S3 integration** → Lustre **NetApp/SnapMirror/multi-protocol** → ONTAP **ZFS features** → OpenZFS.

## Exam domains

- [x] **Secure (30%)**  KMS encryption, Active Directory/Kerberos, security groups (SMB port 445), IAM
- [x] **Resilient (26%)**  Multi-AZ failover, AWS Backup, cross-Region copy, Scratch vs Persistent
- [x] **High-Performing (24%)**  Lustre throughput, SSD vs HDD, provisioned throughput/IOPS tiers
- [x] **Cost-Optimized (20%)**  Scratch for ephemeral, HDD tiers, ONTAP tiering, OpenZFS compression, dedup

## Key gotchas

1. **EFS is the default for Linux NFS**  only pick FSx when a specific feature (Windows, Lustre, ONTAP, ZFS) is named
2. **"Shared Windows storage" + "SMB" → FSx for Windows**, never EFS
3. **Lustre only native S3 integration**  hydrates/export-pushes to S3
4. **Lustre Scratch is not durable**  rehydrate from S3 use Persistent when data matters
5. **Deployment type & storage class fixed at creation**  can't change later
6. **ONTAP is the only multi-protocol (NFS+SMB+iSCSI) flavor**  answer for mixed Linux/Windows estates
7. **Lustre is Linux clients only**  Windows clients can't mount it
8. **FlexClone/SnapMirror keywords → ONTAP** **Zstd compression / instant snapshots → OpenZFS**

## Related services

- **EFS**  general-purpose Linux NFS alternative
- **S3**  backing store for Lustre backup storage for snapshots
- **AWS Backup**  automated backup and cross-Region copy for all flavors
- **Directory Service / on-prem AD**  authentication for FSx for Windows
- **DataSync**  migrating file data into FSx