# S3 Course — Simple Storage Service

## 1. Purpose

S3 is AWS's **object storage** service. Data lives in **buckets** as **objects** (key + value + metadata), addressed over HTTP/HTTPS — there's no filesystem hierarchy (folders are just key prefixes). Data is stored redundantly, durable to **11 nines (99.999999999%)**, and virtually infinite in scale. S3 is the backbone of backups, static assets, data lakes, media, logs, and app state in nearly every AWS architecture.

## 2. How it works

- **Bucket** = container (globally unique name), **region-scoped**; **object** = up to **5 TB**, with a key (path-like name) and optional version
- Single `PUT` handles up to **5 GB**; larger objects use **Multipart Upload** (parallel parts, resume)
- **Consistency**: strong read-after-write + strong read-after-delete/overwrite for PUTS/DELETEs (all S3 handles, including the new directory buckets / S3 Express)
- **Storage class** determines cost vs retrieval speed per object/bucket
- Access is controlled by **IAM**, **bucket policies**, **access points**, or **presigned URLs**
- AWS ingests you define: **lifecycle rules** (move+expire objects over time), **versioning** (history), **replication** (cross-region/cross-account), and **event notifications** (PUT → Lambda/SQS/SNS/EventBridge)

```
Users/EC2/Lambda/App → HTTP(S) → S3 bucket (region X, globally-unique name)
   object key, storage class, version, encryption, lifecycle policies
   → 11-nines durability, replicated across ≥3 AZs (Standard)
   → optional: event → Lambda/SQS/SNS/EventBridge; replication → other region/account
```

## 3. When to use

- **Unstructured/object data**: images, videos, PDFs, logs, backups, app artifacts
- **Static websites / frontend assets** (with CloudFront for HTTPS + speed)
- **Data lakes & analytics** — data source for Athena, Redshift Spectrum, Glue, EMR
- **Backup & DR** — EBS/EFS/RDS snapshots, database backups (S3 is the standard archive vendor)
- **Long-term / compliance archive** — Glacier tiers (cost per GB collapses)
- **Media pipelines** — upload → S3 event → Lambda transcode → back to S3 → CloudFront
- **Distributed shared "file-like" access** across many apps/regions (versioning + replication)

## 4. When NOT to use

- **Block storage for a single VM** — that's EBS (S3 has no in-place mounting/low-latency block semantics)
- **Shared filesystem across many EC2 instances** — that's EFS/FSx (S3 isn't a POSIX mount)
- **Server-generated content being edited on the fly** — S3 objects are immutable/replace-whole; dynamic state belongs in DBs/caches
- **Very hot, low-latency random I/O** — DynamoDB or in-memory (Redis/ElastiCache) instead of per-object GETs
- **Transactional/relational data** — RDS/Aurora/DynamoDB
- Filesystem/SMB/Windows or HPC needs → FSx

## 5. Important features

