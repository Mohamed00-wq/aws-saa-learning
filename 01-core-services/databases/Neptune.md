# Amazon Neptune Course — Fully Managed Graph Database

## 1. Purpose

Amazon Neptune is a **fully managed graph database** for **highly connected data** — you query relationships (edges) between entities (vertices) in milliseconds, at billions of relationships. It supports the three major graph query languages: **Gremlin** (Apache TinkerPop) and **openCypher** for property graphs, and **SPARQL** for RDF (linked-data) graphs. For SAA it's the answer to **"graph database / connected data / relationships / traversals / fraud rings, social, knowledge graphs"**.

## 2. How it works

- **Clusters** — a **primary instance (writes)** + up to **15 fast-failover read replicas**; **storage auto-scales up to ~128 TiB** (no pre-provisioning)
- **In-memory optimized engine** — Neptune's graph engine uses a scale-up architecture tuned for fast traversals (100k+ queries/sec on large graphs)
- **Data models** — **Property Graph** (RDF-based quad store; vertices/edges with properties) queried via **Gremlin** (traversal) or **openCypher** (SQL-like), and **RDF** triples via **SPARQL**
- **Bulk loader** from S3 (CSV/Labels/OpenCSV/JSON) for fast graph loading
- **Neptune Streams** — CDC (property graph + SPARQL ranges) → Lambda/Kinesis downstream (audit, search index)
- **Neptune Serverless** option — scale instantly (pay for used capacity); Graph notebook (Jupyter), Data API, **Neptune ML** (GraphSAGE) for link prediction/recommendations, **Neptune Analytics** engine for graph analytics over tens of billions of relations
- Backups: automatic, incremental, continuous (S3-backed, 99.999999999% durability)

```
App (Gremlin / openCypher / SPARQL) → Neptune cluster
  ├─ primary (writes) + up to 15 read replicas (multi-AZ failover)
  └─ auto-scales to ~128 TiB; bulk loader from S3; Streams → Lambda
Serverless option; Neptune ML for predictions; Neptune Analytics for analysis
```

## 3. When to use

- **Deep, multi-hop relationships** — social networks (friends of friends), fraud detection (ring detection, referral fraud)
- **Knowledge graphs / semantic webs** (RDF, SPARQL), graph-based recommendations ("customers also bought / similar")
- **Network & IT topology**, supply chain, logistics routing, IoT device relationships, identity graphs (identity resolution)
- **Millisecond traversals over billions of nodes/edges** for interactive graph apps
- **Fraud & 360-degree views** — enrich with graph connections

## 4. When NOT to use

- **Simple key-value / CRUD access patterns** → **DynamoDB**
- **Relational OLTP with SQL/joins/transactions** → **RDS/Aurora**
- **Shallow data (few relationships) / mostly aggregates** → a regular database is simpler
- **Document workloads / rich aggregation pipelines** → DocumentDB / OpenSearch
- **Text search / log analytics**, tabular BI → OpenSearch / Redshift / Athena
- **Teams not ready for graph query languages** — a relational model may be easier

## 5. Important features

- **Three query languages** — Gremlin + openCypher (property graph) and SPARQL (RDF); query the same property graph with both Gremlin & openCypher
- **Scale-up, in-memory optimized engine** — fast traversals on large graphs, **~128 TiB** auto-scaling storage
- **Up to 15 read replicas + Multi-AZ failover**; automatic incremental backups (S3 durability)
- **Neptune Streams** (change data capture) for real-time downstream reaction
- **Neptune Serverless** (instant scale) and **Neptune Analytics** (graph analytics engine, billions of relationships in seconds)
- **Neptune ML** (GraphSAGE) — link prediction, recommendation, fraud-prediction models built on the graph
- **Bulk loader, graph notebooks (Jupyter), Data API**, VPC-only security, KMS encryption, audit logs
- **Outposts support** for on-premises graph workloads

## 6. Limitations

- **Single-writer clusters** — scale writes via bigger instances (no native horizontal write scaling)
- **Graph query languages have a learning curve** — Gremlin/SPARQL/openCypher are not SQL
- **Not for general-purpose OLTP/CRUD** — it's specialized for connected-data traversal
- Storage/compute scale-up model — very large graphs need careful instance sizing (that's what Neptune Analytics is for)
- **VPC-only** access model (PrivateLink for cross-VPC)

## 7. Trade-offs

- **Neptune vs DynamoDB** — traversing N-hop relationships/connected data (Neptune) vs key-value point lookups at unlimited scale (DynamoDB)
- **Neptune vs RDS/Aurora** — graph traversal queries vs classic SQL joins (deep joins on RDS become painful; graph DB models them natively)
- **Neptune vs self-managed Neo4j/Apache TinkerPop** — fully managed HA/backups/Serverless/ML vs open-source control
- **Neptune Database vs Neptune Analytics** — interactive online traversals (DB) vs offline analytics over massive graphs (Analytics)

## 8. Architecture

```
Fraud detection:
  Events → Lambda → Neptune (vertices: users/devices/orders; edges: transactions, shares, referrals)
  Realtime check: Gremlin traversal (is this device in a known fraud ring?) → deny/allow
  ├─ Neptune Streams → OpenSearch (secondary index) + audit
  └─ Neptune ML — predict collusion risk on new connections; Prometheus analytics on history
Replica read scaling; Serverless for variable queries
```

## 9. SAA-C03 Perspective

- **"Graph database / connected data / relationships"** → **Neptune**
- **"Fraud/social networks/recommendations/knowledge graphs, billions of relationships in ms"** → **Neptune**
- **"Gremlin / SPARQL / openCypher, RDF/property graph"** → Neptune languages
- **"Key-value CRUD, serverless scale"** → DynamoDB; **"SQL joins/OLTP"** → RDS/Aurora
- **"Analyze billions of graph relationships for insights"** → **Neptune Analytics**

Exam traps: "Neptune is for relational OLTP" → **no, graph traversals**; "Neptune for key-value reads" → **no, DynamoDB**; "Neptune replaces OpenSearch for search" → **no, connected-data queries vs full-text search**; "single-round queries anywhere" → **designed for traversal workloads on connected data**.