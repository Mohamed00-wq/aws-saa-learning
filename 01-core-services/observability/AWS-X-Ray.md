# AWS X-Ray  Distributed Tracing

## What it is

AWS X-Ray is a distributed tracing service that helps you analyze and debug requests across microservices. It collects timing data for each hop  API Gateway → Lambda → DynamoDB → SQS  and stitches them into a single **trace** so you can see where latency lives. For the SAA exam, know the difference between X-Ray and CloudWatch, and when to use active vs passive tracing.

## Core concepts

| Term | Definition |
|---|---|
| **Trace** | Complete path of a single request (identified by `TraceId`) |
| **Segment** | A single unit of work (one Lambda invocation, one API Gateway call) |
| **Subsegment** | Finer breakdown within a segment (e.g. a DynamoDB query) |
| **Annotation** | Key-value pairs for **indexing and search** (e.g. `userId: 123`) |
| **Metadata** | Key-value pairs for detailed data, not indexed |
| **Service graph** | Visual map of all services and how they connect |

## Instrumentation

### X-Ray SDK & Daemon

- SDK available for **Python, Node.js, Java, .NET, Go, Ruby**. Patches AWS SDK calls automatically.
- **Daemon**: lightweight UDP agent (port `2000`), buffers segments locally then sends to X-Ray.
- ECS: **sidecar container**. EKS: **DaemonSet**. Lambda: daemon is **managed by AWS**  just enable active tracing.

### Auto-instrumentation

- **API Gateway**: enable X-Ray tracing at stage level. **Lambda**: enable active tracing in function config.
- **Elastic Beanstalk**: add daemon via `.ebextensions`. **ECS/Fargate**: sidecar container pattern.

## Active vs Passive tracing

| Mode | How it works | Use case |
|---|---|---|
| **Active** | App pushes segments via SDK/daemon | Full control, subsegments, annotations |
| **Passive** | X-Ray reads from CloudWatch Logs / API Gateway access logs | No code changes, basic visibility |

## Sampling rules

- Control **which requests** get traced. **Default**: first request/s + 5% of additional requests.
- **Custom rules**: match by ARN, HTTP method, host → set fixed rate. **Reservoir**: guaranteed minimum traces/s.
- Sampling happens **at the entry point** and propagates downstream.

## Service map & groups

- **Service map**: auto-generated from traces  latency, error rates, request counts per service. Color-coded health.
- **Groups**: filter expressions for persistent trace groups (e.g. `{ fault = true }`). Can trigger CloudWatch Alarms.

## Encryption

- Trace data encrypted at rest using **AWS-managed KMS keys** (default) or **customer-managed CMK**.
- Encrypted in transit (TLS).

## Integration with other services

| Service | Integration |
|---|---|
| **API Gateway** | Stage-level tracing → gateway latency, status codes |
| **Lambda** | Active tracing → segment + downstream subsegments |
| **ECS / EKS** | X-Ray daemon as sidecar or DaemonSet |
| **SQS / SNS** | Trace propagation via message attributes |
| **DynamoDB** | Auto-instrumented  read/write latency per call |
| **App Mesh** | Envoy sidecar reports trace data to X-Ray |

## Pricing

First 100,000 traces recorded free. Above: $5.00/100k. Scanned: $0.50/1M traces. CMK encryption at KMS pricing.

## Exam domains

- [x] **Secure (30%)**  KMS encryption, IAM roles for daemon, sampling
- [x] **Resilient (26%)**  daemon buffers locally, multi-AZ tracing
- [x] **High-Performing (24%)**  sampling rules, annotations for fast search
- [x] **Cost-Optimized (20%)**  sampling rate tuning, annotations vs metadata

## Key gotchas

1. **X-Ray daemon uses UDP port 2000**  security groups must allow outbound
2. **Lambda needs active tracing enabled**  won't trace by default
3. **Annotations are indexed, metadata is not**  searchable fields in annotations
4. **Sampling is entry-point-based**  downstream inherits the decision
5. **Traces expire after 30 days**  X-Ray doesn't store indefinitely
6. **X-Ray is regional**  cross-region requires CloudWatch ServiceLens
7. **SQS/SNS propagation**  both producer/consumer must use X-Ray SDK

## Related services

- **CloudWatch**  metrics/logs X-Ray provides the tracing layer
- **CloudTrail**  audit log X-Ray traces execution paths
- **Lambda**  primary service instrumented with X-Ray
