# SNS Course — Simple Notification Service

## 1. Purpose

SNS is AWS's **fully managed pub/sub (publish–subscribe) messaging** service. A publisher sends a message to a **topic**; SNS **pushes it to all subscribers** simultaneously — SQS queues, Lambda, HTTPS webhooks, email, SMS, mobile push. It's the **fan-out** primitive: one event, many independent consumers, without the publisher knowing who's listening. Pair with SQS when you need durable queuing per consumer.

## 2. How it works

- A **publisher** sends a message to an **SNS topic**
- SNS evaluates **subscription filter policies** and **pushes** the message to each matching subscriber
- Subscribers choose protocol: **SQS, Lambda, Firehose, HTTPS, email, SMS, mobile push, Kinesis Data Firehose, HTTP**
- Two topic types:
  - **Standard** — **at-least-once**, **best-effort ordering**, near-unlimited throughput
  - **FIFO** — **strict ordering**, **deduplication** (5-min window), SQS subscriptions only, 3,000 msg/s per topic
- Delivery is **push-based** — no polling; SNS handles retries with a **backoff** and can route failures to an **SNS DLQ**
- Message size up to **256 KB** (Extended Client → S3 for larger, up to 2 GB)

```
Publisher → SNS topic → filter policy → push to each subscriber:
                                   ├─ SQS standard (durable work queue)
                                   ├─ SQS FIFO (ordered processing)
                                   ├─ Lambda (trigger)
                                   ├─ HTTPS webhook
                                   └─ email / SMS / mobile push (A2P)
              failures → retry w/ backoff → SNS DLQ
```

## 3. When to use

- **Fan-out** — one event → many consumers (indexing, billing, analytics, notifications)
- **A2P messaging** — SMS, email, mobile push to end users
- **Decoupled event notification** — S3/EC2/CloudWatch events → topic → subscribers
- **Cross-account publishing** — resource policy allows another account's topic/queue
- **Filtering** — subscribers only get matching messages (filter policies on message attributes)
- **Pair with SQS** — SNS pushes, SQS buffers per consumer (best-practice fan-out)
- **FIFO pub/sub** — strict order to ordered SQS FIFO queues (e.g., bank transaction logging)

## 4. When NOT to use

