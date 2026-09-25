# Kinesis Course  Data Streams & Firehose

## 1. Purpose

Kinesis is AWS's **real-time streaming data platform**. **Kinesis Data Streams (KDS)** ingests massive volumes of ordered, replayable records (clickstreams, logs, transactions, IoT) and makes them available to multiple consumer applications within milliseconds. **Amazon Data Firehose** (formerly Kinesis Data Firehose) is the fully managed way to **load streaming data straight into S3/Redshift/OpenSearch/HTTP** with batching, transformation, and no code. KDS = durable stream you read Firehose = managed delivery you don't babysit.

## 2. How it works

**Kinesis Data Streams:**
- Data is split across **shards**  the throughput unit:
  - **1 shard = 1 MB/s in, 2 MB/s out, 1,000 PUT records/s**
- Producers `PutRecord`/`PutRecords` (or KPL) each record gets a **sequence number** + **partition key** that routes it to a shard (same key → same shard → **ordered per shard**)
- Consumers **poll** using **shard iterators** (`GetRecords`)  one shard-iterator chain per shard iterators **expire in 5 minutes**
- Records are retained **24 hours by default (up to 365 days)** → **replay** anytime
- **Capacity modes**: **provisioned** (you pick shards, reshard via split/merge) or **on-demand** (auto-scales, pay per usage "On-demand Advantage" for spiky traffic)
- Durable: **synchronously replicated across 3 AZs**
- Consumers: KCL apps, Lambda (event source mapping), Firehose, Analytics

**Amazon Data Firehose:**
- No shards  fully managed **delivery stream** scales automatically
- Optional **Lambda transformation**, **Parquet/ORC conversion**, **dynamic partitioning** to S3
- Buffers (minutes) then delivers to **S3, Redshift, OpenSearch, HTTP endpoints, third-party**
- Records stored up to **24 hours** in delivery stream buffer near-real-time (default buffer interval ~5 min, can be lower)

```
Producers ──PutRecord──▶ Kinesis Data Stream (shards, 3-AZ, 24h–365d retention)
                              ├─ Lambda / KCL apps (read, replay, multiple consumer groups)
                              ├─ Kinesis Analytics (SQL on stream)
                              └─ Firehose delivery stream ──▶ S3 / Redshift / OpenSearch / HTTP
Producers ──────────────▶ Firehose directly ──(Lambda transform, batch)──▶ destinations
```

## 3. When to use

- **Real-time ingestion at scale**  clickstreams, app logs, IoT telemetry, financial transactions, social feeds
- **Multiple independent consumers** reading the **same stream** (replay, reprocess, different analytics)
- **Ordering per key** (e.g., per user/device) with **partition keys**
- **Replay / reprocessing**  retain 24h–365d, read from any point (audit, backfill, bug-fix rerun)
- **Feeding data lakes**  stream → Firehose → S3 (with Parquet + partitioning) for cheap analytics
- **Near-real-time dashboards/alerts**  Kinesis Analytics / OpenSearch / Lambda
- **Big migration/ETL backlog** → Firehose handles batching, compression, encryption without code
- **Chat/activity feeds, live metrics, anomaly detection**

## 4. When NOT to use

- **Simple decoupled work queue (pull, DLQ, long retention)** → **SQS**
- **One-off notifications / fan-out to users** → **SNS/EventBridge**
- **Request/response or synchronous APIs** → API Gateway
- **Batch ETL on a schedule** → **Glue / DataSync**, not a live stream
- **Very high per-record cost sensitivity at low volume** → SQS/SNS cheaper Kinesis is for streaming scale
- **Need global ordering across all shards** → only **per-shard / per-key** order exists no total order
- **Consumers must be push-only with no polling** → SNS/EventBridge (Kinesis consumers poll)
- **Kafka-compatible ecosystem with existing tooling** → consider **MSK (Managed Kafka)** instead
- **You only want to sink data to S3/Redshift with zero code** → **Firehose** (skip managing shards)

## 5. Important features

- **Shards**  1 MB/s in / 2 MB/s out / 1,000 writes per shard **split/merge** to reshard default limit ~500 shards (raisable)
- **Partition key**  determines shard placement same key = same shard = ordered hot keys cause hot shards
- **Capacity modes**  **provisioned** (scale shards, predictable) vs **on-demand** (auto, no shard math, good for unknown/spiky)
- **Retention**  24 hours default, configurable **up to 365 days** (enables replay / reprocessing)
- **Replay**  read from any sequence number/timestamp via shard iterators **iterators expire in 5 min**
- **Multi-AZ durability**  synchronous replication across 3 AZs encryption at rest (KMS) + in transit VPC endpoints
- **Consumers**  KCL (checkpointed, horizontal scaling), **Lambda event source mapping** (auto-scales per shard), Firehose, Analytics SQL, third-party apps **multiple consumer groups** each see full stream
- **PutRecords batching**  batch calls for throughput large records up to 10 MiB but avg must stay ≤1 MiB/s per shard
- **On-record size**  max 1 MB per record (default), avg throughput per shard 1 MB/s
- **Firehose features**  no shards, auto-scale, **Lambda transform**, **Parquet/ORC**, **dynamic partitioning**, **buffer + compress + encrypt**, delivery to S3/Redshift/OpenSearch/HTTP/3rd-party, **24h buffer/backup** on failure
- **Monitoring**  `IncomingBytes`, `GetRecords.IteratorAgeMilliseconds` (consumer lag  key alarm), `ReadProvisionedThroughputExceeded`
- **Security**  SSE-KMS, TLS, bucket/stream policies, PrivateLink

