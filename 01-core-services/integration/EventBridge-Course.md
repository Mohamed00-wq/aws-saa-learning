# EventBridge Course — Serverless Event Bus

## 1. Purpose

EventBridge is AWS's **serverless event bus** for **event-driven architectures**. It receives events from AWS services, your apps, and SaaS partners, then **routes them to targets** based on **event-pattern rules** — with optional filtering, transformation, and enrichment. It replaces a lot of hand-rolled "SNS topic + custom filter code" glue, and it's the modern way to **react to what's happening across your account and SaaS tools** with near-zero infrastructure.

## 2. How it works

- An **event source** (AWS service, custom app, SaaS partner) sends a JSON **event** to an **event bus**
- **Rules** on that bus match events by an **event pattern** (JSON matching on fields like `source`, `detail-type`, `detail`)
- Matching events are sent to up to **5 targets per rule** (Lambda, SQS, SNS, Step Functions, Kinesis, API destinations, other buses…), processed in parallel
- Three bus types: **default** (AWS service events in your account), **custom** (your own events), **partner/SaaS** (third-party events)
- Beyond rules, EventBridge also offers:
  - **Pipes** — **point-to-point** source → (filter/enrich/transform) → single target
  - **Scheduler** — **cron/rate + one-time** scheduled tasks at scale (replaces legacy scheduled rules)
  - **Schema Registry** — discover/validate event schemas, generate code bindings

```
AWS service / app / SaaS → event bus → rule (event pattern)
                                          → target(s): Lambda, SQS, Step Functions…
                                          (≤5 targets, parallel)
Pipes: source ──filter/enrich──▶ single target (point-to-point)
Scheduler: cron/rate/one-time ──▶ many target APIs
```

## 3. When to use

- **Event-driven / reactive architectures** — react to state changes (EC2 state, S3 object created, CodePipeline stage, etc.)
- **Cross-service / cross-account glue** — one bus routes events between accounts (resource policies)
- **SaaS integration** — react to events from Datadog, PagerDuty, Zendesk, etc. without polling
- **Many producers → many consumers with filtering** — rules let each team subscribe to just what it needs
- **Scheduled jobs** — cron/rate tasks at scale → **EventBridge Scheduler**
- **Point-to-point with enrichment** — **Pipes** (e.g., DynamoDB Stream → enrich with Lambda → SQS/Step Functions)
- **Loose coupling** — producers don't know consumers; add consumers without changing producers

## 4. When NOT to use

