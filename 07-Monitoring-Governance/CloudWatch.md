# CloudWatch — Monitoring & Observability

## What it is

CloudWatch is AWS's native monitoring and observability service. It collects metrics, logs, and traces from every AWS service and custom applications, then lets you visualize, alarm, and react to changes. Think of it as the central nervous system for anything running in AWS — if it runs in the cloud, CloudWatch can watch it.

For the SAA exam, CloudWatch appears in nearly every scenario that involves monitoring, alerting, troubleshooting, or operational visibility. You need to know which feature solves which problem, when to use Logs vs Metrics vs Traces, and how alarms integrate with Auto Scaling and SNS.

## Metrics

- **Namespaced**: every metric belongs to a namespace (e.g. `AWS/EC2`, `AWS/RDS`, `AWS/Lambda`). Custom metrics use your own namespace.
- **Dimensions**: key-value pairs that filter/group metrics (e.g. `InstanceId=i-0abc`, `AutoScalingGroupName=web`).
- **Resolution**: standard (60 s) or **high-resolution** (1 s). High-res costs more but catches sub-minute spikes.
- **Statistics**: Average, Sum, Min, Max, SampleCount, p99, p95, etc. Applied over a period.
- **Granularity**: Period defines the length of each data point; Length defines the lookback window for the statistic.

### Standard vs Detailed Monitoring

| | Standard | Detailed |
|---|---|---|
| Default | Yes | Opt-in |
| Period | 5 min | 1 min |
| Use | General workloads | Latency-sensitive / tight autoscaling |

Detailed monitoring is free to enable on launch — it just increases metric granularity.

### Custom Metrics

- Publish your own metrics via **PutMetricData** API (CLI/SDK).
- Use **EMF (Embedded Metric Format)** — put structured JSON in CloudWatch Logs and CloudWatch extracts metrics automatically (no API call needed).
- Custom metrics support **1-second high resolution**.
- Metric streams can export metrics to **S3**, **Datadog**, **Splunk**, or any HTTP endpoint.

## Alarms

- Watch a single metric over time → trigger action when threshold breached.
- States: **OK** → **ALARM** → **INSUFFICIENT_DATA** → OK (and back).
- **Evaluation periods**: how many consecutive periods must breach before triggering.
- **Datapoints to alarm**: e.g. 3 out of 5 periods → tolerant of single blips.

### Alarm actions

| Action | What it does |
|---|---|
| **SNS topic** | Send notification (email, SMS, Lambda, HTTP) |
| **Auto Scaling policy** | Scale in/out |
| **EC2 stop/terminate/recover** | Self-healing |
| **Systems Manager action** | Run automation documents |
| **Inspector** | Trigger findings |

### Alarm types

| Type | How it works | Exam scenario |
|---|---|---|
| **Static threshold** | Fixed number (e.g. CPU > 80%) | Simple alerting |
| **Anomaly detection** | ML baseline → bands around expected value → alarm on deviation | Workloads with natural cycles (daily, weekly) |
| **Composite** | AND/OR of multiple alarms → parent alarm | Reduce noise — only page when multiple things are bad |
| **Metric math** | Expressions combining multiple metrics (e.g. `m1/m2*100`) | Derived metrics like error rate percentage |

- **Missing data**: `missing`, `breaching`, `notBreaching`, or `ignore` — configurable per alarm.
- **Treat missing data as breaching** is a common exam trap — if your app goes down, metrics stop, and a "missing = breaching" alarm correctly fires.

## Logs

- **Log group**: logical container (e.g. `/aws/lambda/my-func`, `/var/log/syslog`). Retention policy set here.
- **Log stream**: sequence of log events within a group (one per instance/container).
- **Log event**: timestamp + message payload.

### Retention & storage