## 6. Limitations

- **Ordering only within a shard**  no global order across shards partition-key hot spots throttle a shard
- **Shard throughput caps**  a single hot partition key can't exceed one shard's 1 MB/s / 1,000 writes
- **Consumers poll**  not push KCL/Lambda must keep up or iterator age climbs
- **Provisioned mode = capacity planning**  too few shards → `ProvisionedThroughputExceeded` too many → waste (use on-demand to avoid)
- **Iterators expire in 5 minutes**  must re-fetch promptly long gaps lose position
- **At-least-once semantics**  duplicates possible consumers must be **idempotent**
- **Firehose is not a stream store**  no replay/read by multiple consumers like KDS it's a delivery pipe (buffer ≤24h)
- **Firehose latency**  seconds-to-minutes (buffer interval), not millisecond use KDS for sub-second
- **Firehose destinations limited**  S3/Redshift/OpenSearch/HTTP/3rd-party not arbitrary AWS APIs
- **Cost at high shard counts**  provisioned shards billed hourly even if idle
- **No cross-region replication built in**  build it or use multi-region design
- **Record/order limits**  1 MB max record (large-record mode avg-limited), hot-key skew

## 7. Trade-offs

- **Kinesis Data Streams vs Firehose**  durable replayable multi-consumer stream (you manage shards/consumers) vs zero-managed delivery sink (no replay, no multi-consumer, simpler/cheaper for pure landing)
- **Kinesis vs SQS**  ordered replayable multi-consumer stream vs durable work queue (single consumer per message, DLQ, no replay log)
- **Kinesis vs EventBridge/SNS**  streaming analytics + replay vs event routing/notification
- **Kinesis vs MSK (Kafka)**  fully managed AWS-native (shards, KCL, Lambda) vs self-managed Kafka ecosystem/tools/compatibility
- **Provisioned vs on-demand capacity**  predictable cost + control vs auto-scale for spiky/unknown (on-demand Advantage for bursts)
- **24h vs 365d retention**  cheap short retention vs long replay window at higher storage cost
- **Direct KDS→app vs KDS→Firehose→S3**  low-latency processing vs cheap batched data-lake landing
- **Partition-key design**  balanced parallelism vs hot-key throttling (salt/hash keys if skewed)

## 8. Architecture

Reference streaming patterns:

```
Real-time fan-out + lake:
  App logs/clickstream → Kinesis stream (on-demand) ├─ Lambda (real-time alerts → SNS)
                                                    ├─ KCL app (fraud rules)
                                                    └─ Firehose → S3 (Parquet, partitioned) → Athena

Replay/backfill:
  stream (30d retention) → reprocess from sequence # after bugfix → Redshift/OpenSearch

Hot-key safe ingest:
  producers hash/shard user_id across N partition keys → stream → per-key ordered Lambda consumers

IoT telemetry:
  devices → Kinesis (per-device key) → Firehose → S3 time-series prefix → QuickSight/OpenSearch dashboards
```

## 9. SAA-C03 Perspective

Kinesis is the **streaming / real-time / big-data ingestion** answer (Domains 1, 3, 4):

- **"Real-time streaming data / clickstream / IoT / log analytics at scale"** → **Kinesis Data Streams**
- **"Multiple independent consumers need the same data / replay / reprocess"** → **Kinesis** (retention + consumer groups)  SQS can't do this
- **"Load streaming data into S3/Redshift/OpenSearch with no code, batch/compress/partition"** → **Firehose**
- **"Ordered processing per customer/device"** → **partition key** (order per shard, not global)
- **Shard math**: N MB/s in ÷ 1 MB = shards writes ÷ 1,000 = shards **2 MB/s out per shard** for consumers
- **`IteratorAgeMilliseconds` high** → consumers falling behind (scale consumers/shards)
- **At-least-once** → idempotent consumers hot-key skew → fix partition keys
- **Firehose vs Kinesis**: replay/multi-consumer → **KDS** just land in S3/Redshift fast → **Firehose**
- **vs SQS**: queue for tasks vs ordered replayable stream for analytics

Exam traps: "near-real-time (not batch) into S3 without writing consumer code" → **Firehose** "must reprocess last hour of events after a bug" → **Kinesis with retention + replay** "per-shard order only, no global order" "iterator expires in 5 minutes" "hot partition key throttles despite spare shards".