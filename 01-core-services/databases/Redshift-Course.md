# Redshift Course — Petabyte-Scale Data Warehouse

## 1. Purpose

Redshift is AWS's **managed, column-oriented, massively parallel processing (MPP)** **data warehouse** — SQL analytics over **petabyte** datasets with queries that fan out across **slices** on many compute nodes. It's the right tool for **large analytical/BI workloads** that need joins + aggregations over big data (not OLTP, not simple KV). For SAA it answers **"analytics / data warehousing / BI / SQL over petabytes"**. Note: AWS announced **Python UDF retirement (June 30, 2026)** — relevant to current questions.

## 2. How it works

- **Cluster**: leader node + compute nodes; data stored **column-wise**, **compressed**, distributed across **slices**
- **Distribution styles** — **EVEN** (round-robin), **KEY** (hash on a column — keeps joins local), **ALL**/**AUTO**; **sort keys** for range scans
- **Node types (2026)**:
  | Family | What | Best for |
  |---|---|---|
  | **RG (Graviton, new May 2026)** | up to 2.2x DW / 2.4x data-lake speed vs RA3, ~30% lower $/vCPU, **integrated data-lake engine** (Spectrum-style queries run on cluster, no per-TB scan charge) | modern DW + heavy lake queries |
  | **RA3 (managed storage)** | compute & storage independent; hot on SSD + auto-tier to S3; Spectrum (dedicated fleet) | growable DW, separate compute/storage |
  | **DC2 / DS2** | dense compute / dense storage | < 1 TB compressed (DC2) / legacy |
- **Managed storage** — RG/RA3: hot SSD per node + auto-move cold data to S3, same price regardless of tier
- **Redshift Serverless** — auto-scaling, no cluster, pay per use, automatic WLM — for unpredictable ad-hoc
- **Query execution** — **automatic WLM** (queues, priorities, short-query acceleration), **concurrency scaling** (adds capacity for peak concurrency, uses earned credits)
- **Data ingestion** — COPY from S3 (Parquet/ORC/JSON/CSV), **Spectrum** (query S3 data lake directly, external tables via Glue/Athena catalog, no load), **Kinesis/Firehose → Redshift**, **Zero-ETL** from Aurora/RDS, CDC, federated queries
- **Distribution & slicing** → query parallelism; **materialized views** for repeated aggregations
- **Backup** — automated snapshots (retention), **cross-Region snapshots** for DR; restore to new cluster; **S3** archival

```
ETL/Stream → (S3 → COPY) / Spectrum (external tables on S3) / CDC / Zero-ETL
   → Redshift cluster (leader + compute nodes, columnar, distributed slices)
      ├─ auto/short WLM + concurrency scaling → concurrent BI/analytics
      ├─ materialized views; VL/session workers; VPC + IAM + KMS
      └─ snapshots → DR / restore; S3 export (UNLOAD)
   Serverless option: Redshift Serverless (no cluster, auto-scale, per-use)
```

## 3. When to use

- **Data warehousing / BI reporting / SQL analytics** on large (**GB → PB**) datasets
- **Ad-hoc & recurring complex analytical queries** (joins, aggregations, windows) at scale
- **Dashboards, enterprise reporting, ELT/ETL targets** with predictable response
- **Querying a data lake in S3 directly** — **Spectrum** / RG integrated lake engine (skip loading)
- **Unpredictable ad-hoc analytics w/o managing clusters** → **Redshift Serverless**
- **Combining transactional + analytical** — **Zero-ETL from Aurora/RDS** (no pipeline)
- **Federated queries** against RDS/Aurora/S3 (single query across sources)
- **When Olahpub requires separate analytics engine from OLTP** (unlike Over-kill Aurora for DW)

## 4. When NOT to use

- **OLTP / transactional row updates, high write concurrency** → **RDS/Aurora** (Redshift is batch-analytics-oriented, not row-ACID OLTP)
- **Key-value / single-digit-ms / serverless NoSQL** → **DynamoDB**
- **Small analytical datasets (< 1 TB, no model)** — Athena on S3 often simpler/cheaper (ad-hoc, no cluster)
- **Very real-time streaming analytics with ms-level** → Kinesis Analytics / OpenSearch (Redshift is batch/near-ETL)
- **Simple interactive queries on small data** → Athena over S3 or QuickSight+view
- **Graph / time-series / document** specialized engines
- **Google-style serverless occasional queries** → Athena/Spectrum may suffice; don't over-provision DW
- **Caching hot row-level fetches in front of an app DB** → ElastiCache

## 5. Important features

- **Columnar + compression + MPP** — analytics on huge tables fast; distribution/sort-key design matters
- **Manageability** — automated snapshots (retention), **cross-Region replication** for DR, restore to new cluster, resize/pause
- **Managed storage (RG/RA3)** — compute/storage independent; auto-tier SSD→S3; no re-size plumbing
- **RG (Graviton, 2026)** — 2.2x DW / 2.4x lake perf vs RA3, 30% lower $/vCPU, integrated data-lake engine (no Spectrum per-TB scan fee)
- **Redshift Serverless** — scale compute to zero/up on demand; automatic WLM; ACU-based
- **Spectrum** — query S3 external tables without ETL/load; partitions; joins local+distant (lazy); RG = cluster-resident lake queries
- **Automatic WLM + concurrency scaling** — dynamic queues; short-query acceleration; credit-based concurrency burst
- **COPY/UNLOAD** to S3; Parquet/ORC/JSON; **materialized views**; **federated query** to RDS/Aurora; **Zero-ETL** with Aurora/RDS/Timestream
- **Query optimize** — RESULT cache, Auto MV, `EXPLAIN`, workload insights views (`SVL_…`)
- **Security** — VPC-only, IAM, KMS, SSL, row/column-level security, dynamic data masking
- **Monitoring** — CloudWatch metrics, `STL_` / `SVL_` / `SYS_` query logs; snapshots & autoscaling alerts
- **Data sharing** (RA3/RG) — share live warehouses across clusters without copy
- **ML** — Redshift ML (predictions in SQL via SageMaker), Amazon Q for Redshift query ANSIs

## 6. Limitations

- **Not OLTP** — poor fit for high-frequency row-wise transactional writes; heavy UPDATE/DELETE costly
- **Single-leader cluster semantics** — one write path (big-query focused); massive concurrent writers need streaming/landing patterns
- **Node types fixed at launch** — RG vs RA3 vs DC2 trade by sizing; type changes = restore/rebuild path
- **Long-running complex queries** still require optimizations (sort/dist keys, auto-VACUUM, proper distribution)
- **Concurrency scaling uses credits** — large sustained peaks can exhaust credits (add nodes or Serverless)
- **Data has skew** on distribution key — hotspot shards hurt performance
- **Compression/columnar natively** — ad-hoc heavy mutate workloads will degrade vs append
- **COPY bottleneck if no proper sizing** — need good staging from S3
- **Spectrum latency** — scans external S3 slower than local tables; per-TB scan charges on RA3/DC2 (RG includes lake engine), optimize partitioning/filtering
- **Python UDFs sunset June 30, 2026** — migrate to native SQL/other scripting
- **Backup/restore time scales with data** — large warehouses restore slower than DynamoDB-style instant
- **GIS/JSON flexibility lighter** than Postgres engine parity (uses SQL dialect, not exact Postgres)

## 7. Trade-offs

- **Redshift vs Athena** — provisioned DW (perf, joins at scale, BI) & flexible/long-running vs S3 + SQL-on-demand (cheap, serverless, slower scans); if you only query S3 occasionally at small scale → Athena
- **Redshift vs Aurora/DynamoDB** — analytics/BI on big data vs OLTP/key-value; different workloads
- **Redshift Serverless vs provisioned** — ad-hoc/unpredictable, zero-manage, per-use vs predictable long-lived heavy workloads (RG) — credits vs hourly per-node
- **RG vs RA3 vs DC2** — latest Graviton (perf + lake engine, 30% cost) vs RA3 managed-storage standard vs DC2 cheap < 1 TB fixed
- **Managed storage vs DC2/DS2** — independent scale/pay vs fixed local, lower upfront
- **Spectrum vs COPY load** — query in-place vs load for performance; lake-first vs DW-first
- **Automatic WLM vs manual WLM** — easy/mostly-right vs fine-grained control for mega-specialized workloads
- **Materialized views vs direct query** — pre-aggregate repeated workloads vs freshness/latency on live data
- **Data sharing vs copying** — share warehouse read live across clusters vs duplicate; zero-copy, cross-account scoped
- **Redshift vs Redshift + S3 lakehouse** — often system-of-truth + lake in S3 with Spectrum/Glue; AWS pushes lakehouse architecture now

## 8. Architecture

Reference patterns:

```
Core DW:
  ETL (Glue/Lambda) → S3 (Parquet) → COPY → Redshift RG cluster (EVEN/KEY + sort, MVs)
   → BI (QuickSight/Tableau) via ODBC/JDBC; WLM priority + concurrency scaling; snapshots cross-Region
   → near-real-time: Firehose → Redshift (COPY staging) + Zero-ETL from Aurora/RDS for fresh Ops data

Lakehouse:
  S3 raw → Glue catalog → Redshift Spectrum (or RG integrated) external tables → join local + lake
  (no ETL duplication; once-query)

Serverless analytics for dev:
  Redshift Serverless → ad-hoc SQL on S3/per-cluster; pause on idle → stop costs

DR:
  snapshot → S3 → cross-Region restore → new cluster (automated pattern) 
```

## 9. SAA-C03 Perspective

Redshift is the **analytics/data-warehouse** answer (Domains 3, 4; light in Domain 1 security):

- **"Data warehousing / BI / analytical SQL on huge data"** → **Redshift**
- **"Query the S3 data lake without loading"** → **Redshift Spectrum** (or RG integrated lake engine / Athena)
- **"Unpredictable or no-cluster admin ad-hoc analytics"** → **Redshift Serverless**
- **"Separate compute from storage, scale independently"** → **RA3/RG managed storage**
- **"Fresh analytics from Aurora/RDS without pipeline"** → **Zero-ETL**
- **"Analyze across RDS (Aurora) + S3 + Redshift in one query"** → **federated query**
- **Node-type story**: RG→Graviton best perf + lake engine; RA3→managed storage; DC2→<1 TB cheap
- **Distribution/sort key** design = query performance; EVEN / KEY / ALL
- **automatic WLM + concurrency scaling** for concurrent BI users; watch credit exhaustion
- **COPY & UNLOAD ↔ S3; Parquet**; snapshots + cross-Region for DR
- **⚠ Python UDFs end June 30, 2026**
- **vs Athena**: provisioned DW vs serverless S3 queries — scenario wording decides

Exam traps: "Redshift for OLTP" → **no (RDS/Aurora)**; "Redshift Spectrum loads data first" → **no, queries in place**; "all events must load" → **no (Spectrum/Zero-ETL/streaming)**; "concurrency scaling unlimited/free" → **credit-based**; "DC2 stores beyond 1 TB well" → **use RA3/RG**; "data always consistent single-writer" → lead-only write; "Python UDFs going strong" → **retired June 2026**; "Serverless = no limits" → ACU-based usage limits exist.