# Amazon Athena — Serverless Interactive SQL over S3

## Purpose

Amazon Athena is a **serverless interactive query service** that lets you run **standard SQL directly on data in Amazon S3** — no infrastructure to manage, no loading, no ETL required. It's the go-to engine for **ad-hoc analysis of a data lake** on S3, paying **only for the data each query scans** (per-TB pricing, rounded to the nearest MB with a 10 MB minimum per query; DDL/partition statements are free).

## Main use cases

- **Ad-hoc SQL queries on S3 data lakes** — logs, clickstreams, CSV/JSON/Parquet/ORC files, S3 log/Trail exports
- **Query data "where it lives"** — analyze data in S3 without moving it into a warehouse (vs loading into Redshift)
- **Serverless analytics for BI** — a query layer atop S3 for QuickSight, Jupyter, or any JDBC/ODBC tool
- **Federated queries** — one SQL query that spans S3 **plus** relational/non-relational sources (via connectors) without copying data
- **Cheap archival/analysis of historical data** — VPC Flow Logs, CloudTrail, access logs
- **Building data pipelines** — CTAS (Create Table As Select) to transform/optimize data into Parquet or Iceberg

## Key features

- **Serverless & highly available** — no clusters, auto-parallel, results typically in seconds; scales automatically
- **Standard ANSI SQL** based on Presto; supports joins, window functions, arrays, user-defined functions (UDFs)
- **Formats**: CSV, JSON, ORC, Avro, Parquet (columnar + compression = 30–90% cost savings), plus Iceberg (insert/update/delete) and upcoming **S3 Tables** integration
- **Optimizations** — partitioning, columnar formats, compression reduce bytes scanned (and thus cost)
- **Managed schema** — tables defined **without ETL**, often via the **AWS Glue Data Catalog / crawlers**
- **Federated query** — connectors (many built-in) to query on-prem/AWS/multicloud sources in place
- **Governance & cost control** — **workgroups** (quotas, cost monitoring, results bucket), Athena for Apache Spark for interactive notebooks
- **Integration** — Glue Data Catalog, QuickSight/BI tools, EventBridge/CloudWatch, Lake Formation fine-grained permissions

## When to use

- Data is in S3 and you want SQL on top without standing up a warehouse
- You want **serverless**, pay-as-you-scan analytics (sporadic queries)
- VPC Flow Logs / CloudTrail / log-file analysis
- Quick joins of a data lake with other sources via federated query
- Where the "no infrastructure to manage" requirement dominates (vs Redshift cluster)

## Important limitation

- **Cost scales with data scanned** — every query scans then discards; without partitioning/columnar formats, costs and latency balloon. Not for **transactional OLTP** (no indexes, no UPDATE/INSERT Row-by-row), **sub-second/low-latency repeated lookups**, or constant big-cluster workloads where a **data warehouse (Redshift) / provisioned engine** is more cost-effective. Federated writes into external sources are not supported, and it's a **query engine, not a data warehouse** (no managed indexes, materialized-view acceleration beyond CTAS/Iceberg).

## SAA relevance

- "Query S3 at low cost, serverless, **standard SQL over data lake**" → **Athena**
- "No ETL / analyze logs / ad hoc SQL on S3" → Athena (pair with **Glue Data Catalog** + **crawlers** for schema)
- "Continuous/interactive BI with **strict SLAs on reserved capacity**" → **Redshift** instead
- "ETL/transform into Parquet" → **Glue** jobs; "repeated live ingestion" → Kinesis
- Exam traps: Athena = **serverless SQL on S3** (not a warehouse, not ETL), cost = **per bytes scanned** (minimize via Parquet + partitioning), Athena vs Redshift = **ad-hoc data lake queries vs provisioned columnar warehouse**.