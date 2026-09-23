# DMS Course — Database Migration Service

## 1. Purpose

DMS is AWS's **managed service to migrate databases to AWS (and between AWS databases) with minimal downtime** — replicating data between widely-used DB engines while your source keeps serving traffic. Combined with **AWS Schema Conversion Tool (SCT)** / **DMS Schema Conversion**, it also handles **heterogeneous** migrations (engine→engine, e.g., Oracle→PostgreSQL). It's the canonical answer for **"migrate our Oracle/MySQL/SQL Server to AWS with ~zero downtime"**.

## 2. How it works

- Three core components:
  - **Replication instance** — a managed (EC2-based) compute host that runs migration tasks; **DMS Serverless** auto-scales for long/bursty migrations
  - **Source endpoint** and **target endpoint** — connections (on-prem, RDS, EC2, Aurora, S3, Kinesis, etc.)
  - **Replication task** — defines tables, transform rules, and migration type
- **Migration types**:
  - **Full load** — one-time copy of existing data
  - **Full load + CDC** — initial copy THEN **ongoing change data capture** (keeps source & target nearly in sync) — the standard **minimal-downtime** cutover
  - **CDC only** — replicate changes once target already has data
- **Homogeneous** (same/compatible engine: MySQL→Aurora, Oracle→RDS Oracle, SQL Server→SQL Server) — one-step; optional native tools ("Homogeneous data migrations" using DB-native copy) or DMS
- **Heterogeneous** (different engines) — **SCT first** converts **schema + stored procedures/functions/views**, flags manual fixes, then DMS moves data
- **CDC reads source transaction logs** (binlog/WAL/reDo) and applies changes to target with low latency (no SLA on latency); **data validation** compares source/target to catch drift
- **Task logging/metrics** — CloudWatch (latency, throughput), SCT assessment reports

```
SCT/DMS Schema Conversion → convert schema + code (heterogeneous); review action items
   → DMS replication instance (or DMS Serverless)
      → tasks: Full Load / +CDC(catch-up) / CDC-only → endpoints (source/target)
      → validation + cutover; monitor latency via CloudWatch; source stays online during migration
```

## 3. When to use

- **Minimal-downtime DB migration to AWS** (on-prem → RDS/Aurora/EC2/Redshift/S3/DynamoDB/OpenSearch/…)
- **Homogeneous lift** — MySQL→Aurora, Oracle→RDS Oracle, SQL Server→RDS SQL Server (or DocumentDB/other target)
- **Heterogeneous** — Oracle→PostgreSQL / Aurora, SQL Server→MySQL, etc., using **SCT** for schema/code conversion
- **Ongoing replication / copy data with CDC** — replicate between two AWS databases, keep environments in sync during window
- **Consolidation** — many sources into one target (scale down fleet)
- **Data validation between source and target** after/­during replication
- **DMS Serverless** — unknown/large/bursty migration profiles without instance sizing

## 4. When NOT to use

- **Same-source data designed for pure schema translation at massive estate** — you still need SCT + DMS, but plan app code / extracted-SQL conversion too
- **Very fast one-off bulk move of SQL data that can tolerate downtime** — dump/restore (mysqldump, pg_dump, Oracle Data Pump) may be simpler/cheaper
- **Migrating large object files / filesystem data** — that's **DataSync** / Snow Family (DMS is databases)
- **Migrating entire servers/apps (not just DB)** — **Application Migration Service (MGN)**
- **Real-time event streaming substitution** — DMS is for DB sync; use Kinesis/MSK for streams (though DMS can extend one-way to S3/Kinesis)
- **Backup/restore of unrelated non-part "<db>" store** — check the proper storage tool (S3, EFS…)
- **Zero-downtime continuous cutover needs extreme velocity with smaller dataset** — Snowball Edge (initial bulk) + DMS CDC is a real hybrid pattern
- **Source unsupported engine/deep proprietary types** — verify DMS source matrix before planning

## 5. Important features

- **Broad matrix** — sources/targets: RDS/Aurora (MySQL, Postgres, Oracle, SQL Server, MariaDB), EC2 DBs, on-prem via VPN/Direct Connect, **S3, DynamoDB, Redshift, OpenSearch, Kinesis, Kafka, DocumentDB, Neptune, Babelfish** target, etc.
- **Minimal downtime** — **Full Load + CDC**: copy then stream changes; app switches at cutover
- **DMS Schema Conversion (managed)** — assessment report of conversion complexity + guided migrations (Oracle→Aurora PostgreSQL, SQL Server→MySQL, etc.); **SCT** legacy desktop shedding also works (assess + convert code, app SQL)
- **Validation** — automatic **row-count / checksum / full-table validation** between source & target
- **Transformation rules** — rename, drop, re-partition, task-filters (include/exclude tables), custom DB mappings
- **Change data capture (CDC)** — ongoing sync; near real-time (latency variable; no SLA)
- **Replication instance sizing** — dms.* classes; **Multi-AZ** option for task continuity; **DMS Serverless** (auto-scaling, no sizing)
- **Security** — SSL/TLS, NAT/PrivateLink endpoints, IAM roles for source/target, KMS encryption, VPC isolation
- **Monitoring/logging** — CloudTrail, CloudWatch metrics (CDCThroughputRows, CDCLatencySource/Target), task logs to S3; task failure=restart, redrive
- **On-prem connectivity** — over Site-to-Site VPN / Direct Connect for private network migration
- **Consolidation** — multiple sources to one target (homogeneous & heterogeneous)
- **Load and encrypt** — migration to "S3" target supports Parquet/CSV+ formatting for lake

