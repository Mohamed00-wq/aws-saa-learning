# Storage Gateway Course  Hybrid Cloud Storage

## 1. Purpose

AWS Storage Gateway is a **hybrid cloud storage service** that bridges on-premises applications to AWS storage. A software appliance (VM) runs in your data center (or on EC2) and presents **standard protocols** your apps already use  NFS/SMB, iSCSI, or a virtual tape library  while seamlessly storing data in AWS (S3, EBS snapshots, Glacier). It gives **local low-latency access + cloud durability/cost** without rewriting applications for the cloud.

Internet Gateway
→ VPC ↔ Internet

Storage Gateway
→ On-Premises ↔ AWS Storage

## 2. How it works

- Deploy the gateway as a **VM appliance** (VMware/Hyper-V/KVM) on-prem, or a **hardware appliance**, or **on EC2**
- Attach **local disks** to the VM: **cache** (recently accessed / hot data) + **upload buffer** (staging for uploads)
- The gateway presents a familiar protocol to apps and translates it to AWS storage behind the scenes
- Data is encrypted in transit (SSL) and at rest (SSE-S3 by default optional KMS for volume/tape)
- Four gateway types map to the four on-prem storage interfaces:
  - **Amazon S3 File Gateway**  NFS/SMB files → **S3 objects** (cached locally)
  - **Amazon FSx File Gateway**  SMB files → **FSx for Windows File Server** shares (Windows/AD)
  - **Volume Gateway**  iSCSI **block volumes** backed by S3 + **EBS snapshots** (cached or stored mode)
  - **Tape Gateway**  iSCSI **virtual tape library (VTL)** → S3/Glacier/Deep Archive

```
On-prem app ── NFS/SMB / iSCSI ── Storage Gateway VM (cache + upload buffer)
                                        │ SSL
                                        ▼
                  S3 objects  /  FSx shares  /  EBS snapshots  /  Glacier tapes
```

## 3. When to use

- **Hybrid / gradual cloud migration**  keep apps on-prem but store data in AWS, no re-architecture
- **On-prem file shares backed by S3** (data lakes, backups, ML datasets)  S3 File Gateway
- **Windows team shares / AD-integrated files**  FSx File Gateway
- **On-prem block storage with cloud backup/DR**  Volume Gateway snapshots restore to AWS or on-prem
- **Replace physical tape libraries** for backup/archive  Tape Gateway with existing backup software
- **Free up local capacity**  cache mode keeps only hot data local, primary data in S3
- **Low-latency local access + cloud durability** for branch offices / data centers

## 4. When NOT to use

- **You're already all-in on AWS**  use S3/EFS/FSx/EBS directly no gateway needed
- **You need active-active multi-site writes**  Storage Gateway is largely single-writer/per-gateway
- **Cloud-native apps that can call S3 APIs directly**  skip the appliance
- **HPC / very high-IOPS block workloads**  local SAN/DAS or FSx for Lustre, not gateway caching
- **Large one-time data migration**  **AWS DataSync** / **Snow Family** move data faster and cheaper Storage Gateway is for ongoing hybrid access
- **Online primary block storage with strict latency/HA**  EBS or on-prem SAN is a better fit

## 5. Important features

- **File Gateway (S3)**  NFS/SMB → S3 objects local cache (150 GiB–64 TiB) files become S3 objects with object-level lifecycle/tiering supports S3 Object Lock/WORM
- **FSx File Gateway**  SMB access to FSx for Windows shares AD integration, Windows ACLs, user home dirs up to 50 shares / 500 sessions per single-instance gateway (verify current support status)
- **Volume Gateway**  iSCSI block volumes:
  - **Cached mode**  primary data in S3, hot subset cached locally volume up to **32 TiB**, **32 volumes/gateway → 1 PiB** **point-in-time EBS snapshots**
  - **Stored mode**  full dataset stored locally with low latency, **async snapshots to S3** volume up to **16 TiB**, **512 TiB/gateway**
  - **AWS Backup integration** for cached and stored volumes
- **Tape Gateway**  virtual media changer + tape drives virtual tapes in service-managed S3, archive to **S3 Glacier Flexible Retrieval / Deep Archive** use existing backup software 99.999999999% durability vs physical tape no tape handling
- **Local cache & upload buffer**  tunable disk allocation cache ~20%+ of working set for performance
- **Encryption**  SSL in transit SSE-S3 at rest by default KMS-optional for volume/tape
- **High availability**  gateway can run as VM (with hypervisor HA), on EC2 EBS snapshots for Volume Gateway DR

