# Aurora Course  MySQL/PostgreSQL-Compatible High-Performance Cluster

## 1. Purpose

Aurora is AWS's **custom relational engine compatible with MySQL and PostgreSQL**  the same wire protocols, but rearchitected on a **distributed storage layer replicated 6 ways across 3 AZs**. It delivers **~5x MySQL / ~3x PostgreSQL throughput** vs RDS (per current AWS claims), **auto-healing storage**, scaled reads via serverless-style **reader instances**, and native **Serverless**, **Global Database**, and **Parallel Query**  the go-to managed relational choice on AWS when a MySQL/PostgreSQL-compatible engine needs more scale and less operational pain.

## 2. How it works

- **Cluster** with a **writer** (+ optional **readers**) storage is a **shared virtualized volume** (64 TB max) replicated **6× across 3 AZs** with automatic repair, no redo replay from the DB
- **Writer + Reader endpoints**  RW for writes, **read-only reader endpoints** (Round-robin) or per-instance endpoints for read traffic
- **Auto scaling reader pool**  adds readers on demand (fast, no replica provisioning)
- **Instant crash resilience**  storage is durable failover is quick because no log-replay on recovery
- **Serverless (formerly Serverless v2)**  scales **0.5–256 ACU** in 0.5 steps, up/down-on-demand, even **scales to zero** when idle, platform version 4 (Graviton, ~**30% faster**, smarter scaling) pay for what you consume
- **Global Database**  DB cluster + up to **10 secondary Regions**, storage-based replication (< 1 s lag, ~1.1 s typical), fast failover/switchover, secondary clusters serve reads, serverless readers supported
- **Parallel Query**  push analytic scans into the distributed storage (up to 2 orders of magnitude for certain queries, MySQL)
- **RDS Proxy**  connection pooling (Lambda) **Zero-ETL** to Redshift for analytics
- **Backups**  PITR (≤ **35 days**, 5-min txn logs) + snapshots **Backtrack** (MySQL)  rewind/wind without restore (up to 72h)
- **Storage auto-scaling** up to 64 TB IO included in Aurora  no separate I/O measure pay by storage + compute + I/O minutes (Aurora I/O-Optimized when bursty)

```
Client → writer endpoint ──► Aurora (writer DB instance)
        ├─ Reader endpoints ─► auto-scaling reader pool (read replicas)
        Shared storage volume replicated 6×/3 AZ, self-healing Backtrack
        Optional: Global Database secondary Regions Serverless ACUs Parallel Query
```

## 3. When to use

- **MySQL/PostgreSQL-compatible workloads** needing **higher performance**, higher availability, or more scale than plain RDS
- **Read-heavy apps with spiky/elastic reads**  auto-scaling reader pool
- **Unpredictable/variable capacity**  **Aurora Serverless** (pay-as-you-go, scale to zero  great dev/test, spiky, agentic/AI burst workloads)
- **Multi-Region DR + low write RPO**  **Global Database** (fast cross-Region failover, < 1 s replication, readers in secondary Regions)
- **Frequent "back to arbitrary point" restores**  **Backtrack** (instant, no full restore MySQL)
- **Serverless apps needing a SQL DB**  combine with **RDS Proxy**
- **Big MySQL/Postgres migration / consolidation** where current RDS instance is limiting

## 4. When NOT to use

- **Any supported non-MySQL/Postgres engine** (Oracle, SQL Server, Db2… strict) → **RDS**
- **Maximum per-instance control / exotic plugins / specific MySQL tuning** → self-managed/EC2 DB, or RDS classic
- **Tiny, cheap, 1-node, predictable** workloads  RDS single-AZ often cheaper upfront than Aurora cluster minimum
- **OLAP/analytics data warehousing at scale** → **Redshift** (though Zero-ETL + Parallel Query helps OLTP-side analytics)
- **NoSQL / key-value at massive scale** → **DynamoDB**
- **Document/graph/timeseries/ledger specialized** → DocumentDB / Neptune / Timestream / QLDB
- **Caching-only data** → ElastiCache
- **Real-time streaming for analytics** → Kinesis (not a DB job)

## 5. Important features

- **Distributed replicated storage**  6 copies across 3 AZs, self-healing, 64 TB max, 20 GiB increments ~**backups, crash recovery faster**
- **Endpoints**  writer/reader/cluster endpoints for traffic routing
- **Auto-scaling readers** (serverless readers too) failover < 30 s typical via storage-based recovery **Backtrack** (MySQL) up to 72 h rewind
- **Aurora Serverless**  ACU-based autoscaling (0.5–256), **scale to zero**, platform v4 ~30% faster (Spring 2026 update), ideal for spiky/agentic/dev/test
- **Global Database**  up to **10 secondaries**, storage-level replication (< 1 s lag typical ~1.1 s), **managed failover/switchover** to secondary Region with near-zero RPO
- **Replica features**  up to **15 Aurora Replicas**, cross-Region automated failover direction read load-balancing
- **Performance Insights, Enhanced Monitoring, CloudWatch, RDS Proxy, IAM database auth, KMS encryption**
- **Zero-ETL integrations** to **Redshift** (no pipeline for analytics) and with other AWS services **Aurora MySQL Parallel Query** (storage-side parallel scan)
- **Aurora DSQL**  recent multi-Region SQL offering at extreme scale (watch for exam mentioning "global strong consistency multi-region relational")
- **Aurora-exclusive high availability model**  no separate Multi-AZ standby needed a cluster's own replicas/readers + replicated storage give HA

