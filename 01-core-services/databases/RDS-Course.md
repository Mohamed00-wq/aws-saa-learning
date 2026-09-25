# RDS Course  Relational Database Service

## 1. Purpose

RDS is AWS's **managed relational database service** for the classic SQL engines: **MySQL, MariaDB, PostgreSQL, Oracle, SQL Server, and Db2**. You get a version, size, and Multi-AZ/read-replica setup  AWS handles **patching, backups, snapshots, failover, encryption, and storage** so you can run production SQL without a DBA for the plumbing. Pick RDS for familiar, standards-based relational workloads pick **Aurora** when you want the higher-performance AWS-native engine.

## 2. How it works

- **DB instance**  an EC2-hosted, AWS-managed DB you pick **engine, version, instance class, storage (gp2/gp3/io1/io2), and VPC/subnet placement**
- **Multi-AZ** for HA:
  - **Multi-AZ (1 standby)**  **synchronously replicated** standby in another AZ **automatic failover** (typically < 35 s) standby is *not* readable
  - **Multi-AZ (2 readable standbys)**  MySQL/PostgreSQL: TWO readable standbys across 3 AZs, **failover < 35 s**, ~**2x faster commit latency**, serve reads
  - **Read Replicas**  separate instances for **read scaling** (async replication), promotable to standalone cross-Region possible up to 5 (Oracle/SQL Server) or 15 (MySQL/MariaDB/PostgreSQL)
- **Backups**  **automated**: daily snapshot + **transaction logs every 5 min** → **PITR to any second within retention (max 35 days)** manual **snapshots** persist until deleted stored in S3
- **Encryption**  **at rest** (EBS + snapshots + replicas) via **KMS** **in transit** via SSL/TLS
- **Storage**  gp2/gp3 (general), io1/io2 (high IOPS), magnetic (legacy) **storage autoscaling**
- **RDS Proxy**  connection pooling for Lambda/serverless (avoids connection exhaustion)

```
App → [RDS Proxy optional] → primary (writer endpoint)
   ├─ Multi-AZ standby (synchronous, invisible)
   └─ Read Replicas (async) → reader endpoint for read-heavy traffic
   Backups: daily snapshot + 5-min txn log → PITR ≤35 days snapshots → S3
```

## 3. When to use

- **Familiar SQL engines** (Oracle/MySQL/PostgreSQL/SQL Server/MariaDB) with **existing skills/tooling**
- **OLTP** transactional workloads with moderate scale
- **Managed operations**  you don't want to patch/backup/tune a self-managed DB
- **Read-heavy apps**  scale reads with **Read Replicas**
- **High availability required**  **Multi-AZ** (AZ failure → automatic failover) + **Cross-Region read replica** for DR
- **Point-in-time recovery**  restore to any second (retention ≤ 35 days)
- **Lift-and-shift** existing databases integrations like RDS Proxy, Performance Insights, CloudWatch

## 4. When NOT to use

- **Single-digit-ms / extreme scale / multi-AZ high-IOPs heavy-duty** → **Aurora** (better storage architecture, 5x throughput, auto-scaling replicas)
- **Highly variable/unpredictable DB demand** → **Aurora Serverless / RDS on Aurora**
- **Analytics/columnar-optimized large data warehousing** → **Redshift**
- **NoSQL / key-value / flexible schema at scale** → **DynamoDB** **document** → DocumentDB
- **In-memory caching** → **ElastiCache**
- **Need full DB engine control / custom OS** → EC2-self-hosted (RDS restricts some admin)
- **Graph, time-series, ledger** → Neptune, Timestream, QLDB
- **Serverless first, low/no use case PITA** → Aurora Serverless / RDS Serverless-GP? (Aurora is common answer)
- **Strict version/patch control or weird plugins** → self-managed DB

## 5. Important features

- **High availability**  Multi-AZ sync failover (single standby classic **two readable standbys** on MySQL/Postgres for 2x commit + reads) **cross-Region read replica / automated backups** for DR
- **Read Replicas**  MySQL/MariaDB/PostgreSQL: **up to 15** Oracle/SQL Server: **up to 5** async replication promote to read-write optional own Multi-AZ
- **PITR & snapshots**  daily snapshot + 5-min txn log backups retention **max 35 days** restore to new instance latest restorable ≈ within **5 minutes** snapshot restore typically *not* instantaneous but fast
- **Storage**  gp3 default, io1/io2 for high IOPS (provisioned), **storage autoscaling** (helps grow without restarts)
- **Encryption**  KMS at rest + SSL/TLS in transit snapshots/read replicas inherit cross-Region snapshots must be KMS-key-compatible
- **Dedicated / managed services**  **RDS Proxy** (pool & reuse connections, esp. Lambda), **Performance Insights** (query tuning), **Enhanced Monitoring** (OS metrics), CloudWatch alarms
- **Maintenance windows** + **blue/green deployments** (safe major upgrades)
- **Amazon RDS on Outposts / External**  run RDS on-prem places
- **AWS Backup** integration **Zero-ETL** with Aurora/Redshift for analytics
- **Database activity streams**  ingest activity into Kinesis for external DB audit (security/compliance)
- **Multi-AZ failover detection**  responds automatically app reconnects to same endpoint after DNS flip

