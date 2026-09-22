# Step Functions Course — Serverless Orchestration

## 1. Purpose

Step Functions is AWS's **serverless workflow orchestration** service. You define a **state machine** (in Amazon States Language) that coordinates Lambda, ECS, SQS, DynamoDB, and 200+ other services into multi-step processes with **retries, error handling, branching, parallelism, and visual debugging** — without writing glue code. It's the answer for **durable, auditable, long-running business processes** that would otherwise become a tangle of Lambda-invokes-Lambda calls.

## 2. How it works

- You define a **state machine** in **ASL (JSON)** with **states**: `Task` (do work), `Choice`, `Parallel`, `Map`, `Wait`, `Pass`, `Succeed`, `Fail`, `Catch/Retry`
- Each execution has an **input**, flows through states, and produces an **output**
- **Task states** call services via **optimized integrations** (Lambda, ECS/Fargate, SQS, SNS, DynamoDB, Glue, Athena, Batch, EMR, SageMaker, Bedrock…) or generic **AWS SDK integrations** (200+ services)
- Integration patterns:
  - **Request Response** — synchronous call (Standard & Express)
  - **`.sync` (Run a Job)** — wait until the job completes (Standard & Express, supported services)
  - **`.waitForTaskToken`** — wait for an external callback with a **task token** (Standard only) — the human/external system calls `SendTaskSuccess`
- Two workflow types (immutable at creation):
  | | **Standard** | **Express** |
  |---|---|---|
  | Duration | up to **1 year** | up to **5 minutes** |
  | Execution rate | **2,000/s** | **100,000/s** |
  | State transitions | 4,000/s | nearly unlimited |
  | Guarantee | **exactly-once** | at-least-once (async) / at-most-once (sync) |
  | History | 90 days via API | CloudWatch Logs |
  | Billing | by **state transitions** | by executions + duration + memory |
- Started by API, EventBridge (140+ sources), API Gateway, Lambda, etc.

```
Input → Task (Lambda) → Choice ─┬─ yes → Parallel (Map A ∥ Map B) → Task (SQS) → Succeed
                                └─ no  → Wait → Task (callback token) ─▶ external → Success
  Retry/Catch on every Task; full execution history in console
```

## 3. When to use

- **Multi-step business workflows** — order fulfillment, claims processing, ETL pipelines, approval flows
- **Orchestrate across many services** without glue code (Lambda + ECS + SQS + DynamoDB…)
- **Human-in-the-loop / approvals** — `.waitForTaskToken` pauses until a human/app confirms
- **Long-running processes** (hours/days) → **Standard** (up to 1 year)
- **High-volume short event processing** (streaming, IoT, mobile) → **Express** (100k/s, ≤5 min)
- **Durable, auditable, retriable processes** — exactly-once, full history, visual map
- **Error handling & retries** across distributed steps (catch, retry with backoff)
- **Fan-out/fan-in with `Map`** — process arrays in parallel, then join
- **Replacing Lambda-chaining** that's hard to debug

## 4. When NOT to use