| Setting | Default | Range |
|---|---|---|
| Retention | **Never expire** | 1 day → 10 years (fixed options) |
| Storage | Encrypted at rest (SSE-SQS default, KMS optional) | — |
| Ingestion | $0.50/GB (first 10 GB/mo free tier) | — |
| Storage | $0.03/GB/month | — |

### Log patterns

- Filter patterns use JSON filter expressions or space-delimited patterns.
- Example: `{ $.errorCode = "AccessDenied" }` or `{ $.duration > 1000 }`.
- **Subscription filters**: pipe matching logs to Lambda, Kinesis Data Firehose, OpenSearch, or Datadog in real time.

### Logs Insights

- Interactive SQL-like query engine for CloudWatch Logs.
- Supports `fields`, `filter`, `stats`, `sort`, `limit`, `parse`, `display`.
- Queries are ad-hoc — no indexing needed, results return in seconds for recent data.
- Useful for troubleshooting and root cause analysis during incidents.

### Logs live tail

- Real-time streaming of log events as they arrive — like `tail -f` but in the console or CLI.
- Useful during deployments or debugging in progress.

## CloudWatch Agent

- Single unified agent for **metrics** and **logs** from EC2 instances and on-premises servers.
- Installed via SSM or manually; configured via JSON config file.
- Collects **system-level metrics** not available by default: memory, disk, swap, network, process count, CPU steal, etc.
- Custom log collection: any file or log location (e.g. `/var/log/nginx/access.log`).
- Supports **statsd** and **collectd** protocols for existing monitoring tools.

### Why you need the agent

- Default EC2 metrics (CPU, network, disk I/O) come from the hypervisor — **no memory or disk space metrics** without the agent.
- On-premises or hybrid: agent is the bridge for CloudWatch to monitor non-AWS servers.

## Dashboard

- Custom visual dashboards with widgets (metrics, logs, alarms, text, graphs).
- Widgets support: line graphs, stacked area, bar, number, single value, text, alarm status.
- **Auto refresh** options: 10 s, 1 s, custom.
- Shareable via URL (read-only) or IAM.
- Supports **cross-account** and **cross-region** widgets.
- Export to **CloudWatch Synthetics canaries** for monitoring dashboard health.

## Synthetic monitoring (Synthetics)

- **Canaries**: Lambda functions that run scripts on a schedule to test endpoints.
- Monitor APIs, URLs, and critical user flows from multiple AWS Regions.
- Catch regressions, latency issues, broken links, broken UI.
- Built on **Puppeteer** (Chrome headless) for browser-based testing or **API Gateway** for API checks.
- Output: screenshots, HAR files, logs, metrics → CloudWatch dashboards/alarms.

## Contributor Insights

- Analyzes **log data** or **metrics** in real time to find top contributors.
- Use cases: top 10 IP addresses hitting your ALB, most frequent error codes, busiest API paths.
- Works with CloudWatch Logs, RDS log files, VPC Flow Logs, CloudFront access logs.
- Shows trending data — helps spot emerging issues before they become incidents.

## Application Insights

- Automated monitoring for applications running on EC2 or on-premises.
- Discovers and monitors associated AWS resources (RDS, SQS, EBS, etc.).
- Creates **recommended CloudWatch alarms** and dashboards.
- Uses ML to detect anomalies across correlated metrics.
- Integrates with **X-Ray** for distributed tracing.

## Cross-account observability

- **CloudWatch cross-account observability**: monitor and troubleshoot across multiple AWS accounts from a single account.
- **Metric streams** can export to third-party tools (Datadog, Splunk, Sumo Logic) via Kinesis Data Firehose.
- **Organization-wide dashboards** using IAM and resource policies.

## Lambda monitoring

- Lambda automatically sends metrics to CloudWatch: invocations, duration, errors, throttles, concurrent executions.
- **Logs**: each invocation creates log events in `/aws/lambda/<function-name>`.
- **Duration metric** — use to right-size Lambda memory (more memory = more CPU = faster execution).
- **IteratorAge** metric for stream-based triggers — tells you how far behind consumers are.

