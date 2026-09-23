# ElastiCache Course — In-Memory Caching & Redis/Valkey/Memcached

## 1. Purpose

ElastiCache is AWS's **managed in-memory caching / in-memory data store** built on **Redis OSS / Valkey** (rich datatypes, persistence, replication) and **Memcached** (simple distributed object cache). It sits **in front of databases** or serves **standalone sub-millisecond reads** — cutting backend DB load (and DB RU/cost), reducing latency, and speeding up hot-data access for web apps, sessions, leaderboards, rate limiting, pub/sub. A core "performance/cost" tool on AWS.

## 2. How it works

- **Cluster** of **cache nodes** (EC2-backed) running the engine; **node type** defines RAM/CPU; **VPC placement + SG** for access
- **Redis/Valkey modes**:
  - **Cluster Mode Disabled (CMD)** — single shard, 1 primary + **up to 5 read replicas** (same shard); simpler, modifies easily
  - **Cluster Mode Enabled (CME)** — **1–500 shards**, hash-slots divided; **1–5 replicas per shard** → horizontal scaling, online resharding, partitions data
- **Replication** — primary → replica(s); **Multi-AZ auto-failover**; replicas serve reads; **Read Replicas** for read scaling
- **Read-through** in app: check cache (hit → return; miss → fetch DB → store)
- **Data tiering** (r6gd node type): LRU-move cold items from RAM → **local SSD**; up to **1 PB @ 500 nodes**; ~+300 µs on SSD hits; good when only ~20% of dataset is hot
- **Serverless cache (Valkey/Redis)** — engine-managed, autoscale, no node management (new ElastiCache offering)
- **TTL/eviction policies** — `volatile-lru`, `allkeys-lru`, `volatile-lfu`, `allkeys-lfu`, `noeviction`…
- **Persistence** (Redis/Valkey) — RDB snapshots + AOF append-only logs (best-effort) — cache mainly in-memory, restore from RDB on cold
- **Populate approaches** — **lazy-loading** (cache-aside; simple, but cache misses hit DB) vs **write-through** (always write to cache+DB; fresh, more writes)

```
App → Cache (Valkey/Redis CME shards ∥ replicas, or Memcached nodes) → on hit serve µs-ms (no DB)
   ├─ on miss → DB → populate cache (lazy) / DB → write-through updates
   ├─ Multi-AZ failover; data tiering r6gd SSD; TTL/eviction; optional persistence (RDB/AOF)
   ├─ DAX? (that's DynamoDB-specific); ElastiCache is generic Redis/Memcached
   └─ Serverless cache option when node mgmt is unwanted
```

## 3. When to use

- **Frequently read, rarely-changing data** — product catalogs, configs, lookups
- **Session/token store** (short TTL) — web sessions, user state
- **Databases load reduction** — reduce RDS/DynamoDB read units & latency (esp. read-heavy)
- **Leaderboards / counters / counting** — Redis sorted sets, atomic increments
- **Pub/sub / messaging / real-time presence** — Redis publish/subscribe (concise)
- **Rate limiting** — tokens, sliding window counters
- **Cache anything where sub-millisecond-to-a-few-ms beats DB and data tolerable to stale** 
- **Standalone in-memory data structure store** (Redis/Valkey rich types)

## 4. When NOT to use

- **Source of truth / transactional durable data** → use a real DB; cache is not the system of record
- **Key-value durable NoSQL with single-digit-ms** → **DynamoDB** (durable, serverless, no cache ops); use DAX if it's DynamoDB+µs
- **Relational app needing joins/PITR** → RDS/Aurora
- **Simple µs-reads for DynamoDB** → **DAX** (not ElastiCache) — different integration
- **Windows-only, AD-integrated, or heavy TLS/key-value global scale** — beware ElastiCache is a memory-tier not the "database" answer
- **Data that changes constantly and MUST be strongly consistent globally** — cache eventual; DB of record
- **Very large working set that never fits RAM** → data tiering or different store (cache works when hot ≤ RAM + SSD)
- **High-availability as a database promise** — ElastiCache is fast + replicated, but not a durable DB (losing node = losing data unless persistence/RDB restored & re-warmed)

## 5. Important features

- **Engines** — **Valkey (new AWS-preferred open-source fork)**, **Redis OSS**, **Memcached**
- **Cluster mode enabled** — sharding + replicas; online resharding, rebalancing; **huge horizontal scale** (500 shards max); CMD ≤ 5 replicas
- **Read replicas + Multi-AZ autofailover** — for Redis/Valkey
- **TTL & eviction policies** — flexible LRU/LFU; **noeviction** for strict
- **Data tiering** — r6gd family: RAM ↔ **SSD** LRU; store much more per node at lower $/GB; ~300 µs SSD latency; up to **1 PB/cluster (500 nodes)**; cannot scale to non-r6gd, no S3 export, RDB-restore only to r6gd
- **Serverless caches** (newer) — auto scales, no node admin, pay per-use (ECPU), cluster-enabled internally
- **Persistence** — RDB snapshots, AOF (Redis/Valkey optional)
- **Snapshots (RDB)** — backup/restore; export to S3 (non-r6gd); use for DR/re-warm
- **Security** — TLS, Redis AUTH / IAM auth (Redis), SG + VPC, **Redis Cluster** data-plane encryption (TLS), **OS-level** (ElastiCache managed)
- **Autoscaling** (Valkey/Redis 6.2+/7.2+) — target-tracking on memory/CPU over node count
- **Global Datastore** — cross-Region replication for Redis/Valkey (DR, local reads)
- **Monitoring** — CloudWatch (CPU, memory, evictions, cache hits/misses, network), metrics like `CurrConnections`, `CacheHits`/`CacheMisses`, `BytesUsedForCache`
- **ElastiCache Service IAM/SGs**; **ElastiCache CMD-friendly** app connection endpoints (configuration endpoint for CME)

