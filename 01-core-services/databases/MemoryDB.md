# Amazon MemoryDB Course  Redis-Compatible, Durable In-Memory Database

## 1. Purpose

Amazon MemoryDB for Redis is a **Redis-compatible, in-memory database with durability** for **ultra-low latency (< 1 ms reads) and high throughput**  it is a **database, not just a cache**. Unlike ElastiCache (a cache backed by another data store), MemoryDB has a **durable, S3-backed copy** of data so you can use it as a **system of record** while still getting sub-millisecond latency. For SAA it's the answer to **"Redis data structures + durability / microsecond-ms in-memory database / no cache-rewarming on restart"**.

## 2. How it works

- **Cluster of nodes** (shards with replicas)  data replicated across AZs read replicas scale reads, primary handles writes
- **Durability**  every write is durably stored in the **MemoryDB distributed storage layer** (backed by logs in S3/SDB) before acknowledgment, so a node/instance failure **doesn't lose data and doesn't require cache rewarming** (in-memory DB)
- **Redis-compatible API**  strings, hashes, lists, sets, sorted sets, streams, pub/sub, Lua scripts, transactions
- **Scale**  add shards/read replicas scale up instance class snapshots (RDB) to S3 for backup/restore
- **Failover**  multi-AZ with automatic failover to a replica (up to 2 replicas per shard)
- **Security**  VPC-only, TLS in transit, KMS at rest, Redis AUTH password + IAM auth, Redis ACLs
- Cluster mode similar to Redis Cluster (partition by slot, multi-key ops restricted to same slot)

```
App (Redis client  RedisAPI) → MemoryDB cluster
   ├─ primary node(s) (writes) + replicas (reads, multi-AZ)
   └─ every write → durable distributed storage (S3-backed) BEFORE ack → survives restarts
Snapshots to S3 sub-ms read / low-ms write auto-failover across AZs
```

## 3. When to use

- **Redis data structures (leaderboards, sessions, queues/streams, caches, pub/sub) that must SURVIVE a restart**  durability of a database, latency of memory
- **Real-time, low-latency apps**  gaming leaderboards, chat, streaming counters, personalization, feature stores (ML)
- **Session/state store where losing the data after failover is unacceptable**
- **Replacing a "cache + DB" pair** (ElastiCache → RDS/DynamoDB) with a single durable in-memory store
- **High-throughput, low-latency workloads** that redis-ded apps already have

## 4. When NOT to use

- **Pure caching** of DB/other data where eviction is fine and cost matters → **ElastiCache** (cheaper, no durable copy) or DynamoDB Accelerator (DAX)
- **Relational/SQL/ACID-joins** → RDS/Aurora
- **Key-value at petabyte/serverless scale** → **DynamoDB**
- **Analytics / document rich-queries** → Redshift / DocumentDB
- **Redis as a cache only, tolerate losing everything on restart** → ElastiCache (less cost)

## 5. Important features

- **Redis-compatible API**  drop-in for Redis client apps (strings, hashes, lists, sets, sorted sets, streams, pub/sub)
- **Durability + sub-ms latency**  durable S3-backed storage layer (unlike ElastiCache) no rewarm after failover
- **Multi-AZ automatic failover**, up to 2 replicas/shard, read scaling
- **Data tiering** option  move rarely-used keys to SSD to cut cost (like ElastiCache data tiering)
- **Snapshots (RDB) to S3**, point-in-time restore scaling up/down anytime
- **Enterprise security**  VPC-only, TLS, KMS, Redis AUTH/ACLs, IAM auth, AWS PrivateLink
- **CloudWatch metrics**, engine upgrade in-place (to current Redis OSS versions available)

## 6. Limitations

- **It's in-memory  cost is premium** vs disk-backed databases capacity planning on memory
- **RedisAPI compatibility, not 100% Redis**  some Redis modules/extensions and certain server-side capabilities are not supported (check per-version compatibility)
- **Cluster mode**  key design (slots) applies some commands restricted in clustered mode
- **VPC-only** access (needs VPC endpoints/peering  normally fine)
- **Provisioned clusters** (not serverless)  you manage node sizing/scaling actions

## 7. Trade-offs

- **MemoryDB vs ElastiCache (Redis)**  durable in-memory DATABASE (survives restart, system of record) vs cache (ellipsis, faster-to-evict, cheaper, pairs with a backing datastore). Rewarm vs durability is THE distinguishing decision
- **MemoryDB vs DynamoDB**  Redis structures/sub-ms for hot data vs key-value at massive scale/serverless/global tables (use MemoryDB when Redis structures + durability are required)
- **vs self-managed Redis on EC2**  fully managed HA/durability vs control/custom modules (still limited to supported Redis feature set)

## 8. Architecture

```
Gaming real-time stack:
  Game servers → MemoryDB (leaderboards via sorted sets, live counters as hashes,
  match state in streams)  sub-ms reads, durable writes to S3-backed layer
  ├─ replicas scale reads multi-AZ failover keeps availability
  └─ snapshots → S3 for analytics/bumps data tiering for cold keys
No separate cache layer needed  MemoryDB is the store itself
```

## 9. SAA-C03 Perspective

- **"Redis-compatible IN-MEMORY DATABASE with DURABILITY (survives crash) / microsecond-to-ms"** → **MemoryDB**
- **"Redis cache only, eviction/warm-up acceptable, backing DB exists"** → **ElastiCache**, not MemoryDB
- **"Purely microsecond cache layer in front of DynamoDB"** → **DAX**
- **"Leaderboards/sessions/streams that must not be lost"** → MemoryDB
- **"Key-value serverless at scale"** → DynamoDB

Exam traps: the #1 distinguishing question  **durability and "no source-of-truth loss on restart" = MemoryDB caching/epoch-temporary = ElastiCache** "MemoryDB is a cache" → **no, durable database** "Redis-compatible = every Redis module" → **not exactly, check support** "serverless" → **provisioned clusters (not serverless)**.