- **Durable work queue / buffering** → **SQS** (EventBridge isn't a queue; it delivers then discards)
- **Fan-out to many consumers each needing their own durable buffer** → **SNS → SQS** (or rules to separate queues)
- **Replayable streaming / real-time analytics across consumer groups** → **Kinesis Data Streams**
- **Strict ordering + dedup** → **SNS/SQS FIFO**
- **Simple one-to-one notify** → SNS or direct Lambda trigger may be simpler
- **Workflow orchestration with retries/branches/visual history** → **Step Functions** (EventBridge can *start* one)
- **Very high-throughput ordered streaming** → Kinesis/Kafka (MSK)
- **Legacy scheduled rules** — prefer **EventBridge Scheduler** for new schedules

## 5. Important features

- **Event pattern rules** — JSON matching (`source`, `detail-type`, `detail.*`), content filtering, matching multiple fields; event can match **multiple rules**
- **Up to 5 targets per rule** (AWS recommends **one target per rule** for manageability — duplicate the rule instead)
- **Event buses** — default / custom / partner; **cross-account routing** via bus resource policies (separate teams own their rules)
- **Targets** — Lambda, SQS, SNS, Kinesis, Step Functions, ECS/Fargate, API Gateway, API Destinations (HTTP), other buses, systems-manager, and more
- **Input transformation** — reshape/filter the event before it hits the target (InputTransformer/InputPath)
- **Pipes** — single source → optional **filter + enrichment (Lambda/API) + transform** → single target; preserves source order; **you only pay for events matching the filter**; sources include DynamoDB Streams, Kinesis, SQS, Kafka, EventBridge buses
- **Scheduler** — **cron & rate expressions**, one-time or recurring, flexible windows, retry limits, retention for failed invocations; scales far beyond legacy scheduled rules; thousands of schedules per group
- **Schema Registry** — auto-discover schemas, validate events, generate bindings (open-source libraries)
- **Reliability** — retries with backoff up to **24 hours** (or custom policy); **DLQ** for failed targets; CloudWatch metrics (`TriggeredRules`, `ThrottledRules`, `Invocations`, `FailedInvocations`)
- **Archive & replay** (for event buses) — retain and replay events to rules/subscribers
- **SaaS integration** — partner event buses receive events from SaaS providers without you building connectors
- **Private APIs** — PrivateLink / VPC Lattice integration

## 6. Limitations

- **Not a queue** — no long-term per-consumer buffering; add SQS behind targets if you need durability/retry-with-order
- **5 targets per rule max** (duplicate rules to scale out)
- **Rule ≠ pipeline** — no built-in multi-step orchestration (use Step Functions)
- **Event size** — events have payload size limits (large payloads → store in S3 and pass a reference)
- **At-least-once delivery semantics with retries** — targets must be **idempotent**
- **Throttling** — heavily matched rules can throttle (`ThrottledRules`); events still retried up to 24h
- **Cross-region not automatic** — buses are regional
- **Partner/SaaS events need a partner bus** — not all SaaS events land on the default bus
- **Scheduled rules are legacy** — limited vs Scheduler (prefer Scheduler for new work)
- **Debugging pattern mismatches** — a wrong field means silent no-match; use metrics/logs carefully
- **No ordering guarantee across rules/targets** (Pipes preserve order for their source; buses do not guarantee global order)

## 7. Trade-offs

- **EventBridge vs SNS** — filtering/schema/SaaS/cross-account routing vs simple fan-out push to fixed subscribers; SNS+SQS for durable fan-out, EventBridge for richer event routing
- **EventBridge vs SQS** — event routing/delivery-once vs durable buffer/pull consumers
- **EventBridge vs Kinesis** — ephemeral event routing vs replayable ordered stream for analytics
- **EventBridge Rules vs Pipes** — many-to-many bus routing vs single point-to-point with enrichment (Pipes often cheaper + ordered for one source→target)
- **EventBridge Scheduler vs CloudWatch/legacy scheduled rules** — scale, retries, many targets vs simple legacy cron rule
- **EventBridge vs Step Functions** — trigger vs orchestrate; often combined (rule starts a state machine)
- **Event pattern filtering vs consumer-side filtering** — central, cheaper, consistent vs simpler infra (no bus)
- **Custom bus per team/account** — isolation and autonomy vs more buses to manage

## 8. Architecture

Reference event-driven patterns:

```
Reactive + orchestrate:
  EC2/S3/CodePipeline event → default bus → rule → Step Functions (workflow)
                                              → Lambda (auto-remediation)

Cross-account / multi-team:
  Account A custom bus ──resource policy──▶ Account B bus → B's own rules/targets
  (each consumer team owns its rules)

SaaS:
  PagerDuty/Zendesk partner bus → rule → SQS → Lambda

Point-to-point w/ enrichment:
  DynamoDB Stream → Pipe (filter: INSERT) → enrich Lambda → target SQS/Step Functions

Scheduled:
  EventBridge Scheduler (cron) → batch APIs / Lambda / Step Functions
```

## 9. SAA-C03 Perspective

EventBridge is the go-to for **decoupling, event-driven, and serverless glue** (Domains 1, 3):

- **"React to events across AWS services / SaaS without polling"** → **EventBridge**
- **"Loose coupling between many producers and consumers"** → **event bus + rules**
- **"Complex filtering so each consumer sees only relevant events"** → **EventBridge rules** (or Pipes filters)
- **"Scheduled job every 15 min at scale"** → **EventBridge Scheduler**
- **"One source, one target, enrich the data in between"** → **EventBridge Pipe**
- **"Cross-account event routing"** → **custom bus + resource policy**
- **vs SNS** — if the question is simple notify/fan-out → SNS; if SaaS/schema/many buses/complex patterns → **EventBridge**
- **vs SQS** — need durable buffer/retry with consumer control → **SQS** (EventBridge targets can be SQS)
- **Reliability**: retries + **DLQ** on targets; **idempotent** targets
- **Up to 5 targets per rule**; one-target-per-rule recommended for maintainability

Exam trap: "DynamoDB stream event → enrich with Lambda → send to SQS" → **EventBridge Pipe** (not a Rule). "React to Zendesk ticket created" → **partner event bus**. "Buffer events so slow Lambda doesn't lose them" → **rule target = SQS**, then Lambda on the queue.