## 6. Limitations

- **Cache ≠ durable** — data can be lost (after failover without persistence/on eviction/Spot); **design for cache loss** — re-warm from DB
- **Memory-bound per node** — if dataset exceeds RAM (without tiering), heavy evictions or OOM; must re-warm
- **Memcached has no**: replication/failover (multi-node cluster, node loss = lost cached objects), documents/leaderboard structures, persistence, replication-based HA — you just re-populate
- **Multi-AZ** — on Redis/Valkey; Memcached has no HA mechanism built-in (design external + tolerate empty cache)
- **Cluster Mode Enabled resharding** done online but client must support cluster protocol (config endpoint + hash-slot aware) — some clients need upgrade
- **CMD → CME migration supported one-way (CMD→CME); no reverse**
- **TTL not a substitute for correctness** — stale-data issues; need cache-invalidation discipline
- **Cost** — node-based per-hour + memory; bigger-node cheaper-per-GB but over-provisioning; DAX/ElastiCache decisions are price-sensitive
- **Connection/session limits** — node client limits apply
- **Cross-AZ latency** — replicas across AZs add minor replication latency; reads from replicas are eventually-consistent
- **Backup/restore** — snapshotrestore OK (RDB); AOF rebuild slower; export to S3 for off-AWS copy is constrained on r6gd

## 7. Trade-offs

- **ElastiCache vs DAX** — generic Redis/Memcached (any backend, rich structures) vs **DynamoDB-only µs front-end** (no app code changes)
- **Valkey vs Redis OSS vs Memcached** — Redis/Valkey = rich types/persistence/replication/leaderboards; Memcached = ultra-simple distributed key-value cache, no HA
- **CMD vs CME** — easy admin/single-shard vs sharded horizontal scale + online resharding (choose by predicted size/read scale)
- **One primary + replicas vs many shards (CME)** — replication-only read scaling vs partition across many primaries (hot shards avoided)
- **Lazy loading vs write-through** — simplest, tolerate stale & cold miss cost vs always-fresh, extra writes, must handle failures carefully
- **RAM vs data tiering (r6gd)** — µs from RAM vs +300µs SSD but much bigger working set per $; ideal when ~20% hot
- **Serverless vs node-based** — no capacity/patch mgmt, per-use billing vs predictable hourly node costs + explicit sizing; serverless for variable/lightweight
- **TLS/IAM-AUTH vs simple AUTH** — enterprise security vs simplicity/perf overhead
- **Global Datastore vs single-region** — DR + local reads across regions vs simpler + no lag
- **Persist (RDB/AOF) vs pure cache** — recoverability vs write-cost/latency; caching usually pure-memory

## 8. Architecture

Reference patterns:

```
Database offload:
  ALB → EC2/ECS app → ElastiCache (Valkey CME, Multi-AZ, read replicas)
      → on cache miss → RDS read replica → populate (lazy); write-through on updates
      → metrics: CacheHits/CacheMisses, evictions alarm

Session store / rate limit:
  ALB sticky-less → ElastiCache (Redis) sessions TTL 30 min + Redis rate-limit counters per user/IP

Leaderboard/gaming:
  Redis sorted-sets (leaderboards), incr counters; data tiering r6gd if large; CME for many shards

Cross-region:
  ElastiCache Global Datastore primary (us-east-1) + secondary (eu-west-1) → local low-latency reads, DR
```

## 9. SAA-C03 Perspective

ElastiCache is a **performance & cost** lever (Domains 3, 4):

- **"Reduce DB load / speed up repeated reads"** → **ElastiCache in front of RDS/Aurora/DynamoDB** (or DAX for DynamoDB)
- **"Sub-millisecond / cache-aside / sessions / leaderboards"** → **ElastiCache for Redis/Valkey**
- **"Large dataset, only ~20% truly hot, want cheaper-than-RAM"** → **data tiering (r6gd)**
- **"Autoscale cache nodes / skip node management"** → **Serverless cache** or autoscaling
- **"Read replicas + Multi-AZ failover for cache"** → **Redis/Valkey replicated cluster**
- **Drawing on Redis structures for real-time (pub/sub, rate limit, counter, geospatial)** → ElastiCache for Redis/Valkey
- **Know the cache-algorithm pattern**: lazy/cache-aside (common) vs write-through; **alarm on CacheMisses↑ + evictions**
- **Memcached**: simple KV, no HA/persistence — answer expects you to pick Redis/Valkey for anything needing failover/types
- **vs DAX**: DynamoDB-specific, µs, no app rewrite → DAX; generic caching/replicated/rich-store → ElastiCache
- **vs DynamoDB**: cache is not durable; durability/strong-consistency = DB

Exam traps: "ElastiCache is durable, use as database" → **no — re-warm on loss**; "Memcached has Multi-AZ/failover" → **no**; "DAX = generic Redis" → **no, DynamoDB-focused**; "add Shards to CMD" → **need CME**; "r6gd restore into normal node" → **cannot (backup r6gd→r6gd only)**; "global tables' = strong cross-region forever" → LWW/eventual semantics apply; "always use CME for small cluster" → CMD fine & simpler.