## ECS / EKS monitoring

- Container Insights: collect, aggregate, and summarize metrics and logs from containers.
- Metrics: CPU/memory utilization, storage, network per task or service.
- Available for **ECS** (Fargate and EC2 launch type) and **EKS**.
- Logs: typically pushed to CloudWatch Logs via **awslogs** log driver (ECS) or **Fluent Bit** sidecar (EKS).

## RDS monitoring

- **Enhanced Monitoring**: real-time OS-level metrics (CPU, memory, disk, network) via agent → CloudWatch Logs. Granularity: 1 s–60 s.
- **Performance Insights**: identifies database load bottlenecks, top SQL queries, wait events. Free tier for 7 days; paid for longer retention.
- Both are complementary: Enhanced Monitoring = OS level, Performance Insights = database query level.

## Pricing

| Component | Cost |
|---|---|
| Metrics (first 10,000) | Free |
| Metrics (above 10,000) | $0.30/metric/month |
| Custom metrics | $0.30/metric/month |
| High-resolution metrics | $0.30/metric/month |
| Alarms (first 10) | Free |
| Alarms (above 10) | $0.10/alarm/month |
| Logs (ingestion) | $0.50/GB |
| Logs (storage) | $0.03/GB/month |
| Logs Insights (queries) | $0.005/GB scanned |
| Dashboards | $3.00/dashboard/month |
| Synthetics canaries | $0.0012/minute |
| Contributor Insights | $0.50/million log events evaluated |
| Metric streams | Kinesis Data Firehose pricing |

## Exam domains

- [x] **Secure (30%)** — Log group IAM policies, KMS encryption on logs, cross-account roles for metric streams
- [x] **Resilient (26%)** — Composite alarms, anomaly detection, missing data handling, multi-AZ log replication
- [x] **High-Performing (24%)** — High-resolution metrics, Logs Insights for fast troubleshooting, Container Insights
- [x] **Cost-Optimized (20%)** — Retention policies (delete old logs), metric filtering to reduce ingestion, right-size Lambda with duration metrics

## Key gotchas

1. **Default EC2 metrics don't include memory or disk space** — install the CloudWatch agent for those
2. **CloudWatch Logs are not free** — ingestion costs $0.50/GB; set retention to avoid storage creep
3. **Metric resolution**: standard is 60 s, high-res is 1 s — don't pay for high-res unless you need sub-minute
4. **Alarms evaluate across periods** — understand `datapoints_to_alarm` vs `evaluation_periods`
5. **Treat missing data as breaching** is often the correct answer for availability alarms
6. **Subscription filters** are real-time; **metric filters** create CloudWatch metrics from logs (both useful, different purposes)
7. **Logs Insights** queries scan log data — cost is per GB scanned, not per query
8. **CloudWatch agent requires IAM permissions** (CloudWatchAgentServerPolicy) to push metrics/logs
9. **Dashboards cost $3/month each** — build only what you need; use CloudWatch default dashboards (free) for basics
10. **Metric streams** go to Kinesis Data Firehose — you pay for Firehose delivery on top of CloudWatch costs

## Related services

- [[CloudTrail]] — audit log of API calls (who did what, when) vs CloudWatch operational metrics
- [[EventBridge]] — event-driven rules triggered by CloudWatch alarms or metric thresholds
- [[X-Ray]] — distributed tracing for request paths across microservices
- [[SNS]] — notification target for CloudWatch alarm actions
- [[Auto-scaling]] — scales based on CloudWatch metrics (CPU, custom, etc.)
- [[Lambda]] — triggers from alarms, runs in response to events
- [[SSM]] — runbooks execute in response to CloudWatch alarms
- [[OpenSearch-Service]] — centralized log analytics destination
- [[Kinesis]] — real-time log streaming to Firehose → S3/OpenSearch
