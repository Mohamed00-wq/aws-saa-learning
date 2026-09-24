# Amazon DocumentDB Course — MongoDB-Compatible Document Database on Aurora

## 1. Purpose

Amazon DocumentDB is a **fully managed, MongoDB-compatible document (NoSQL) database** built on the **Aurora storage engine**. It runs MongoDB 4.0/5.0/6.x-compatible workloads (wire protocol/drivers) with Aurora's reliability: storage replicated **6 copies across 3 AZs**, auto-healing, self-managed backups. For SAA it's the answer to **"MongoDB / document model / JSON, wants Aurora-like managed durability"**.

## 2. How it works

- **Clusters** with a single **primary write instance** + up to **15 read replicas** (reads scale out; replicas and primary share the same **Aurora-style storage**, so no write amplification at replicas)
- **Storage** auto-scales to **64 TB max**, 6-way replicated across 3 AZs, daily snapshots + **PITR up to 35 days**
- **No native horizontal write scaling** (single writer) — you scale writes by sharding at the application layer or using bigger instances (Node sizes up to ~30,000 concurrent connections / 2TB+ memory classes)
- **Change streams** (CDC), **TTL indexes** for automatic expiry, **transactions**, aggregation pipeline support
- **Global cluster** option — read replicas in other Regions (storage-level replication), promote for DR
- Natively accessible with MongoDB drivers (change the connection string), query via `mongo` shell compatible API (allows Mongo 4-compatible APIs; some 5/6 features vary)

```
App (Mongo drivers) → primary (writes, 1)
   ├─ up to 15 read replicas (same storage, reads scale out)
   └─ Aurora storage: 6 copies / 3 AZs, auto-scaled, PITR 35 days
Change streams / TTL / transactions available; global cluster: replicas cross-Region
```

## 3. When to use

- **Document/JSON flexible-schema workloads** — CMS, catalogs, mobile/partner metadata, profiles
- **Migrating from MongoDB** to a fully managed AWS service (keep Mongo drivers/APIs, minimize code change)
- **Need MongoDB query language, aggregations, $-operators** without running servers
- **Multi-AZ durability with Aurora-level storage reliability** and autoscaling reads
- **Workloads with the write amplification/backup pain of self-managed Mongo**

## 4. When NOT to use

- **Key-value / single-digit-ms at millions of rps / global active-active** → **DynamoDB**
- **Relational ACID with joins / SQL** → **RDS/Aurora (PostgreSQL/MySQL)**
- **Need ALL native MongoDB features** (e.g., some $ operators, aggregation stages, sharding transparency, certain server features) → consider **MongoDB Atlas** or self-managed (DocumentDB supports a large subset of Mongo 4.0/5.0/6.0 but not everything)
- **Heavy multi-region active-active writes** → DocumentDB is single-writer; consider DynamoDB global tables or Aurora Global Database
- **Analytics / columnar scans** → Redshift / Athena

## 5. Important features

- **MongoDB 4.0/5.0/6.0 wire-protocol compatibility** — existing drivers/apps need only connection-string change
- **Aurora storage engine** — 6 copies/3 AZs, auto-scaling to **64 TB**, self-healing, instant-node-failover, incremental snapshots + **PITR ≤ 35 days**
- **Up to 15 low-latency read replicas** (reads scale; good for read-heavy apps)
- **Change streams**, **TTL indexes**, **transactions**, **aggregation pipelines**
- **Global clusters** — cross-Region read replicas + promote for disaster recovery
- **Encryption at rest (KMS)/in transit (TLS), IAM, VPC, audit logs**
- **Serverless v3** option (autoscaling ACUs) for unpredictable workloads

## 6. Limitations

- **Single-writer architecture** — writes don't horizontally scale natively (app-level sharding needed for very high write volumes)
- **Not 100% MongoDB** — subsets/semantics of Mongo 5/6 features, some operators/stages, and the "sharding transparency" of MongoDB Atlas may not match
- **No fully managed native horizontal sharding** (unlike Atlas)
- Complexity tuning for large-aggregation workloads; instance-sizing develops to manage hot partitions
- Global cluster supports cross-Region **reads**, active-active writes require app-level handling (LWW etc.)

## 7. Trade-offs

- **DocumentDB vs DynamoDB** — document/a-GraphQL/JSON with rich aggregation & Mongo compatibility vs key-value/serverless/millions-rps/global tables (DynamoDB wins for huge-key/value scale, DocumentDB wins for document/Mongo workloads)
- **DocumentDB vs Aurora (relational)** — document model + Mongo drivers vs SQL/relational features
- **DocumentDB vs MongoDB Atlas** — fully managed on AWS, Aurora-durable, pay per hour (no Atlas) vs full native MongoDB (incl. native sharding but managed by a different company & pricing)
- **vs self-managed MongoDB on EC2** — ops-free, Aurora durability vs total control/customization

## 8. Architecture

```
Read-heavy content service:
  Mongo driver → DocumentDB cluster (primary for writes, N read replicas behind ALB/proxy)
  ├─ change streams → Lambda → search index (OpenSearch)
  ├─ TTL index cleans stale sessions/catalog cache
  └─ global cluster: US primary, EU replica (read-local), promote for DR
DB size auto-scales; PITR ≤ 35 days + snapshots
```

## 9. SAA-C03 Perspective

- **"MongoDB compatible / document database / JSON, fully managed"** → **DocumentDB**
- **"Migrate existing MongoDB to managed AWS without rewriting code"** → **DocumentDB** (keep drivers)
- **"Up to 15 read replicas scalable reads, single writer"** → DocumentDB
- **"Key-value, sub-ms, unlimited scale, serverless"** → NOT DocumentDB — **DynamoDB**
- **"Relational SQL/joins"** → RDS/Aurora

Exam traps: "DocumentDB = fully managed, no replication/backup ops" → **true, Aurora-backed**; "DocumentDB is 100% MongoDB" → **compatible with Mongo 4/5/6 wire protocol & a subset of features, not everything**; "DocumentDB scales writes horizontally natively" → **no, single primary writer; app-level sharding**; "DocumentDB for key-value millions-rps" → **no, DynamoDB**.