- **Storage classes** (the exam's bread and butter):
  | Class | Use | Retrieval |
  |---|---|---|
  | **S3 Standard** | Hot, frequent access (≥3 AZ) | Instant |
  | **Intelligent-Tiering** | Unknown/oscillating patterns — auto-tiering (+ monitoring fee) | Instant |
  | **Standard-IA** | Infrequent, fast retrieval (≥3 AZ) | Instant, per-GB fee |
  | **One Zone-IA** | Infrequent, re-creatable, single AZ (~20% cheaper) | Instant |
  | **Glacier Instant Retrieval** | Archived, rarely accessed, ms retrieval | ms |
  | **Glacier Flexible Retrieval** | Archive, backup | minutes–hours |
  | **Glacier Deep Archive** | Long-term (compliant) archive, lowest cost | 12–48 h |
  - All Standard/IA/Glacier give 11 nines durability (**except One Zone-IA**)
- **Lifecycle rules** — transition (Standard→IA→Glacier→Deep Archive) and expiration (delete old versions). Min-dwell: 30 days before IA, 90 days before Glacier/Deep Archive
- **Versioning** — keeps history; delete creates a **delete marker** (data retained); can suspend (never fully off); **MFA Delete** for permanent deletes
- **Encryption** — SSE-S3 (AES-256, zero-config), SSE-KMS (managed keys, audit + rotation), SSE-C (your keys), client-side (S3 never sees plaintext). **S3 enforces encryption via `x-amz-server-side-encryption`** header; **default bucket encryption** can force it
- **Access controls** — IAM identity policies; **bucket policies** (cross-account/public); **Block Public Access** (account+bucket safety override); **Access Points** (per-use-case policies, incl. multi-account via S3 Object Lambda); **presigned URLs** (time-limited access without changing permissions)
- **Replication** — **S3 Replication** (CRR/SRR, cross-account) for DR/compliance; **S3 Multi-Region Access Point** for global failover endpoints
- **Event notifications** → SQS/SNS/Lambda/EventBridge (OAC for CloudFront) for pipeline/processing workflows
- **Performance** — Transfer Acceleration (faster uploads via edge), Multipart Upload, **S3 Express One Zone** (sub-ms for hot access, single AZ), parallel GETs via CloudFront
- **Static website hosting** — index/error docs, HTTP-only endpoint (use CloudFront + ACM for HTTPS)

## 6. Limitations

- **Object storage semantics** — no in-place edits, no POSIX mount, no random access at block level; must GET-MODIFY-PUT
- **Glacier retrieval latency** — Deep Archive 12–48 h; not for anything time-sensitive
- **Minimum storage durations** — IA 30 days, Glacier 90 days (deleting early still bills the min)
- **Bucket names globally unique** — availability shocks; lowercase 3–63 chars (`.`, `-` allowed)
- **Single 5 GB PUT cap** — anything bigger requires Multipart (an extra step)
- **Versioning costs** — every old version bills storage; delete markers/old versions quietly accumulate
- **Block Public Access can break intended public buckets** — a trap when you actually want public static hosting
- **Min object size for KMS** (per-request charge), and KMS with EventBridge/SQS can throttle at high request rates
- **Region-bound** by default — need replication or transfer for cross-region mobility

## 7. Trade-offs

- **Standard vs Intelligent-Tiering vs Glacier** — instantly-readable & expensive vs auto-tiering (+ fee) vs cheap & slow-retrieval. Pick by access frequency + retrieval SLA
- **One Zone-IA vs Standard-IA** — ~20% cheaper but a single-AZ durability risk; choose only for re-creatable data
- **SSE-S3 vs SSE-KMS vs SSE-C** — zero-config vs audit/rotation (KMS cost/throttling) vs you hold the keys
- **Bucket policy vs IAM** — resource-based (public/cross-account, covers unknown principals) vs identity-based (least-privilege user/role control)
- **Versioning + lifecycle vs no versioning** — DR/history (storage bills) vs simplicity/cost
- **S3 vs EFS vs EBS** — object (shared, HTTP, cheap, 11-nines) vs shared NFS (POSIX, latency-sensitive) vs block (single AZ, attach-to-EC2)
- **Direct upload vs CloudFront vs Transfer Acceleration** — simplicity vs edge-speed-via-CDN (caches) vs edge-speed-for-origin-uploads
- **CRR vs SRR** — cross-region DR (redundant regions) vs same-region compliance/availability

## 8. Architecture

Reference data platform / static site patterns:

```
Static site:  Route53 → CloudFront (ACM/HTTPS, OAC) → S3 bucket (private, SSE-KMS)
               → optional WAF, geo restriction; versioned + lifecycle for rollback/cost

Analytics lake: Apps/OT → S3 raw bucket (Multipart/Acceleration, Strong consistency)
               → event → Lambda/Glue (transform) → S3 curated (Intelligent-Tiering)
               → Athena/Redshift Spectrum query
               CRR to DR region + versioning + Glacier Deep Archive via lifecycle (compliance)

Backup:  RDS/EBS snapshots → S3 via AWS Backup → lifecycle → Glacier Deep Archive (25-yr retention)
```

## 9. SAA-C03 Perspective

S3 is the **single biggest exam topic** — appears in all 4 domains. Master:

- **Storage class selection** by access pattern phrase: "frequent" → Standard; "infrequent" → IA/Glacier-Instant; "archive/compliance 12-48h OK" → Deep Archive; "unknown/fluctuating" → Intelligent-Tiering; "can be recreated" → One Zone-IA
- **Lifecycle numbers**: 30-day IA min, 90-day Glacier min; expiration for versioned buckets; Deep Archive for long retention
- **11 nines durability vs availability**; One Zone-IA is the exception
- **Encryption options + KMS** questions (at rest, in transit with ACM/TLS)
- **Access/policy traps**: bucket policies for cross-account; Block Public Access; presigned URLs; Access Points; S3 Object Lambda
- **Versioning + delete marker + MFA Delete**; **replication** (CRR/SRR) for DR/compliance
- **Events → Lambda/SQS/SNS/EventBridge** — pipeline/decoupling scenarios
- **S3 ↔ CloudFront (OAC)** for static sites with HTTPS; **Transfer Acceleration** and **Multipart** for performance
- Know **strong consistency** (no eventual-worry for S3)

Exam trap: "archive data kept 7 years, retrievable occasionally — cheapest" → **Glacier Deep Archive** (not Glacier Flexible, not IA). "Static site HTTPS + fast globally" → **CloudFront + S3 + OAC** (S3 endpoint alone is HTTP-only). "Block storage attach to one EC2" → EBS, not S3.