## 6. Limitations

- **Not for migrations**  it's ongoing hybrid access, not a bulk transfer tool (DataSync/Snow move data)
- **Cached-mode primary data** is only accessible through the gateway (not via S3 API/console directly)  no S3-side reads
- **Single gateway = potential SPOF**  needs hypervisor HA / redundant deployment for production
- **Volume size/gateway caps**  cached 32 TiB/volume, stored 16 TiB/volume overall per-gateway limits apply
- **More moving parts**  appliance patching, local disk sizing (cache + upload buffer), network bandwidth to AWS
- **Latency to AWS data**  cache misses / writes not yet uploaded hit cloud latency
- **Gateway-specific protocols**  iSCSI for Volume/Tape no arbitrary block or object protocol
- **FSx File Gateway support status**  confirm current AWS guidance, as AWS has been steering Windows file shares toward other patterns
- **Bandwidth bound**  upload throughput depends on your link to AWS

## 7. Trade-offs

- **Storage Gateway vs DataSync vs Snow**  ongoing hybrid access vs periodic/one-time data movement DataSync for scheduled bulk transfer, Snow for offline/very large
- **Cached vs Stored Volume Gateway**  primary data in cloud + local hot cache (scales to 1 PiB, lower local cost) vs full local dataset with cloud backup (fastest local, 512 TiB cap)
- **S3 File Gateway vs FSx File Gateway**  generic NFS/SMB object access vs Windows-native SMB + AD/ACLs
- **Tape Gateway vs real tape**  cloud durability + no media handling + pay-as-you-go vs physical tapes (offline, cheaper at massive scale but operational burden)
- **Gateway VM on-prem vs on EC2**  local app latency vs cloud-resident applications accessing cloud-local storage
- **Bigger local cache vs smaller**  performance for hot data vs local disk cost
- **Storage Gateway vs direct S3/EFS**  protocol continuity/no app rewrite vs simplest cloud-native path

## 8. Architecture

Reference hybrid patterns:

```
On-prem file shares → S3 File Gateway (NFS/SMB) → S3 bucket → lifecycle to IA/Glacier
                                                   → analytics (Athena/Glue) on the S3 objects

Windows shares → FSx File Gateway → FSx for Windows (AD, ACLs) → AWS Backup

On-prem DB/app → iSCSI → Volume Gateway (cached) → S3 primary + EBS snapshots
                                                      → restore snapshot to EC2 (DR) or on-prem

Backup server → iSCSI VTL → Tape Gateway → virtual tapes in S3 → Glacier/Deep Archive
```

## 9. SAA-C03 Perspective

Storage Gateway is a **hybrid storage / migration** exam topic (Domains 1, 3, 4):

- **Scenario→Storage Gateway**: "keep on-prem applications, store data in AWS", "low-latency local access + cloud durability", "extend on-prem file/block/tape to AWS without app changes"
- **Gateway type matching is the key skill**:
  - Files → S3 objects → **S3 File Gateway**
  - Windows/SMB shares → **FSx File Gateway**
  - Block iSCSI volumes + EBS snapshots → **Volume Gateway** (cached vs stored)
  - Replace tape backup → **Tape Gateway**
- **Cached vs Stored volume**: cached = primary in S3 (scale, cost), stored = full local + async backup
- **Volume caps**: cached 32 TiB/volume → 1 PiB/gateway stored 16 TiB/volume → 512 TiB
- **Tape Gateway** archives to **Glacier/Deep Archive** uses existing backup software
- **Encryption**: SSL in transit + SSE-S3 at rest (KMS optional)
- **vs DataSync** (scheduled bulk transfer) and **Snow Family** (offline petabyte migration)  a classic three-way choice
- **vs direct S3/EFS/EBS** when apps are cloud-native

Exam trap: "on-prem app must keep writing files at low latency while data also lands in S3 for analytics" → **S3 File Gateway**. "Replace physical tape backups, archive to lowest-cost storage" → **Tape Gateway → Glacier Deep Archive**. "Migrate 500 TB to AWS once as fast as possible" → **Snow Family or DataSync**, not Storage Gateway.