- **Point-to-point durable work queue** (single consumer, buffering, long retention) → **SQS alone**
- **Request/reply or synchronous APIs** → API Gateway / AppSync
- **Event filtering across many AWS+SaaS sources with schema registry** → **EventBridge**
- **Replayable streams / multiple consumer groups reading a retained log** → **Kinesis Data Streams**
- **FIFO topic → Lambda directly** — not supported; must subscribe an SQS FIFO/standard queue first, then trigger Lambda from the queue
- **FIFO → email/SMS/HTTP endpoints** — unsupported (ordering can't be guaranteed); attempts error out
- **Workflow orchestration** → Step Functions

## 5. Important features

- **Standard topic** — near-unlimited msg/s; best-effort ordering + at-least-once; up to **12.5M subscriptions** per topic (100k topics/account)
- **FIFO topic** — up to **3,000 msg/s or 10 MB/s per topic**; strict order + dedup; **1,000 FIFO topics/account, 100 subscriptions/topic**; **300 msg/s per message group** — use many groups for parallel throughput
- **Message filtering** — subscription **filter policies** (JSON) on message attributes; subscribers receive only matching messages (reduces downstream cost/processing)
- **Raw message delivery** — deliver the raw message body instead of the SNS envelope (great for SQS/HTTPS)
- **SNS DLQ** — failed deliveries after retries land in a DLQ for debugging
- **Fanout to SQS** — standard pattern: each consumer gets its own queue; consumers scale independently
- **Encryption** — server-side encryption on topics (SSE-SNS / KMS); access policies for cross-account
- **Message archiving & replay** — FIFO topics: in-place **ArchivePolicy** retention (1–365 days) + replay to new/existing subscriptions; Standard topics: archive via **Firehose → S3**
- **EventBridge/S3/CloudWatch** can publish to topics; **SNS Extended Client** for >256 KB payloads
- **Mobile push (A2P)** — APNs/FCM platform applications; **SMS** with origination IDs

## 6. Limitations

- **At-least-once + best-effort ordering on standard** — duplicates and reordering possible; consumers must be idempotent
- **FIFO limits** — 3,000 msg/s/topic, 300 msg/s/group, 100 subs/topic; no email/SMS/HTTP/FIFO-Lambda direct
- **256 KB message cap** — larger needs S3 references (Extended Client)
- **Push, not pull** — no native buffering; a slow consumer needs **SQS behind the topic**
- **FIFO topics can't fan out to Lambda directly** — extra hop through SQS
- **No cross-region delivery** — topics are regional (use cross-region patterns manually)
- **Retries are finite** — after backoff fails, message → DLQ or dropped (configure DLQ!)
- **Ordering only within a message group**
- **SMS/push costs** are per-message and can spike unexpectedly (set spend limits)

## 7. Trade-offs

- **SNS vs SQS** — push/fan-out (many subscribers, no buffering) vs pull/queue (durable, one consumer, buffering); **SNS+SQS** is the standard combo for durable fan-out
- **SNS vs EventBridge** — simple pub/sub to fixed subscribers vs event bus with **filtering, schema registry, SaaS sources, cross-account buses, Pipes**
- **SNS vs Kinesis** — ephemeral push notification vs durable replayable stream for analytics
- **Standard vs FIFO topic** — throughput vs strict order + dedup
- **Direct Lambda subscription vs SQS-backed** — simpler vs durable buffering + retries + concurrency control (SQS-backed preferred for reliability)
- **Filter policies vs fan-out-everything** — less downstream load/cost vs simpler publisher
- **Raw message delivery on/off** — native consumer format vs SNS envelope metadata

## 8. Architecture

Reference patterns:

```
Durable fan-out (classic):
  S3/EC2 event → SNS topic → SQS A → Lambda A
                           → SQS B → ECS workers
                           → SQS C → on-prem poller
  (each queue buffers independently; DLQs per queue)

Filtered subscriptions:
  Order events → SNS → filter policy: region=eu → EU consumer queue
                              type=refund → returns queue

Alerting:
  CloudWatch alarm → SNS topic → email + Lambda (auto-remediation) + chat webhook

Ordered fan-out:
  transaction events → SNS FIFO → SQS FIFO queues (strict order per customer group)
  → Lambda triggered from SQS FIFO (never directly from SNS FIFO)
```

## 9. SAA-C03 Perspective

SNS is a **decoupling/fan-out/notification** staple (Domains 1, 3):

- **"One event must be processed by multiple independent services"** → **SNS topic** (usually **SNS → SQS per consumer**)
- **"Send SMS/email/push to users"** → **SNS** (A2P)
- **"Strict ordering + dedup to multiple queues"** → **SNS FIFO + SQS FIFO**
- **Filtering** = subscription filter policies (cheaper than every consumer filtering itself)
- **FIFO → Lambda requires SQS in between** — a favorite gotcha
- **SNS DLQ** for failed pushes; configure retries/backoff
- **Fan-out durability**: SNS alone is at-least-once but ephemeral; add **SQS** for buffering/independent scaling
- **vs EventBridge**: if the question mentions SaaS events, schema registry, many buses, or complex event patterns across services → EventBridge
- **vs SQS**: one-to-many → SNS; one-to-one durable buffer → SQS

Exam trap: "process the same uploaded file with three different pipelines, each scaling independently" → **S3 event → SNS → three SQS queues → three consumers**. "Notify users by SMS and email" → **SNS**. "SNS FIFO directly triggers Lambda" → **not supported; use SQS first**.