- **Simple one-shot task** → just invoke Lambda directly
- **Pure pub/sub or fan-out** → SNS/EventBridge
- **Durable work queue** → SQS (Step Functions isn't a queue; it orchestrates)
- **Real-time streaming processing** → Kinesis/Firehose
- **Workflows needing >5 min AND Express-level throughput** → split (Standard orchestrates, Express runs short sub-steps)
- **Millisecond-latency synchronous API** → API Gateway + Lambda directly (Express still has orchestration overhead)
- **Need `.waitForTaskToken` or long history on Express** → **Standard only**
- **Global coordination across regions** → not multi-region; build per-region or use another pattern
- **Pure cron scheduling** → EventBridge Scheduler (can start Step Functions, but don't use SF as a cron)

## 5. Important features

- **Standard workflow** — exactly-once, up to **1 year**, 90-day history, visual debugging, **billed by state transitions**; suited to non-idempotent actions (payments, EMR cluster start)
- **Express workflow** — up to **5 min**, **100k executions/s**, billed by exec/duration/memory; **async = at-least-once**, **sync = at-most-once** (via API Gateway/Lambda/`StartSyncExecution`); logs to **CloudWatch**
- **ASL states** — Task, Choice (branching), Parallel, **Map** (iterate over array, with bounded concurrency), Wait, Pass, Succeed, Fail
- **Error handling** — **`Retry`** (backoff, max attempts, error filters) and **`Catch`** per state; catch to fallback states
- **Integrations** — optimized for Lambda, ECS/Fargate, EKS, Batch, EMR, Glue, Athena, DynamoDB, SQS, SNS, Step Functions, Bedrock, SageMaker, CodeBuild, MediaConvert…; **AWS SDK integration** for 200+ services
- **Task tokens (`.waitForTaskToken`)** — pause for external callback (human approval, 3rd-party API) — Standard only
- **`.sync` pattern** — run job to completion (ECS/Fargate/EKS/Batch/EMR/Glue/Athena/CodeBuild/MediaConvert/Bedrock…)
- **Standard + Express combine** — Standard orchestrates; Express handles high-volume short subtasks cost-effectively
- **Visual workflow** — Workflow Studio, execution graph, per-state input/output/timing in console
- **Express in API Gateway** — synchronous Express behind an API for fast request/response
- **Triggered by** — API, EventBridge (140+ sources), API GW, Lambda, IoT rules
- **Pricing** — Standard: **per state transition**; Express: per execution + GB-seconds

## 6. Limitations

- **Workflow type immutable** — can't convert Standard ↔ Express after creation
- **Express ≤5 min, no task tokens, limited history** (CloudWatch Logs, not full 90-day API history)
- **Standard billed by transitions** — chatty high-frequency workflows get expensive (use Express for high volume)
- **Exactly-once is per execution, not per task-call semantics for all services** — design idempotent tasks anyway (Retry can re-run a state)
- **Single-region** — no built-in cross-region orchestration
- **Not a queue/stream/bus** — no buffering; pair with SQS/SNS/EventBridge as needed
- **ASL learning curve** — JSON DSL; Map/Parallel max concurrency limits apply
- **Payload size limits** on state input/output (large data → S3 references)
- **Human tasks need extra infra** — token + external callback (no built-in human-approval UI)
- **Express sync executions** don't contribute to account capacity but still have duration caps
- **No native cron** — use EventBridge Scheduler/CloudWatch Events to start executions

## 7. Trade-offs

- **Standard vs Express** — long-running exactly-once auditable (≤1yr, 2k/s, billed per transition) vs short high-volume cheap (≤5min, 100k/s, billed per exec/duration); **immutable choice**
- **Step Functions vs Lambda-chaining** — durable history/retries/visual vs simpler but fragile invoke chains; SF better for multi-step + error paths
- **Step Functions vs EventBridge** — orchestrate a defined sequence vs react to events; often **EventBridge rule starts a state machine**
- **Step Functions vs SQS/SNS** — orchestration/sequencing vs queue/fan-out; combine (SF task sends to SQS, or SF waits on SQS-driven callback)
- **Step Functions vs AWS Glue/Airflow (MWAA)** — serverless code-first orchestration vs managed Airflow for complex DAG/Python-heavy data pipelines
- **`.sync` vs `.waitForTaskToken`** — wait for AWS service job vs wait for any external/human callback
- **Per-transition pricing vs Express** — control-heavy Standard workflows cost more at scale; offload hot paths to Express
- **Standard exactly-once vs Express at-least-once** — payments/cluster ops vs idempotent event processing

## 8. Architecture

Reference orchestration patterns:

```
Order workflow (Standard):
  API → state machine → validate (Lambda) → Choice
        ├─ in-stock → reserve (DynamoDB) → charge (task token → payment svc) → ship (ECS) → done
        └─ out-of-stock → notify (SNS) → wait (1d) → retry check
  Retry/Catch on charge; full 90-day audit trail

High-volume ETL (Express fan-out):
  EventBridge (file landed) → Express → Map (per shard) → transform (Lambda) → write (S3)
  (cheap at 100k/s, ≤5 min each)

Long-running + hot-path combo:
  Standard orchestrates the month-end job → invokes Express sub-workflows for high-volume rows
```

## 9. SAA-C03 Perspective

Step Functions is the **orchestration / decoupling / serverless** answer (Domains 1, 3):

- **"Coordinate multiple Lambda/services into a durable multi-step process"** → **Step Functions**
- **"Human approval in the middle of a workflow"** → **`.waitForTaskToken` (Standard)**
- **"Run a job and wait for it to finish (ECS/EMR/Glue/Batch)"** → **`.sync` integration**
- **"High-volume, short (<5 min), cost-sensitive event processing"** → **Express**
- **"Long-running workflow / needs exactly-once / full history"** → **Standard**
- **Billed by state transitions (Standard)** vs Express (executions + duration + memory) — cost angle
- **Retries/Catch** built in — resilience without custom code
- **`Map` state** for parallel array processing; **Choice** for branching
- **Workflow type immutable** — pick correctly up front
- **Trigger from EventBridge** for event-driven workflows; **API Gateway → Express** for sync APIs

Exam traps: "orchestrate a 3-step order process with retries and a visual audit trail" → **Standard Step Functions**; "process 50,000 streaming events/sec, each under 1 minute" → **Express**; "pause until a human approves" → **task token (Standard only)**; "Standard is exactly-once, Express async is at-least-once"; "don't use Step Functions as a cron — use EventBridge Scheduler to start it".