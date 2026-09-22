# CloudWatch Course — Monitoring & Observability

## 1. Purpose

CloudWatch is AWS's **native monitoring and observability** service: **metrics, logs, alarms, dashboards, synthetics (canaries), and traces** in one place. It tells you *how* your AWS services and workloads are actually behaving — not just whether resources exist. For SAA, nearly every "monitor / alarm / troubleshoot / react" scenario is a CloudWatch scenario, and it's the input layer for many other services (autoscaling, EventBridge, SNS alerting).

## 2. How it works

- **Metrics** — time-series data, organized by **namespace** (`AWS/EC2`, `AWS/Lambda`…) and filtered by **dimensions** (key-value, e.g. `InstanceId=i-0abc`); **standard 60 s** or **high-resolution 1 s** resolution; statistics: Sum, Average, Min, Max, p99…
- **Alarms** — evaluate metric(s) over periods → **OK → ALARM → INSUFFICIENT_DATA**; act via SNS/EC2 actions/autoscaling
- **Logs** — **log groups** (retention policy) containing **log streams**; ingested from EC2 agent, Lambda, services, on-prem
- **Unified agent** on EC2/on-prem collects OS-level **metrics + logs** (memory, disk, swap, process count — *not* visible from the hypervisor by default)
- **Dashboards** — custom widgets combining metrics/logs/alarms; cross-account/cross-region
- **Event pipeline** — subscription filters (real time to Lambda/Firehose/OpenSearch), metric filters (logs → metrics), Logs Insights (SQL-ish query)

```
Sources (EC2 agent, Lambda, services, on-prem) → Metrics / Logs
   → Alarms (static/anomaly/composite/metric-math) → SNS/EventBridge/action
   → Dashboards; subscription filters → Lambda/Firehose/OpenSearch
   → Canaries (Synthetics) → endpoint availability metrics
   Container Insights (ECS/EKS) → cluster→pod/container metrics + logs
```

## 3. When to use

- **Monitor EC2/ECS/Lambda/RDS/etc.** CPU, network, errors, invocation counts, latency
- **Alarm + react** — page ops, trigger autoscaling, kick off Lambda remediation
- **OS-level EC2 telemetry** (memory/disk/swap) — CloudWatch agent
- **Centralized logging / troubleshooting** — Logs Insights queries, live tail
- **Container health** — **Container Insights** for ECS/EKS/Fargate (enhanced observability: cluster→service→task→container), incl. ECS container health metric + OTel/prometheus metrics for EKS
- **Business metrics from apps** — embedded metric format (EMF), custom metrics, statsd/collectd
- **Synthetic monitoring** — canaries test endpoints/APIs on a schedule (uptime, login flows)
- **Cross-account/cross-region visibility** — dashboards, metric streams, subscriptions
- **Automated anomaly/anatomy discovery** — anomalistic alarms, Contributor Insights (top-N), Application Insights

## 4. When NOT to use

- **Who called what API / who made a change** → **CloudTrail** (audit), not CloudWatch
- **Full configuration drift/compliance of resources** → **AWS Config**
- **Deep per-request distributed tracing** → **X-Ray** (CloudWatch tracing links via X-Ray)
- **Rich Prometheus visualizations/high-cardinality** → Amazon Managed Prometheus/Grafana (AMG) instead of pure CloudWatch
- **Log management with SIEM-grade retention/analytics at low cost** → forward logs to S3/OpenSearch/3rd-party (CloudWatch Logs is per-GB pricey at scale)
- **Millisecond-granularity custom metrics in huge volumes** → high-res metrics cost; push into a time-series store
- **Real-time stream ingestion/analytics** → Kinesis (CloudWatch is observability, not an ingest pipeline for app data)
- **Monitoring your application UI/UX from real users** → CloudWatch RUM/third-party frontend tools
- **Network packet-level capture** → VPC Flow Logs (though Flow Logs can land in CloudWatch Logs)

## 5. Important features

- **Alarm types**: **static threshold**, **anomaly detection** (ML baseline — great for cyclical workloads), **composite** (AND/OR of several alarms — reduce noise), **metric math** (derived metrics like error-rate %)
- **`datapoints_to_alarm` vs `evaluation_periods`** — be tolerant to blips (3-of-5) or strict (all)
- **"Treat missing data as breaching"** — if the app is fully down, metrics stop; alarm still fires (classic gotcha)
- **Logs** — subscription filters (real-time to Lambda/Kinesis/Firehose/OpenSearch), metric filters (count patterns like `AccessDenied`), **Logs Insights** (SQL-ish, per GB scanned), **live tail**, retention policies (default: never expire — set them!)
- **Unified agent** — metrics + logs, memory/disk/process, **statsd & collectd** support, JSON config, IAM `CloudWatchAgentServerPolicy`
- **Container Insights** — ECS/EKS/Fargate/ROSA; **enhanced observability** (curated dashboards, cluster→container drill-down); **ECS health metric** `UnHealthyContainerHealthStatus` (0/1) for alarms; **OTel Container Insights** (preview) for EKS — PromQL in Query Studio, GPU/Trainium/EFA detection; billed **per observation** in enhanced mode
- **Synthetics** — **canaries** = Lambda + Puppeteer; schedule-driven endpoint/widget tests; screenshots, HAR, metrics
- **Contributor Insights** — real-time **top-N** contributors from logs (top IPs, error codes, API paths)
- **Application Insights** — automated monitoring + ML-based anomaly detection for research workloads, recommended alarms
- **Custom metrics / EMF** — PutMetricData; embedded metric format extracts metrics from structured JSON logs automatically
- **Metric streams** — export metrics in near real time to **S3 / Datadog / Splunk / HTTP endpoints** (via Firehose); cross-account via roles
- **Cross-account observability** — search/investigate across accounts and regions (central monitoring account)
- **ServiceLens** — combine metrics, logs, traces (X-Ray) in one console

