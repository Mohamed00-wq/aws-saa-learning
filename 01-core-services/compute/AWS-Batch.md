# AWS Batch Course — Fully Managed Batch Computing at Scale

## 1. Purpose

AWS Batch runs **hundreds of thousands of batch computing jobs** at any scale using EC2 (On-Demand/Spot, Graviton) and Fargate — fully managed, no batch scheduler to run. You package your job as a **Docker container image**, submit it to a **job queue**, and Batch picks a **compute environment** (ECs) that can use **On-Demand or capacity-optimized Spot Instances** to run it until completion. For SAA it's the answer to **"batch processing / run-to-completion jobs / large-scale compute / scheduled jobs,"** and the natural pair for **Spot Instances for cost**.

## 2. How it works

- **Job definition** — JSON that describes the job: container image (from ECR), vCPU/memory requirements, environment variables, retry strategy, timeout
- **Job queue** — jobs wait here; queues have a **priority ordering** and map (possibly many-to-many) to compute environments
- **Compute environment** — managed (Batch provisions/spatches/scales EC2 + Fargate for you) or unmanaged (you control the instances); **ECS Managed Instances**; can be On-Demand, Spot, or Fargate
- **Dependency graph** — job A only starts after B succeeds; sequential/parallel DAG of steps
- **Array jobs** — one job definition fanned out to hundreds/thousands of replicas (same container, different input) — great for embarrassingly parallel work
- **Multi-node parallel jobs** — coordinate multiple nodes (MPI/HPC, tightly-coupled workloads) sharing a `/dev/shm`-backed network filesystem
- **Events** — state changes enqueue to **EventBridge** (e.g., trigger downstream steps on SUCCEEDED/FAILED)

```
App / CLI / EventBridge → submit Job Definition (ECR image)
        │                        │
        ▼                        ▼
   Job Queue ──────► Compute Environment (EC2 On-Demand / Spot / Fargate, auto-scales)
        │                        │
        └──────── jobs run to completion → results to S3 / DynamoDB / downstream service
```

## 3. When to use

- **Batch data processing / ETL-style** workloads that "run to completion" with no interactive UI
- **ML training / inference jobs**, genomic analysis, financial Monte-Carlo simulations, risk modeling
- **Media/image/video transcoding** at scale, large-scale log-file processing
- **Scheduled jobs** (e.g., nightly reconciliation) combined with EventBridge schedules
- **Embarrassingly parallel (array job) fan-out workloads**
- **Cost-sensitive compute** — run on **Spot with capacity-optimized strategy** (Batch is the classic Spot companion)

## 4. When NOT to use

- **Interactive / long-running services** (web apps, APIs) → ECS/EC2/Beanstalk/Lambda
- **Real-time streaming ingestion** with sub-second latency → Kinesis / SQS / Lambda
- **Short, event-driven, serverless tasks** (< 15 min, small payloads) → **Lambda**
- **You don't want to containerize the job** — Batch jobs are Docker containers
- **Simple managed ETL pipelines** → AWS Glue (crawlers, transforms, serverless Spark)

## 5. Important features

- **Managed compute environments** — Batch provisions and auto-scales EC2 (incl. **Spot with capacity-optimized/nil strategies**) or **Fargate**
- **Job queues with priority ordering** — route work to On-Demand or Spot queues based on cost vs urgency
- **Random/array jobs** — fan-out hundreds of thousands of containers; **multi-node parallel (MPI)** for HPC
- **Job dependencies & retries** — DAG ordering, retry/backoff, timeouts
- **Any Linux Docker image** (from ECR) — full control of runtime
- **EventBridge integration** — react to `SUCCEEDED`/`FAILED`/`RUNNING` events, schedule with EventBridge Scheduler
- **CloudWatch logs/metrics** per job, job placeholders, parameters, envvars
- **Price-performance** — Spot-first for tolerant workloads; On-Demand for critical jobs

## 6. Limitations

- Jobs are **containerized** — you maintain images/ECR (though you can use public images)
- **Not real-time** — provisioning/scaling adds minutes of queue latency (cold start on new instances)
- Job sizes have quotas (vCPU/memory limits per job/queue) — plan big jobs as arrays
- No native Windows containers (Linux via Docker / Fargate); Fargate jobs have vCPU/memory ceilings
- Compute environments need time to scale; very bursty spiky demand can wait

## 7. Trade-offs

- **Batch vs Lambda** — long-running (minutes-hours, containerized, DAG, ML/HPC) vs short (< 15 min, event-driven, serverless micro-functions)
- **Batch vs ECS/EKS** — managed job scheduler queue/dispatch model vs you operate a long-running service fleet
- **Batch vs Glue** — arbitrary containers/generic batch vs managed serverless Spark ETL
- **On-Demand vs Spot compute envs** — guaranteed availability/cost vs ~90%-cheaper but interruptible (Batch retries on Spot interruptions with capacity-optimized strategy)
- **Fargate vs EC2 compute env** — no instance management, simpler vs cheapest/custom EC2 instances

## 8. Architecture

```
Scheduled trigger (EventBridge Scheduler) → submit array job
  → Job Queue (priority) → Compute Environment (Spot fleet, capacity-optimized)
  → N containers read input from S3, write results to S3/DynamoDB
EventBridge sees SUCCEEDED → runs next pipeline stage
Failures → retry policy → DLQ-equivalent / alarm
```

## 9. SAA-C03 Perspective

AWS Batch appears in **Domain 3 (high availability) / Domain 4 (cost / design)**:

- **"Batch / batch processing / run-to-completion jobs at scale"** → **AWS Batch**
- **"Run compute as cheaply as possible for a fault-tolerant workload"** → **Batch + Spot (capacity-optimized)**
- **"Monte Carlo / ML training / media transcoding / genomics"** → array jobs on Batch
- **"Thousands of independent, identical tasks"** → **array jobs**
- **"Process only after previous step succeeds"** → job dependencies (DAG)
- **"Scheduled nightly heavy job"** → Batch + EventBridge Scheduler

Exam traps: "Batch is for real-time streaming" → **no, containerized run-to-completion jobs**; "Batch replaces Lambda for all compute" → **no, Lambda for short serverless tasks**; "On-Demand is cheapest" → **Spot is cheapest, use for tolerant workloads**; "jobs are scripts not containers" → **jobs are Docker containers**.