## 6. Limitations

- **Backup retention max 35 days**  need longer? manual snapshots / export to S3 / AWS Backup
- **Fam sex**: standby in 1-standby Multi-AZ **can't serve reads** (CPU idle) make a readable replica instead if you want read scaling
- **Read Replica replication lag**  async not for write-critical consumers
- **Single instance max size / IOPS caps**  very high-throughput OLTP exceeds RDS storage/storage limits → **Aurora**
- **Engine restrictions**  some admin tasks, plugin load, superuser needs are blocked compared to self-managed
- **Failover is not instant**  app must handle brief connection outage DNS propagation a few sec
- **Version upgrades / patches**  AWS-controlled maintenance windows (can schedule)
- **Cross-Region read replica promote** creates a new standalone instance (not automatic global failover)
- **Cost**  RDS bills instance + storage + I/O + backup storage expensive if keep idle Multi-AZ/HA everywhere
- **SQL Server/Oracle licensing pricing**  different per engine/sizing beware per-core charges
- **Not unlimited connections**  max connections (based on instance memory) use **RDS Proxy** for Lambda/serverless

## 7. Trade-offs

- **RDS vs Aurora**  RDS = classic engines + managed template Aurora = higher performance **MySQL/PostgreSQL-compatible** engine with **6-way replicated storage**, auto-scaling readers, **Serverless**, Global Database, ~lower cost-per-performance at scale
- **Multi-AZ vs Read Replicas**  synchronous HA failover vs async read scaling a **read replica can also serve as DR target/promote**
- **Single-standby vs two-readable-standbys**  simpler classic vs 3-AZ 2x commit + read capacity (MySQL/Postgres only)
- **PITR (daily + logs) vs manual snapshots**  last-5-min restore vs point-in-time-aggregate long-term retention
- **gp3 vs gp2/io2**  cost-efficiency at baseline IOPS vs high provisioned IOPS for heavy OLTP
- **RDS vs self-managed EC2 DB**  ops automation + HA out-of-box vs full control/tuning/licensing approach
- **RDS Proxy vs app pooling**  quick win for Lambda/serverless (pre-warmed connections) vs added hop/latency/vendor dependency
- **Cross-Region replicas vs restore-snapshot**  continuous async replica (RPO low) vs periodic snapshot restore (RPO days)
- **RDS vs ElastiCache behind it**  DB of record + caching tier vs pure cache semantics

## 8. Architecture

Reference patterns:

```
HA Web + DB:
  ALB → EC2/ECS app → RDS Proxy → RDS MySQL (Multi-AZ) primary + read replica for reporting dashboard
  → PITR 7 days + daily snapshots → cross-Region read replica for DR (promote on disaster)

Serverless + relational:
  API Gateway → Lambda → RDS Proxy (pool) → RDS PostgreSQL (gp3, storage autoscaling)

Careful low-latency OLTP:
  RDS Oracle io2 (high IOPS) Multi-AZ + read replica for heavy analytic queries
```

## 9. SAA-C03 Perspective

RDS covers **HA, scalability, security, and cost** (Domains 1, 2, 4):

- **"Managed relational DB with MySQL/Postgres/Oracle/SQL Server"** → **RDS** (Aurora if it's "higher-throughput MySQL/PostgreSQL-compatible")
- **"Automatic failover to second AZ"** → **Multi-AZ** (synchronous <35 s failover)
- **"Scale read traffic"** → **read replicas** (async cross-Region for DR)
- **"Restore to any second"** → **PITR** (need automated backups, retention ≤ 35 days) older than retention → manual snapshots/S3
- **"Encrypt at rest"** → **KMS** "pool DB connections for Lambda" → **RDS Proxy**
- **Storage autoscaling** to avoid full storage gp3 cost-efficient baseline
- **vs Aurora**: scenario mentions **MySQL/PostgreSQL compatibility + higher performance/scale/serverless/global** → **Aurora** purple-flag scenario mentioning any SQL engine that needs managed → **RDS**
- **Backups are stored in S3** snapshot restore = new instance failover flips DNS
- **Security**  SG/VPC, IAM auth (RDS supports IAM for MySQL/Postgres), encryption, activity streams, CloudTrail

Exam traps: "Multi-AZ standby serves reads" → **false in 1-standby (passive)** "PITR beyond 35 days" → **not from automated backups, need snapshots** "failover is instant/zero-impact" → **brief reconnection** "read replicas are synchronous" → **async (lag)** "3rd AZ standby = Multi-AZ" → Multi-AZ spans exactly **2 AZs** (3 with two-readable-standbys) ✅ "RDS is fully managed, no OS admin" → correct, that's the point.