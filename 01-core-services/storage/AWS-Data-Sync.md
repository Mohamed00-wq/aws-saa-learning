# AWS DataSync Course  Fast Online Data Migration & Replication Service

## 1. Purpose

AWS DataSync is a **high-speed, online (network) data movement service** that migrates and replicates **file & object data between on-premises storage, other clouds, edge, and AWS storage (S3, EFS, FSx)**. It's agent-based, **up to 10 Gbps per agent**, and preserves metadata (POSIX/Windows) with built-in scheduling, incremental sync, and integrity verification  replacing scripts/rsync-style DIY pipelines. For SAA it's the answer to **"migrate on-prem files/objects to AWS online / replicate datasets to S3-EFS-FSx / frequent incremental sync"**.

## 2. How it works

- **DataSync agent**  a small VM deployed on-prem/edge (or in other clouds/VPC) that connects via TLS to the DataSync service and streams data at up to **10 Gbps per agent**
- **Locations**  source & destination: **NFS, SMB, HDFS, self-managed object storage, Azure/GCP storage** ⇄ **Amazon S3 (any storage class incl. Glacier), EFS, FSx (Windows, Lustre, OpenZFS, ONTAP)**
- **Tasks → task executions**  locations + options: **schedule** (hourly/daily/weekly), **incremental transfers** (only changed), **filters/includes**, **bandwidth throttling** (to avoid choking production), **verify data integrity** (checksums after transfer), overwrite/delete semantics, task reports
- **Enhanced mode** (parallel listing/prepare/transfer/verify  no file-count limits, detailed metrics) for S3↔S3 and NFS/SMB→S3, and now **EFS & FSx for Lustre** **Basic mode** covers the full matrix (incl. EFS/FSx, HDFS, object storage)
- **Metadata preserved**  POSIX (NFS/HDFS) mapped to S3 object metadata and restored when copied back Windows ACLs for SMB/FSx Windows
- **Cloud-native transfers**  S3↔S3/EFS/FSx, across accounts & Regions **VPC endpoints (PrivateLink)** CloudWatch metrics, logging, EventBridge events, IAM roles

```
On-prem NFS/SMB/HDFS/appliance ──DataSync agent (10 Gbps, TLS)──► DataSync service
  → S3 (any class, incl. Glacier) | EFS | FSx | other clouds
Task options: schedule, incremental, bandwidth limit, verify, filters, enhanced/basic mode
Cloud-only: S3 ⇄ EFS/FSx (no agent needed) with PrivateLink endpoints
```

## 3. When to use

- **Migrate on-premises / edge / on-MapDatasets folders to AWS** (S3/EFS/FSx) with minimal downtime: initial full copy + **scheduled incrementals** until cutover
- **Archive / free up on-prem capacity**  copy to S3 infrequent/Glacier classes (incl. **Glacier Deep Archive**), then decommission old systems
- **Periodic replication & DR copies** between AWS storage (S3⇄EFS/FSx, cross-region/account)  replicate changed files on schedule
- **Data movement for processing**  on-prem datasets into the cloud for analytics/ML/media/HPC (genomics, rendering)
- **Copy between other clouds / self-managed object stores and AWS** on a schedule
- **File servers with mixed protocols** (NFS/SMB/HDFS) need copying with metadata/timestamps preserved

## 4. When NOT to use