## 6. Limitations

- **CDC latency not SLA'd** — real time varies by load, can lag under heavy write rates
- **Data-types edge cases** — some exotic types may need manual mapping; SCT flags action-items
- **Views migratable only in full-load** (CDC tasks include tables only — reconcile separately)
- **Bidirectional replication** — has loopback/config caveats (no conflict detection/resolution — use validation; `BatchApplyEnabled=false` recommended)
- **Replication instance scale** — even DMS Serverless has limits; very large/fluctuating loads need burst-tolerant config
- **Limit on source "ALTER TABLE" semantics** — schema changes post-start require task rework/restart
- **Landing to S3/stream targets** — DMS-to-S3 has partitioning/format limits vs COPY-style
- **Source DB load during CDC** — adds pressure to source redo/logs; need source change-log availability
- **Requires changes on source (some engines) enabling logging/binlog** — not zero-footprint
- **Extra components to operate**: if migration is trivial/downtime-tolerant, dump/restore simpler
- **Cost** — hourly replication instance + logs during migration window; Serverless cost model per-use

## 7. Trade-offs

- **DMS vs dump/restore** — minimal downtime + CDC + heterogeneous + validation vs dead-simple, cheap, offline
- **DMS vs DataSync** — databases with CDC vs file/shares sync (DataSync for filesystem, not DB tables)
- **DMS vs MGN (Application Migration Service)** — DB tables vs **whole servers** (OS+apps); combine DMS + MGN for full-different migrations
- **DMS vs Snowball Edge** — direct online migration vs **bulk offline initial load** (petabytes) then DMS for CDC tail
- **DMS Serverless vs provisioned replication instance** — auto-scaling per-use vs predictable hourly sizing/control
- **Homogeneous (native/DMS) vs heterogeneous (SCT+DMS)** — few steps for same-engine vs schema+code conversion project for cross-engine
- **DMS Schema Conversion (managed) vs SCT desktop** — guided assessment in AWS vs local full conditioning + extraction of embedded app SQL
- **Full Load+CDC vs CDC-only** — greenfield initial copy+sync vs only delta onto preloaded target
- **Consolidation vs per-DB tasks** — single fleet to one DW vs isolated pipelines (inc. RPO isolation)
- **Validation on/off** — data-integrity assurance vs extra compute/time on task

## 8. Architecture

Reference migration patterns:

```
Standard minimal-downtime:
  SCT (schema/code assessment, convert → target schema) →
  DMS provisioned/Serverless → endpoints (on-prem via Direct Connect/VPN, target RDS/Aurora Multi-AZ)
  → Full Load + CDC → wait for target caught up + validation → flip app DNS → cutover; keep/revert path

Consolidation / lake:
  Many RDS/on-prem Oracle sources → DMS tasks → single Redshift (or S3+Parquet) target → analytics; ongoing CDC

Bulk:
  On-prem Oracle ≤ 50TB → Snowball Edge (initial load to RDS/Aurora via Snowball) → DMS CDC tail → cutover

DR/duplicate:
  Aurora writer → DMS→ S3/stream; or Aurora→Aurora cross-account copy with CDC for DR segregation
```

## 9. SAA-C03 Perspective

DMS mostly sits in **Domain 4 (migration)** and sometimes **cost/resilience**:

- **"Migrate on-prem Oracle/MySQL/SQL Server to AWS with < downtime"** → **DMS** (Full Load + **CDC**), validate, cutover
- **"Same engine"** → **homogeneous** (simple DMS); **"different engine"** → **add SCT** / DMS Schema Conversion for schema+code
- **"Continue replicating ongoing changes after initial copy"** → **CDC**
- **"Validate nothing lost"** → **data validation**
- **"Keep systems synced during migration"** → DMS replication; source stays online
- **"Very large DB initial load"** → **Snowball Edge bulk + DMS CDC**
- **"Servers not DB"** → **MGN**, not DMS; **"files"** → **DataSync**
- **Targets**: RDS, Aurora, Redshift (DW), S3 (lake/Parquet), DynamoDB; tasks monitored via CloudWatch (CDC latency metrıcs)
- **DMS Serverless** for no-sizing migrations; **Multi-AZ** replication instance for task continuity
- Costs while migration runs = instance-hours + storage of replication logs

Exam traps: "DMS replaces dump/restore with more features" → **not always — if downtime acceptable, dump is fine**; "DMS moves whole server" → **no (MGN)**; "DMS moves files" → **no (DataSync/Snow)**; "heterogeneous without SCT" → **no — SCT/DMS-SC needed for schema/code**; "instant zero-downtime everywhere" → CDC has latency & source-log requirements; "single source only" → **consolidation supported**; "views in CDC task" → **full-load only caveat**; "locked restarts" → source schema changes mid-task need restart; "no cost during migration" → **instance-hours billed**.