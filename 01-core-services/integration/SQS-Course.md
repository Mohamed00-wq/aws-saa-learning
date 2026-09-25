# SQS Course  Simple Queue Service

## 1. Purpose

SQS is AWS's **fully managed message queue**  the workhorse of decoupled architectures. Producers push messages consumers poll and process them independently, so components scale, fail, and deploy separately. It absorbs traffic spikes, buffers work, and prevents one failing service from taking down its producers. It's the default answer whenever a question says **"decouple", "buffer", "asynchronous processing", or "preventing point-to-point coupling"**.

## 2. How it works

- A **producer** sends a message to a queue (retained up to **14 days**)
- A **consumer** polls the queue (**long polling** preferred  `WaitTimeSeconds` up to 20s, cuts empty reads/cost)
- When a consumer receives a message it becomes **invisible** (visibility timeout) so no other consumer gets it
- Consumer processes, then **deletes** the message if it doesn't, the message becomes visible again and is redelivered (**at-least-once** delivery)
- Maximum message size **1 MiB** (use **Extended Client** + S3 for larger payloads)
- Two queue types:
  - **Standard**  **at-least-once**, **best-effort ordering**, **nearly unlimited throughput**
  - **FIFO**  **exactly-once** processing, **strict ordering**, `.fifo` suffix required

```
Producer → SQS queue (visible → in-flight → deleted/visible again)
              ├─ long poll → Consumer group A
              └─ after maxReceiveCount failures → DLQ (dead-letter queue)
```

## 3. When to use

- **Decoupling** producers from consumers (scale them independently)
- **Buffering / smoothing** traffic spikes (ELB → SQS → workers)
- **Work queues**: task distribution, job processing, order processing
- **Fan-out**: SNS → SQS per consumer (each team gets its own queue)
- **Retry + failure isolation**: DLQ holds poison messages for debugging
- **Ordering required** → FIFO (banking, inventory, flight status)
- **Anything that must survive a consumer crash**  messages persist until deleted

## 4. When NOT to use

- **Need strict order AND high throughput with complex filtering** → consider SNS+SQS or EventBridge
- **Request/reply, synchronous RPC** → API Gateway / AppSync, not a queue
- **Broadcast to many different consumers with filtering** → SNS / EventBridge (queues don't fan out on their own)
- **Streaming with replay + real-time analytics across multiple consumer groups** → Kinesis Data Streams
- **Message > 1 MiB** (unless Extended Client/S3)
- **Cross-region / global ordering** → not supported use regional queues or a global solution
- **Workflow orchestration with retries/branches** → Step Functions

## 5. Important features

- **Visibility timeout**  default **30 s**, max **12 h** set longer than your SDK read/processing time to avoid duplicate processing `ChangeMessageVisibility` extends per-message
- **Dead-letter queue (DLQ)**  after `maxReceiveCount` (default 10) receives, message moves to DLQ **DLQ must be same type** (FIFO→FIFO, standard→standard) set DLQ retention **longer** than source alarm on `ApproximateAgeOfOldestMessage`
- **DLQ redrive**  move messages back to source queue after fixing the bug
- **Polling**  short poll (default, 0s wait, inefficient) vs **long poll** (0–20s, recommended, fewer empty responses)
- **Delay queue / message timers**  postpone delivery (0–15 min)
- **Retention period**  1 min to 14 days (default 4 days)
- **FIFO extras**  `MessageGroupId` (parallel ordered groups), `MessageDeduplicationId` or **content-based dedup** (SHA-256, 5-min window) **high-throughput FIFO** up to ~70,000 TPS per API action with batching (700,000 msg/s)
- **Server-side encryption**  SSE-SQS default, SSE-KMS optional
- **In-flight limit**  ~**120,000** concurrent in-flight messages (received not deleted)
- **Fair queues**  throttle noisy consumers **Lambda event source mapping** consumes automatically

## 6. Limitations

- **At-least-once on standard**  duplicates possible consumers must be **idempotent**
- **Best-effort ordering on standard**  no ordering guarantee use FIFO if order matters
- **FIFO FIFO-capable throughput** is lower than standard (mitigated by high-throughput FIFO + many message groups)
- **1 MiB message max**  bigger payloads need S3 references
- **DLQ with FIFO breaks exact order**  messages arrive out of sequence after redrive (AWS warning)
- **No request/reply**  one-way no built-in pub/sub or filtering
- **In-flight cap 120k**  delete messages promptly or add queues
- **No native delay > 15 min**  use scheduled Lambda/EventBridge for longer deferrals
- **Ordering is per message group only**  different groups interleave
- **Global SSO/cross-region replication not built in**

## 7. Trade-offs

- **Standard vs FIFO**  nearly unlimited throughput + at-least-once/best-effort vs strict order + exactly-once + lower throughput (with high-throughput mode, higher cost)
- **SQS vs SNS**  pull/queue/buffer (one consumer per message) vs push/fan-out (many subscribers)
- **SQS vs EventBridge**  simple durable queue vs event filtering/routing/SaaS + schema registry
- **SQS vs Kinesis**  work queue vs replayable multi-consumer stream
- **Long vs short polling**  fewer empty responses/cost vs lowest latency first message
- **Visibility timeout shorter vs longer**  faster redelivery on failure vs risk of duplicate concurrent processing
- **DLQ vs inline retries**  isolate poison messages vs keep everything in one place (DLQ preferred for debuggability)
- **Extended Client vs smaller messages**  2 GB via S3 vs keep queue payloads clean

## 8. Architecture

Reference decoupling patterns:

```
Buffered tier:
  ALB → EC2/ECS producers → SQS standard → ASG consumers (scale on queue depth/age)
     → DLQ for poison messages + CloudWatch alarm

Fan-out (multiple independent consumers):
  Event source → SNS topic → SQS A (billing) + SQS B (analytics) + Lambda
  (each consumer owns its own queue → independent scaling & failure)

Ordered processing:
  Producer → SQS FIFO → group by customer_id → one consumer per group, in order
     → DLQ for failures (know order may break on redrive)

Hybrid: on-prem batch → SQS → Lambda workers or Step Functions coordinating SQS task queues
```

## 9. SAA-C03 Perspective

SQS appears in **decoupling, resilience, and cost** scenarios (Domains 1, 3, 4):

- **"Decouple A from B / prevent coupling / buffer spikes"** → **SQS**
- **"Retries with isolation of failed messages"** → **SQS + DLQ**
- **"Strict order + exactly-once"** → **FIFO** (with message groups for parallelism)
- **Visibility timeout > processing time**  classic config question too short = duplicate processing
- **DLQ type must match** source queue type
- **Fan-out = SNS→SQS** (SQS alone doesn't broadcast)
- **Idempotent consumers** required for standard queues
- **Lambda + SQS** scales on queue depth  use as the "scale workers automatically" answer
- **vs Kinesis** (replay, multiple consumer groups, streaming) and **vs EventBridge** (filtering, SaaS events, cross-account)

Exam traps: "guaranteed exactly-once + unlimited throughput" → false (FIFO trades throughput) "DLQ for FIFO must be FIFO too" "set visibility timeout longer than processing" "long polling reduces empty reads and cost".