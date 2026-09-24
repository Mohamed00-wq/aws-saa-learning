# Amazon OpenSearch Service Course — Search, Log Analytics & Observability

## 1. Purpose

Amazon OpenSearch Service is AWS's **fully managed OpenSearch (Apache 2.0) engine** (Elasticsearch 7.10-compatible lineage) for **full-text search, log analytics, observability, and now vector / AI search**. You deploy a **domain** (cluster) with just a few clicks, including security, HA, snapshots, and *UltraWarm/cold* storage tiers. For SAA it's the answer to **"full-text search / log & application monitoring / sub-second analytics on semi-structured data"** (Kibana/OpenSearch Dashboards accessible).

## 2. How it works

- **Domain = cluster** — running data nodes + optionally **dedicated master nodes**; supports up to **1002 data nodes / up to 25 PB** of attached storage (cost-efficient **Graviton** instances)
- **Index documents** via REST (JSON maps to analyzed/searchable fields) — full-text search with relevance scoring, aggregations, filters, fuzzy/complex queries
- **Ingest** — directly (REST), via **OpenSearch Ingestion** pipelines, Kinesis Data Firehose, Kinesis Data Streams, CloudWatch Logs subscription, S3/Lambda (incl. **zero-ETL integrations with S3, DynamoDB, DocumentDB**)
- **Query** — search APIs, **SQL + PPL** (piped processing language), OpenSearch **Dashboards** (visualization, alerting, anomaly detection)
- **Storage tiers** — **hot** (default), **UltraWarm** (S3-backed warm nodes for cold-ish data, up to **3 PB** per domain), **cold storage** (S3, detached — retain virtually any amount cheaply), **request storage, Index State Management (ISM)** automates migration between tiers
- **Security** — IAM, VPC + SGs, **fine-grained access control** (index/doc/field-level), encryption at rest & node-to-node TLS, auth via **Amazon Cognito**, SAML, or basic
- **OpenSearch Serverless** option — on-demand auto-scaling collections (no domain sizing)
- Cross-cluster search/replication, automated snapshots, CloudWatch integration

```
S3 / DynamoDB / DocumentDB (zero-ETL) ─┐
CloudWatch Logs / Firehose / Kinesis ──→ OpenSearch ingestion → OpenSearch domain
OpenSearch SQL / PPL / Dashboards / alerting ← queries & visualizations
Tiers: hot (fast) → UltraWarm (S3-backed, 3PB) → cold (S3, detached) via ISM
```

## 3. When to use

- **Full-text / flexible / fuzzy search** — product/site search, e-commerce, document search, website search
- **Log & application analytics / observability** — centralize app logs (CloudWatch Logs subscription, Firehose), trace & SIEM
- **Operational search over semi-structured JSON** where SQL joins aren't needed; free-text + aggregations dashboards
- **Vector / semantic search, neural/ML-powered search** and RAG backends
- **Monitoring/alerts & anomaly detection** on metrics/logs; Kibana/OpenSearch Dashboards end users

## 4. When NOT to use

- **Transactional OLTP / relational joins / ACID** → RDS/Aurora/DynamoDB
- **Numeric-heavy OLAP data warehousing** with large scans → **Redshift / Athena**
- **Key-value single-digit-ms at huge throughput** → DynamoDB
- **Simple structured log search at small scale** (barely-worth-an-index case) → CloudWatch Logs Insights
- **Primary storage for the application** — OpenSearch is search/analytics *over* ingested data (index freshness, eventual consistency), not a system of record

## 5. Important features

- **Full-text search, aggregations, filters, relevance scoring, suggestion/autocomplete**
- **OpenSearch Dashboards / Kibana-compatible** — visualization, SQL/PPL workbench, alerting, anomaly detection
- **UltraWarm (warm, up to 3 PB) & cold (S3) storage tiers** + **ISM** policies — index lifecycle automation
- **Zero-ETL integrations** with S3, DynamoDB, DocumentDB; ingests from Kinesis/Firehose/CloudWatch Logs
- **Vector engine** for neural/semantic search (RAG-ready)
- **Security** — fine-grained access control, IAM/Cognito/SAML, encryption at rest/in transit, VPC, audit logs
- **Cross-cluster search & replication**, automated snapshots, up to 1002 nodes/25 PB, Graviton
- **OpenSearch Serverless** collection option; 99.99% SLA (multi-AZ with standby)

## 6. Limitations

- **Index freshness / eventual consistency** — not a source of truth; batch/stream latency before searchable
- **Operational cost** — nodes are EC2 (Graviton helps); shard sizing/index design matters for performance
- **Not for OLTP transactions or broad relational analytics** — it's search/observability
- **Query performance depends on index mapping** — poorly-designed indexes degrade (heap, shards)
- Learning curve of the query DSL/analyzer concepts; clusters have scaling quotas

## 7. Trade-offs

- **OpenSearch vs DynamoDB/RDS** — search/analytics/free-text over semi-structured data vs transactional/key-value storage (use OpenSearch *in addition to* your store, index via streams)
- **OpenSearch vs Redshift/Athena** — interactive search, logs, observability, dashboards (sub-second on indexed data) vs large-scan analytical SQL on columnar stores
- **vs CloudWatch Logs** — central cross-account search + Kibana-style analysis/SIEM vs AWS-native log mgmt for AWS services (CW Logs Insights)
- **Provisioned domain vs OpenSearch Serverless** — tuning/sizing control vs auto-scaling serverless collections (pay per use)

## 8. Architecture

```
E-commerce product search:
  Catalog writes → DynamoDB (source of truth) ─ zero-ETL / streaming → OpenSearch index
  Users → OpenSearch search API (fuzzy, filters) → results; suggestions/analytics in Dashboards
Observability:
  App logs → CloudWatch Logs → subscription → OpenSearch Ingestion → domain
  ISM: hot 7d → UltraWarm → cold (S3) 90d; SNS alerts on threshold breaches
```

## 9. SAA-C03 Perspective

- **"Full-text / fuzzy / flexible search over JSON"** → **OpenSearch**
- **"Log analytics / application monitoring / Kibana-Dashboards dashboards / SIEM"** → **OpenSearch**
- **"Free-text across a large corpus where SQL is wrong; sub-second results"** → OpenSearch
- **"Vectors / semantic search (RAG)"** → OpenSearch vector engine
- **"Relational analytics at scale / data warehouse"** → Redshift/Athena, not OpenSearch
- **"Key-value serverless store"** → DynamoDB (OpenSearch complements it as an index)

Exam traps: "OpenSearch = relational DB" → **no, search/index engine**; "OpenSearch for OLTP/transactions" → **no, it's eventual-consistency search**; "you manage Elastic/OpenSearch nodes" → **fully managed**; "UltraWarm=cold cheapest" → **UltraWarm is warm/S3-backed up to 3PB; cold storage is the S3-detach tier**.