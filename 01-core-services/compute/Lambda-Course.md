# Lambda Course  Serverless Compute

## 1. Purpose

Lambda is AWS's **serverless compute** service: you upload code, connect it to a trigger, and AWS runs it for you. No servers to provision, patch, or scale. AWS manages the infrastructure, so you focus purely on **application logic**. You pay only for what you execute (requests + compute time).

## 2. How it works

- Write a **handler function** in a supported runtime (Python, Node.js, Java, Go, .NET, Ruby, Rust…) 
- Connect a **trigger**  API Gateway, S3, SQS, SNS, EventBridge, DynamoDB Streams, 200+ services
- Each invocation runs in its own **execution environment** (isolated, stateless)
- Lambda scales **up to thousands of concurrent executions automatically** per account/region (default 1,000, last used memory 10,240 MB max, timeout up to **15 min**)
- "Warm" environments (reused) respond fast "cold starts" add latency after idle periods
- Pay: `requests + GB-seconds (memory × time)`  $0.20/1M requests, ~$0.0000166667/GB-s Graviton (ARM) ~20% cheaper

```
Event (S3 put / API call / SQS msg / cron) → Lambda → does work → writes to DB/S3/API
```

## 3. When to use

- **Event-driven / short tasks**  handle S3 uploads, SNS/EventBridge events, cron jobs
- **APIs / microservices** via API Gateway  spiky, variable traffic
- **Serverless backends**  mobile/web apps, IoT ingest
- **Data pipelines**  transform, ETL (triggers on S3, Kinesis, DynamoDB streams)
- **Automation & glue**  image/video processing, security remediation, cost checks
- **High burst, variable, or infrequent traffic**  scale from zero, no idle cost
- **Stateless workloads**  each request independent (state goes to DB/S3)

## 4. When NOT to use

- **Long-running processes** (>15 min)  use ECS/EKS, EC2, or Fargate
- **Always-on, predictable steady load**  EC2/Reserved or Fargate may be cheaper
- **Need a server** (OS, daemons, listening on arbitrary ports)  EC2, or Lambda MicroVMs
- **Heavy CPU/GPU/ML training**  GPU instances are better
- **Very low latency with no cold-start tolerance**  provisioned concurrency helps but costs otherwise keep a server warm
- **Stateful / sticky sessions**  Lambda execution environments are ephemeral

## 5. Important features

- **Trigger integrations**  API GW, S3, SQS, SNS, EventBridge, DynamoDB/Kinesis Streams, Cognito, CloudWatch, etc.
- **Auto-scaling**  up to 1,000 concurrent executions (default) per region, 1,000 new environments every 10s per function
- **Pricing model**  pay-per-request + GB-seconds **free tier: 1M requests + 400K GB-s/month forever**
- **Graviton (ARM)**  up to 34% better price/performance vs x86
- **Provisioned Concurrency**  pre-warmed instances, kills cold starts (extra cost)
- **Reserved concurrency**  cap/guarantee concurrency for critical functions (also throttles downstream load)
- **Versions, aliases, canary deployments**  safe rollouts via weighted aliases
- **Layers**  share dependencies/code across functions (up to 5)
- **VPC access**  join a VPC (via ENIs) to reach RDS/private resources
- **Environment variables & secrets**  config + KMS-encrypted secrets
- **Event source mapping / DLQ**  async retries, dead-letter queues for failures
- **Destinations**  route success/failure to SQS/SNS/Lambda (async flows)
- **IAM**  execution roles (least privilege) govern what the function can call
- **New (2026): Durable Functions** (multi-step workflows up to a year) and **Lambda MicroVMs** (near-instant-start isolated environments, 8h sessions, any code/port)

## 6. Limitations

- **Stateless**  no state between invocations (store in DB/S3, not instance memory)
- **Timeout cap  15 min** per invocation (huge workflows need Step Functions/Durable Functions)
- **Cold starts**  latency after idle or on burst (Java/.NET worst) unless provisioned concurrency
- **Memory cap**  up to 10,240 MB (CPU scales with it) no GPU
- **Concurrency quota**  default 1,000 throttling → `429 ThrottlingException` when exceeded
- **Deployment package limits**  250 MB zip (unzipped ~250MB incl. layers, ~50MB zipped direct) big binaries via container images / S3
- **/tmp ephemeral storage**  in-memory only (512 MB default, up to 10,240 MB lost after invocation)
- **1,000 ENIs/VPC limits**  VPC functions need ENIs, affects cold start, IP consumption
- **Not for interactive sessions or listening** on arbitrary ports (that's EC2/MicroVM territory)

## 7. Trade-offs

- **Lambda vs EC2**  no ops & pay-per-use vs full control, long-running, cheaper at steady high load
- **Lambda vs Fargate/ECS**  simpler for event tasks vs containers for complex/long-running/port-bound workloads
- **Pay-per-invocation vs provisioned**  idle cost=0 but cold starts always-warm costs money. Choose by latency budget vs cost budget
- **x86 vs ARM (Graviton)**  ~20–34% cheaper, usually drop-in compatible verify dependencies
- **1 big function vs many small**  fewer cold starts/warm overlap vs granular scaling, IAM per function, and cost isolation
- **Cold start vs cost**  provisioned concurrency removes cold starts but bills even when idle

## 8. Architecture

Reference serverless pattern:

```
Client → API Gateway → Lambda (auth/business logic) → DynamoDB / RDS
                          ↘ SQS → Lambda (async workers)
S3 upload → EventBridge/S3 event → Lambda → transform → S3 bucket
CloudWatch cron → Lambda maintenance/cleanup job
```

- **Async**: send to SQS → Lambda processes  decouples bursts, allows retry/DLQ
- **Error handling**: DLQ or Destinations for failed invocations
- **Secrets**: use Secrets Manager with KMS, never plaintext env vars
- **IaC**: deploy with SAM or CloudFormation/CDK
- **Observability**: X-Ray + CloudWatch Logs + Lambda Powertools

## 9. SAA-C03 Perspective

The exam tests **when to pick Lambda vs other compute** and its operational model:

- **Scenario→Lambda**: short-running, event-driven, spiky/unpredictable traffic, serverless/no-ops, integration with S3/SQS/API Gateway/EventBridge
- **Scenario→EC2/ECS**: long-running (>15 min), specific OS/GPU, steady load, or container orchestration
- Know **pricing model** (requests + duration), **cold starts**, and **concurrency** (reserved = throttle/protect downstream DB)
- Know **statelessness**  state must live in external stores
- **Triggers vs event source mappings**  S3/SNS/API GW can push SQS/DynamoDB/Kinesis use polling
- **Retries/DLQs**  async invocation retries twice failures route to DLQ/destinations
- **VPC functions need ENIs**  impacts cold start gateway vs interface endpoints for private resources
- **Graviton ARM**  cost-optimization questions (up to 34% better price/performance)
- **Newer exam-relevant items**: Durable Functions, Lambda MicroVMs  know at a high level

Exam trap: "choose compute for a Unix-based server where you can install custom daemons and it must listen on a port" → **EC2**, not Lambda. Only choose Lambda when the workload is event-driven and short-lived.