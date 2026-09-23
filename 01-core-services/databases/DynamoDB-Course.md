# DynamoDB Course — Serverless NoSQL Key-Value & Document Database

## 1. Purpose

DynamoDB is AWS's **fully managed, serverless NoSQL database** for **key-value and document** data with **single-digit-millisecond latency at any scale** — petabytes of data, tens of millions of requests/sec. It auto-scales, is multi-AZ durable by default, has **global tables** for multi-Region active-active, and integrates natively with Lambda/serverless. For SAA it's the answer to **"key-value / single-digit-ms / serverless / scale to millions of requests"**.

## 2. How it works

- **Tables**: schema-less items identified by a **primary key** — **partition key** (hash) + optional **sort key** (range)
- **Indexes**: **Local (LSI)** — alternate sort key, same partition key, fixed at creation; **Global (GSI)** — different partition/sort keys, added anytime, each with own capacity
- **Capacity modes** per table:
  - **On-demand (recommended, default)** — pay per request; instantly scales up to **2x previous peak** ("warm throughput" avoids cold starts); no capacity planning
  - **Provisioned** — set **RCU/WCU**, auto-scale via **Auto Scaling** policy; cheaper when predictable; can add per-table max caps in on-demand
- **Units**: **1 RCU** = 1 strongly-consistent read of **4 KB** (2 x eventually-consistent) — **1 WCU** = 1 write of **1 KB**; items bigger → more units; transactions = 2x units (read/write)
- **Consistency**: default **eventually consistent** reads (cheaper); optional **strongly consistent** (one RCU for 4 KB)
- **Streams**: capture item **INSERT/MODIFY/DELETE** by key-change → Lambda/Firehose/other; **Kinesis Data Streams for DynamoDB** for item-level streaming
- **DAX** — fully managed **in-memory accelerator** (write-through): single-digit-µs reads, up to ~~10x faster~~ (from ms to µs), API-compatible, no app changes (just DAX client SDK)
- **Global tables** — multi-Region **active-active** (each region writable) with **last-writer-wins conflict resolution**, 99.999% availability; strong consistency option cross-Region
- **TTL** — automatic expiry/delete of expired items (cost saving, time-series cleanup)
- **Transactions**, **conditionals**, **atomic counters**, **PartiQL**, **JSON documents**, **Point-in-time recovery (PITR)** — restore to any second in last **35 days**
- **Adaptive capacity** — hot partitions auto-split; burst on-demand; no hot-key management (mostly)

```
App/lambda → DynamoDB (data replicated across 3 AZs) ← DAX cache (µs reads)
         ├─ Streams (CDC) / Kinesis for DynamoDB → Lambda/firehose/analytics
         ├─ GSI/LSI indexes; TTL expiry; PITR backups
         └─ Global Tables (multi-Region active-active, LWW conflicts)
Capacity: on-demand (pay per request) or provisioned (RCU/WCU + autoscaling)
```

## 3. When to use

- **Key-value / single-item / flexible schema** data: user profiles, sessions, metadata, IoT/sensor state, carts, high-score tables
- **Serverless-first applications** — natural fit with Lambda, API Gateway, EventBridge
- **Predictable/required low latency at ANY throughput** — millions of rps, no ops
- **Clickstreams, gaming, ad-tech, live leaderboards** — write-heaviest patterns
- **Time-series-like with TTL cleanup** — IoT, telemetry, search history
- **Multi-Region availability** — **global tables** for active-active local writes
- **Caching** — DAX for µs reads on read-heavy hot-data
- **Transactions across multiple items**, atomic counters, conditional writes
- **Metadata / session store** where a relational schema is overkill

## 4. When NOT to use

- **Complex joins / arbitrary ad-hoc SQL queries across tables** → **RDS/Aurora**
- **Relational ACID with joins**, second-order normalization → RDS/Aurora
- **Analytics / DW / columnar scans at petabyte scale** → Redshift (DynamoDB is not for big analytical scans)
- **Document-only DB with rich aggregation/query/pipelines** → **DocumentDB/MongoDB**
- **Text search / full-text / flexible queries over large corpus** → OpenSearch
- **Graph traversals** → Neptune
- **Very large objects / blobs > 400 KB limit** → S3 (store refs in DynamoDB)
- **Require global strong consistency for relational semantics** → Aurora Global DB / RDS
- **In-memory store fronting DynamoDB** — DAX if read-heavy; ElastiCache if Redis semantics outside DynamoDB
- **One-off SQL BI reporting tooling** (ad-hoc analyst SQL) — DynamoDB has no general SQL engine (PartiQL limited)

## 5. Important features

- **Primary keys** — simple (partition) or composite (partition + sort); design via **GSI/LSI** for access patterns
- **GSI/LSI** — LSIs (same partition key, alternate sort key, fixed at create) vs GSIs (own capacity, add anytime; useful for cross-partition lookups)
- **Consistency** — default eventual; strongly-consistent reads when needed
- **RCU/WCU** math — the classic exam formula: 1 KCU read = 4 KB strong / 8 KB eventual; write 1 KB
- **Capacity modes** — **on-demand (default, recommended)** vs **provisioned + autoscaling**; on-demand supports **max throughput caps** per table/GSI; **warm throughput** = always-instant capacity
- **DAX** — µs reads, 10x read perf, write-through cache, handles millions rps, 3-10 nodes cluster, VPC-only, no AppServer changes
- **Streams vs Kinesis** — DynamoDB Streams (ordered per item, 24h) for Lambda trigger; **Kinesis Data Streams for DynamoDB** (item-level, up to 24h+ retention, for consumer-heavy/analytics)
- **Global tables** — multi-Region active-active, LWW conflict resolution, 99.999% availability, strong consistency option
- **TTL** — auto-delete expired items (saves cost, patterns: sessions, history)
- **Backups** — **PITR** (≤ 35 days, restore to any second) + **on-demand backups** (long-term, retained indefinitely)
- **Transactions / conditional writes / atomic counters** — ACID within and across items (2x RCU/WCU units)
- **Security** — IAM, KMS at rest, TLS, VPC Endpoints/PrivateLink, DAX IAM/SG
- **Adaptive capacity + burst capacity** — deals with hot partitions; on-demand handles spikes instantly
- **Table classes** — Standard vs **Standard-IA** (infrequent-access data cheaper)
- **Zero-capacity for many** — pay only consumed units

