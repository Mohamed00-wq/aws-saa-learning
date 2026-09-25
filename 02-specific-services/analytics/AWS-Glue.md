# AWS Glue  Serverless ETL, Crawlers & the Data Catalog

## Purpose

AWS Glue is **serverless data integration (ETL)** so that data can be prepared for analytics, ML, and applications **without managing Spark clusters**. It pairs two big things: **crawlers** that auto-discover schemas into the centralized **AWS Glue Data Catalog** (metadata store), and **Apache Spark-based ETL jobs** (plus Python Shell and streaming jobs) that move and transform data between sources  e.g. **S3 → Redshift**. You pay per **DPU-hour** ($0.44/DPU 1 DPU = 4 vCPU + 16 GB), billed per second.

## Main use cases

- **ETL pipelines**  batch transforms and loads between services (S3, Redshift, RDS, JDBC, DynamoDB, Kafka/Kinesis)
- **Schema discovery & cataloging**  crawlers read S3/data sources, infer schemas, populate the **Data Catalog** (used by Athena, Redshift Spectrum, EMR, QuickSight)
- **Streaming ETL**  continuous (Spark Streaming) jobs consuming **Kinesis / MSK**, cleaning/enriching in-flight
- **Data quality & prep**  Data Quality checks, Glue Studio visual transforms, **DataBrew** (no-code), **FindMatches** (ML dedup)
- **Schema Registry**  validate/enforce Avro schemas for streaming producers/consumers (no separate charge)

## Key features

- **Crawlers**  auto-infer schemas/partitions into the Data Catalog (scheduled, on-demand, or event-triggered)
- **ETL jobs**  Scala/Python on serverless **Spark**, Spark Streaming, or lightweight **Python Shell** triggers, scheduling, **workflows** with dependencies, retries **job bookmarks** for incremental processing
- **Glue Studio**  visual drag-and-drop job authoring **Interactive Sessions** for notebooks **autoscaling** (Glue 4.0+)
- **Data Catalog**  the standard AWS metastore (tables, partitions, schemas, catalog objects) that Athena/Redshift Spectrum/EMR query
- **Data sources/connectors**  S3, JDBC DBs, MongoDB, Kafka, Kinesis + Marketplace connectors streaming ETL out of the box
- **GenAI-assisted debugging & Spark upgrade plans** Schema Registry **Deep Lake Formation integration** for fine-grained security

## When to use

- You need managed, serverless Spark-based ETL **without operating EMR**
- Schema/table discovery for Athena + Redshift Spectrum (crawlers feeding the Data Catalog)
- S3-to-Redshift and DB-to-DL ETL pipelines streaming transformations from Kinesis/MSK
- Teams that want visual ETL (Studio) or no-code (DataBrew) with Spark underneath
- When you want a single metadata catalog across all analytics services

## Important limitation

- **Spark job startup (cold start) 1–2 minutes** and **DPU-hour cost** make it poor for very frequent/short or always-on low-volume jobs (consider Lambda/Step Functions or Python Shell for small transforms). **In-precision of crawler schema inference** can require manual corrections advanced control (custom Spark tuning) is **limited vs self-managed EMR**. Long-running streaming jobs accrue continuous DPU cost, and small always-on use can be cheaper with Kinesis Data Analytics or Lambda-based pipelines.

## SAA relevance

- "**Serverless ETL** between data stores (esp. **S3 → Redshift**) / transform data" → **AWS Glue**
- "**Crawlers infer schemas** into the **Data Catalog** for Athena/Redshift Spectrum" → Glue
- "Serverless **Spark** jobs on AWS" → Glue (vs **EMR** = you manage clusters, more control, non-serverless)
- "DR/restore Glue jobs" → remember jobs + Data Catalog/finder metadata are Region-scoped resources replan for Region failover
- Exam traps: Glue = **serverless ETL + metadata catalog** (not a query engine/warehouse) crawlers ≠ ETL (they *discover* schemas) Athena *uses* the Catalog Glue created for always-on high-volume processing choose **EMR** for control, **Kinesis Data Analytics/Firehose** for streaming simplicity.