## 6. Limitations

- **Default EC2 metrics lack memory/disk/swap** — must install the unified agent
- **Metrics resolution** — standard 60 s; high-res (1 s) costs more; per-minute granularity for many services
- **Logs cost** — ingestion **$0.50/GB**, storage **$0.03/GB/month**; default retention is *never expire* → runaway cost if unfettered (set retention!)
- **Dashboards cost** — **$3/dashboard/month**; build only what you need
- **Permissions** — agent needs IAM policy to push; cross-account streams need roles
- **Alarms are periodic** — cannot react *inside* a period; latency up to a few minutes
- **Not a full APM** — needs X-Ray for distributed tracing and AMG/OTel for high-cardinality Prom metrics
- **No packet capture** — network-level monitoring via VPC Flow Logs / Traffic Mirroring instead
- **Querying scans** — Logs Insights billed per GB scanned; broad/time-span-heavy queries get pricey
- **Region scoping** — metrics/streams are regional; build cross-account/cross-region plumbing deliberately
- **Signals arrive async** — ~1–2 min typical; not sub-second guaranteed for alarms
- **Custom metric cardinality limits** — dimensions limit (10 per metric); huge cardinality breaks metric models (EMF/metric streams mitigate)

## 7. Trade-offs

- **CloudWatch vs CloudTrail** — operational metrics/logs *now* vs **audit trail** of *who did what* (two different jobs; often feed each other)
- **CloudWatch vs X-Ray** — aggregate metrics/logs vs per-request **distributed traces** (use together)
- **CloudWatch vs AWS Config** — real-time performance/health vs **configuration drift & compliance**
- **CloudWatch Logs vs S3/OpenSearch/SIEM** — easy real-time ingestion & Insights vs cheaper long-term retention + powerful analytics/newer tooling
- **Static vs anomaly alarms** — fixed threshold simplicity vs ML baseline for cyclical/spiky workloads
- **Container Insights original vs enhanced** — per-metric pricing vs **per-observation** flat pricing + richer dashboards + drill-down (recommended)
- **60 s vs 1 s resolution** — cost vs catching sub-minute spikes
- **Aggregated Dashboards vs per-service consoles** — one-pane troubleshooting vs simplicity
- **CloudWatch only vs OpenTelemetry/Prometheus/Grafana** — managed native (less setup) vs open-source ecosystem/flexibility at scale

## 8. Architecture

Reference operational patterns:

```
Alerting pipeline:
  App/EC2/ECS → CloudWatch metric → anomaly/composite alarm → SNS → email + Lambda remediation
  (missing-data-as-breaching catches full downtime)

Log pipeline:
  EC2/on-prem unified agent → CW Logs → metric filter (count errors) → alarm
                                          → subscription filter → Lambda / Firehose→S3 / OpenSearch
                                          → Logs Insights for ad-hoc SQL

Container health:
  ECS cluster (enhanced Container Insights) → UnHealthyContainerHealthStatus → alarm → restart task

Synthetic + business:
  Canary → endpoint availability → alarm
  App logs (EMF) → custom metrics → dashboards; Metric Stream → Datadog/Splunk/S3 for SIEM
```

## 9. SAA-C03 Perspective

CloudWatch is a **High-Performing + Resilient + Secure** staple (Domains 1–4):

- **"Monitor & alert on resource health / react to thresholds"** → **CloudWatch alarms → SNS**
- **"Track memory/disk on EC2"** → **unified CloudWatch agent** (metrics NOT included by default — top gotcha)
- **"Alarm even when app is fully down"** → treat **missing data as breaching**
- **"Reduce noisy paging"** → **composite alarms** / anomaly detection
- **"Cyclical workload baselines"** → **anomaly detection alarms**
- **"Query logs to troubleshoot"** → **Logs Insights**; real-time routing = **subscription filters**; logs→metrics = **metric filters**
- **"Container monitoring"** → **Container Insights with enhanced observability**
- **"Test endpoint uptime incl. login flow"** → **Synthetics canaries**
- **"Export metrics to 3rd-party/SIEM"** → **metric streams → Firehose**
- **Autoscaling often scales on CloudWatch alarms** — memory/CPU metrics feeding ASG policies
- Cost: set **log retention**, watch ingestion, dashboard count
- **vs CloudTrail** (audit) — CloudWatch = observability, CloudTrail = accountability

Exam traps: "EC2 default metrics include memory" → **false, install agent**; "CloudWatch Logs free forever" → **no, set retention**; "missing data" handling; "3 of 5 datapoints"; "dashboards cost $3/mo"; "Container Insights metrics are custom metrics (charge)"; "canary = Lambda + Puppeteer".