## 6. Limitations

- **Item max size 400 KB** — store large payloads in S3, pointer in DynamoDB
- **No joins / limited query semantics** — data modeling must anticipate access patterns (key-design heavy lifting)
- **RCU/WCU per-partition limits** — extremely hot single key can still throttle (though adaptive/on-demand mitigate); choose a good partition key
- **PITR max 35 days** — longer = on-demand backups or export to S3
- **Streams 24-hour retention** (DynamoDB streams) — use Kinesis form for longer/managed
- **Transactions 2x cost** — trade-off for atomicity
- **Eventual consistency default** — strong consistency costs more RCUs; global tables use LWW (not true causal)
- **GSI comes eventually consistent, few key-design hard rules baked in; adds cost (own capacity)** — design once, hard to change
- **LSI fixed at table creation** (can't change partition-key set of LSI)
- **DAX** — sub-node memory limits, VPC-only, no cross-Region natively, extra cost; strong consistency within DAX limited (read-through/cache semantics)
- **PartiQL is limited SQL** — not a general-purpose analytical SQL engine
- **Not for complex/OLTP ad-hoc relational reporting** — no flexible OLAP queries
- **Capacity spikes > 2x peak** in on-demand not instantly (doubling pattern), but warm throughput + peaking helps; spike beyond → some throttling until scaling catches up

## 7. Trade-offs

- **DynamoDB vs RDS/Aurora** — schema-flexible, key-base access, scale-out global, RN is single-digit-ms vs relational joins/migrations/features/BI — pick by data shape and query needs
- **On-demand vs provisioned** — truly variable/spiky/unpredictable (pay-per-request, no planning) vs predictable steady load (pre-provision, autoscaling, cheaper unit-rate)
- **DAX vs ElastiCache** — DynamoDB-native in-memory µs front-end (no app rewrite, write-through) vs general Redis/Memcached cache (any data, flexible expiry/structures) 
- **DynamoDB Streams vs Kinesis for DynamoDB** — simple ordered-stable-per-item trigger (24h, 1 consumer math) vs longer/multi-consumer/analytics ingestion
- **Global Tables vs single-region** — multi-Region active-active (writes anywhere, local latency, DR) vs simpler + no cross-Region conflict semantics
- **GSI vs LSI** — flexible global lookup on new keys/own capacity vs cheaper-ish fixed alternate sort-key on same partition
- **Standard vs Standard-IA table class** — rarely-accessed datasets cheaper vs hot key-value active sets
- **Transactions vs plain ops** — atomicity across items vs 2x cost & complexity
- **strong vs eventual reads** — always-current view vs cheaper 2x-eff. reads (pick per workflow)
- **PITR vs on-demand backups** — continuous 35-day any-point restore vs ad-hoc durable snapshot, no restore-window limit

## 8. Architecture

Reference patterns:

```
Serverless e-commerce:
  API GW → Lambda → DynamoDB (on-demand) + DAX cluster for hot product/user reads
  ├─ Streams → Lambda (order events → SQS → services)
  └─ TTL on carts/sessions (auto-cleanup); PITR + on-demand backups

Global gaming:
  DynamoDB Global Tables (US-EU) — players write locally; active-active LWW; 99.999% SLA

IoT/telemetry:
  Devices → Kinesis → Lambda → DynamoDB (time-series, TTL 30 days) → dashboards via Athena/OpenSearch from stream

Read-heavy live feed:
  DynamoDB (source of truth) ← write-through → DAX: ms→µs reads at millions rps behind ALB/Lambda
```

## 9. SAA-C03 Perspective

DynamoDB is a **core Domain 1/3/4** choice for **scalability, performance, serverless**:

- **"Key-value / single-digit-ms / unlimited scale / serverless database"** → **DynamoDB**
- **"Unpredictable/spiky traffic without capacity planning"** → **on-demand capacity**
- **"µs reads" / "accelerate read-heavy workload"** → **DAX**
- **"React to item changes (CDC) serverless"** → **Streams → Lambda**; "multi-consumer/long retention streaming" → **Kinesis for DynamoDB**
- **"Multi-Region active-active with local writes"** → **Global Tables** (LWW conflict; 99.999%)
- **"Cheap expiry of sessions/history"** → **TTL**
- **"Restore accidental career"**: PITR ≤ 35 days / on-demand backups
- **RCU/WCU math** — likely compute: (reads × size/4KB) × strongest; writes ×(size/1KB)
- **Consistency**: eventually vs strongly; **adaptive capacity** for hot keys
- **Item 400 KB cap** → S3-reference pattern
- **vs RDS/Aurora** — relational = RDS/Aurora; key-value/serverless/global = DynamoDB

Exam traps: "DynamoDB does joins" → **no, key-based; model with GSIs**; "on-demand is always cheapest" → **provisioned + autoscaling better at predictable volume**; "global tables = strong consistency all regions" → **LWW eventual by default (option for strong)**; "LSI can be added later" → **no, at creation**; "streams retain 7 days" → **24 h**, Kinesis form for longer; "transactions cost normal units" → **2x**; "DAX beyond regions / VPC" → VPC-only.