## 6. Limitations

- **Only MySQL/PostgreSQL-compatible dialects**  not Oracle/SQL Server
- **No free-tier small single-standby planner**  Aurora clusters have compute + storage + I/O costs small single-AZ classic RDS can be cheaper for trivial stable OLTP
- **Global Database lag non-zero**  secondary Regions may lag (sub-second typical, not zero) strong cross-Region consistency ≠ DynamoDB global tables
- **Backtrack only MySQL, only up to 72 h**, requires the feature pre-enabled single cluster scope
- **Serverless ACU min 0.5**  not atomic-zero "scales to zero" means no capacity/charge when idle but cold start on first request
- **Write scale limited to single writer region**  one writer endpoint per cluster (Global DB = one primary writer)
- **Paralleled Query / features are MySQL-family-first**  PostgreSQL variants get some (limited) features later
- **PITR max 35 days** (like RDS) longer retention via snapshots/S3
- **Compute per instance classes**  reader auto-scaling has limits very heavy single-query loads still need right reader size
- **Custom storage tuning different**  you don't pick low-level EBS volume types (gp3/io1/io2)  storage is Aurora-managed
- **Some cursor/global variables/annoying MySQL edge** behaviors differ from classic RDS MySQL (Aurora-compat isn't byte-identical)

## 7. Trade-offs

- **Aurora vs RDS MySQL/Postgres**  Aurora engine + storage (6×, cheap per-GB management, auto readers, failover) at **~1/10th storage cost** in the long run vs RDS storage classes (gp3/io1/io2) + classic instances Aurora wins scale & HA RDS wins if strict engine-fidelity / smallest footprint
- **Provisioned (cluster) vs Serverless**  predictable steady (or fixed sizing) vs autoscale ACUs incl. zero (variable/spiky/predictable burst only)  Serverless now mature (platform v4)
- **Aurora Cluster (provisioned instances) vs RDS Multi-AZ**  Auto-replicas+multi-AZ storage vs RDS classic sync standby
- **Single-AZ vs Multi-AZ / Global Database**  local HA (+ 3-AZ storage) vs cross-Region failover (Global DB)
- **Parallel Query vs Redshift / Athena**  real-time transactional + analytic in same engine vs separate warehouse for heavy DW
- **Backtrack vs PITR**  instant in-place rewind (no restore) vs full restore/new cluster Backtrack better DX for dev/test
- **Aurora vs DynamoDB**  relational/joins/ACID-rich vs scale/global multi-active choose by data shape & consistency needs
- **Aurora vs DocumentDB**  MySQL-compat relational vs MongoDB-compat document (Aurora near-identical API)

## 8. Architecture

Reference patterns:

```
Spiky / dev–test / serverless → ALB→Lambda→RDS Proxy→Aurora Serverless (ACUs autoscale, scale-to-zero nights)

Read-heavy web → Aurora cluster: writer + auto-scaling reader pool → ALB/cache front Performance Insights tuning

Global/DR → Global Database: primary Region + secondary Region readers (+ serverless readers in backup Region)
                               switchover on primary-Region outage (near-zero RPO)

Analytics side → Aurora MySQL + Parallel Query (quick scans) OR Zero-ETL → Redshift for heavy DW
```

## 9. SAA-C03 Perspective

Aurora is a **high-availability, high-performance, and cost** answer (Domains 1, 2, 3, 4):

- **"MySQL/PostgreSQL-compatible at higher throughput / scale / availability than RDS"** → **Aurora**
- **"Elastic/unpredictable SQL capacity, pay per use"** → **Aurora Serverless** (scales to zero)
- **"Cross-Region DR with fast failover for MySQL/Postgres app"** → **Aurora Global Database**
- **"Undo schema mistakes back in time"** → **Backtrack** (MySQL, ≤ 72 h)
- **"Serverless Lambda + relational DB connection pooling"** → **RDS Proxy + Aurora**
- **"Analyze inside the same OLTP engine"** → **Parallel Query / Zero-ETL to Redshift**
- **Auto-scaling reader pool** vs RDS fixed read replicas
- **vs RDS**: classic engine parity + cheap predictable → RDS performance/scale/serverless/global → Aurora
- **Durability**: 6 copies across 3 AZs, self-healing storage = the headline differentiator

Exam traps: "Aurora runs Oracle" → **no (MySQL/PostgreSQL only)** "choose RDS storage IOPS" → **Aurora manages storage, no gp3/io2** "Global Database = zero lag" → **sub-second typical, not zero** "Serverless always zero cost" → **0.5 ACU min + cold-start caveat** "Backtrack also on PostgreSQL" → **MySQL only** "PITR > 35 days for free" → **no snapshots/S3 longer**.