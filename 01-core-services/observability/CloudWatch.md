# CloudWatch — Monitoring & Observability

## What it is

CloudWatch is AWS's native monitoring and observability service. It collects metrics, logs, and traces from every AWS service and custom applications, then lets you visualize, alarm, and react to changes. For the SAA exam, CloudWatch appears in nearly every scenario involving monitoring, alerting, troubleshooting, or operational visibility.

## Metrics

- **Namespaced**: every metric belongs to a namespace (e.g. `AWS/EC2`, `AWS/Lambda`). Custom metrics use your own namespace.
- **Dimensions**: key-value pairs that filter/group metrics (e.g. `InstanceId=i-0abc`).
- **Resolution**: standard (60 s) or **high-resolution** (1 s). High-res costs more but catches sub-minute spikes.
- **Statistics**: Average, Sum, Min, Max, SampleCount, p99, p95, etc.

### Custom Metrics

- Publish via **PutMetricData** API or use **EMF (Embedded Metric Format)** — structured JSON in logs, CloudWatch extracts metrics automatically.
- Metric streams export metrics to **S3**, **Datadog**, **Splunk**, or any HTTP endpoint via Kinesis Data Firehose.

## Alarms

- Watch a metric over time → trigger action when threshold breached.
- States: **OK** → **ALARM** → **INSUFFICIENT_DATA** → OK.
- **Datapoints to alarm**: e.g. 3 out of 5 periods → tolerant of single blips.

| Type | How it works | Exam scenario |
|---|---|---|
| **Static threshold** | Fixed number (e.g. CPU > 80%) | Simple alerting |
| **Anomaly detection** | ML baseline → alarm on deviation | Workloads with daily/weekly cycles |
| **Composite** | AND/OR of multiple alarms | Reduce noise — only page when multiple things are bad |
| **Metric math** | Expressions combining metrics (`m1/m2*100`) | Derived metrics like error rate percentage |

- **Treat missing data as breaching** is a common exam trap — if app goes down, metrics stop, alarm fires correctly.

## Logs

- **Log group**: logical container with retention policy. **Log stream**: sequence of events within a group.
- **Subscription filters**: pipe logs to Lambda, Kinesis Firehose, OpenSearch in real time.
- **Metric filters**: create CloudWatch metrics from log patterns (e.g. count `AccessDenied` errors).
- **Logs Insights**: SQL-like query engine — ad-hoc troubleshooting, cost is per GB scanned.
- **Logs live tail**: real-time streaming like `tail -f` in console or CLI.

## CloudWatch Agent

- Unified agent for **metrics** and **logs** from EC2 and on-premises servers.
- Collects **memory, disk, swap, process count** — not available from hypervisor by default.
- Supports **statsd** and **collectd** protocols. Configured via JSON config file.

## Dashboard & Synthetics

- **Dashboard**: custom widgets (metrics, logs, alarms, graphs). Cross-account/cross-region. $3/dashboard/month.
- **Synthetics**: canaries (Lambda + Puppeteer) test APIs/URLs on a schedule. Output: screenshots, HAR, metrics.

## Contributor & Application Insights

- **Contributor Insights**: real-time top-N analysis on logs/metrics (top IPs, error codes, API paths).
- **Application Insights**: automated monitoring for EC2 apps, ML anomaly detection, recommended alarms.

## Pricing

| Component | Cost |
|---|---|
| Metrics (first 10,000) | Free |
| Metrics (above 10,000) | $0.30/metric/month |
| Alarms (first 10) | Free |
| Logs (ingestion) | $0.50/GB |
| Logs (storage) | $0.03/GB/month |
| Dashboards | $3.00/dashboard/month |

## Exam domains

- [x] **Secure (30%)** — Log group IAM policies, KMS encryption, cross-account roles for metric streams
- [x] **Resilient (26%)** — Composite alarms, anomaly detection, missing data handling
- [x] **High-Performing (24%)** — High-resolution metrics, Logs Insights, Container Insights
- [x] **Cost-Optimized (20%)** — Retention policies, metric filtering to reduce ingestion

## Key gotchas

1. **Default EC2 metrics don't include memory or disk space** — install the CloudWatch agent
2. **CloudWatch Logs are not free** — ingestion costs $0.50/GB; set retention to avoid storage creep
3. **Alarms evaluate across periods** — understand `datapoints_to_alarm` vs `evaluation_periods`
4. **Subscription filters** are real-time; **metric filters** create metrics from logs (different purposes)
5. **CloudWatch agent requires IAM permissions** (CloudWatchAgentServerPolicy) to push data
6. **Dashboards cost $3/month each** — build only what you need
7. **Metric streams** go to Kinesis Data Firehose — you pay for Firehose delivery on top

## Related services

- **CloudTrail** — audit log of API calls vs operational metrics
- **EventBridge** — event-driven rules triggered by CloudWatch alarms
- **X-Ray** — distributed tracing for request paths