- **No/low bandwidth or physically hard-to-reach sites** → **Snowball / Snow Family** (offline device)  DataSync is network-based
- **Users/partners upload/download files over FTP/SFTP/AS2/HTTP** → **AWS Transfer Family** (DataSync moves data between storage, it's not an end-user file server)
- **Ongoing protocol access for on-prem apps** (mount an S3-backed share, NFS/SMB gateway) → **Storage Gateway (File Gateway)**
- **Real-time event streaming** (Kafka/event ingestion) → Kinesis/MSK/AppSync
- **Database schema/table migration (incl. CDC across DB engines)** → **AWS DMS**
- **S3-native object replication only within S3** → **S3 Replication** (simpler, serverless, no DataSync cost)

## 5. Important features

- **Online, agent-based high-speed transfer**  up to **10 Gbps per agent** scale by adding agents and splitting workloads
- **Scheduled + incremental** sync (hourly/daily/weekly, run-at)  detect & copy only changes
- **Metadata fidelity**  POSIX/Windows ACLs, timestamps, ownership preserved across NFS/SMB/HDFS ↔ S3/EFS/FSx
- **Bandwidth throttling** & traffic shaping  protect production bandwidth during off-hours
- **Data integrity verification**  end-to-end (checksums) on every transfer task reports
- **Enhanced mode**  parallelized operations, no file-count limits, richer metrics (S3⇄S3, NFS/SMB→S3, EFS/FSx-Lustre)
- **Broad matrix**  NFS, SMB, HDFS, object storage, Azure/GCP any S3 class incl. Glacier EFS, FSx (Win/Lustre/OpenZFS/ONTAP) cross-account & cross-Region
- **CloudWatch metrics & logs, EventBridge events, PrivateLink endpoints**, IAM pay-per-GB (no upfront)

## 6. Limitations

- **Network-transfer model**  depends on available bandwidth/connectivity (very large/unreliable links → Snowball)
- **On-prem/other-cloud sources require an agent VM** (deploy & manage includes activation, network route)
- **Enhanced mode supports a subset of locations** (Basic mode covers the rest with older limits)
- **Not a file server**  no SFTP/FTP end-user endpoints, no gateway caching semantics
- Per-GB pricing + data-transfer costs cross-Region transfers incur DT charges
- Not for real-time streaming or DB replication (DMS for databases)

## 7. Trade-offs

- **DataSync vs Snow Family**  online high-speed (any bandwidth, incremental-only/cutover) vs **offline physical devices** when bandwidth/time is impractical (bandwidth ≤~ hundreds Mbps Snowcone/Snow/Snowmobile)
- **DataSync vs Storage Gateway (File Gateway)**  one-way migration/replication between locations (DataSync) vs **ongoing protocol-based access** + local cache for on-prem apps (Gateway)
- **DataSync vs S3 Replication**  cross-storage-file/agent-based (DataSync) vs S3-native object-only replication (Replication)
- **vs Transfer Family**  store-to-store data movement (DataSync) vs managed FTP/SFTP server endpoints for people/partners (Transfer Family)

## 8. Architecture

```
Migration of a file farm:
  Datacenter NFS/SMB share → DataSync agent (each 10 Gbps, TLS, throttled off-peak)
    → initial FULL copy to S3 (Intelligent-Tiering/Glacier for cold + EFS for active apps)
    → scheduled INCREMENTAL syncs during migration window
  Verify (checksums, task reports) → cutover → decommission old storage
Cloud DR: replicate EFS/FSx → standby EFS in DR Region via DataSync (schedule)
Monitor CloudWatch metrics/events PrivateLink where required
```

## 9. SAA-C03 Perspective

- **"Migrate on-prem file/object data to S3/EFS/FSx ONLINE (network transfer)"** → **DataSync**
- **"Move NFS/SMB/HDFS metadata + incremental scheduled replication to AWS"** → **DataSync**
- **"Copy files to Glacier/long-term storage, free on-prem capacity"** → DataSync (any S3 class)
- **"Physical or slow-link area, 'ship us a device'"** → **Snowball / Snow Family**
- **"FTP/SFTP endpoints for partners"** → **Transfer Family** "S3 object-only replication" → **S3 Replication** "database migration" → **DMS**

Exam traps: Distinguish the four movers  **DataSync** (online store-to-store, agent, schedule, metadata) vs **Snowball** (offline physical) vs **Storage Gateway** (ongoing protocol access/caching) vs **Transfer Family** (SFTP/FTP/AS2 server endpoints). Not for FTP users not for DB schema not real-time streaming.