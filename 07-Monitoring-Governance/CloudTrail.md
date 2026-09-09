# CloudTrail — API Auditing & Governance

## What it is

CloudTrail records **every API call** in your account — console, CLI, SDK, CloudFormation. Every `CreateBucket`, `RunInstances`, `PutObject` is captured with who, what, when, from where, and the outcome. For the SAA exam, it appears in any scenario about compliance, auditing, security investigation, or governance.

## Event types

| Type | What it captures | Default |
|---|---|---|
| **Management events** | Control plane (CreateUser, RunInstances, DeleteBucket) | ✅ Enabled |
| **Data events** | Data plane (S3 GetObject, Lambda Invoke) | ❌ Opt-in, high volume |

## Event history

- **Free**, last 90 days, in the CloudTrail console. Searchable by time, event name, user, resource.
- No configuration needed — every account records management events automatically. Beyond 90 days requires a **trail**.

## Trails

- **Single-region trail**: delivers to S3 in one region.
- **Multi-region trail**: delivers events from **all regions** to one S3 bucket. **Best practice**.
- **Organization trail**: created in management account, applies to **all accounts** in the organization.

### Trail configuration

| Setting | Options |
|---|---|
| **S3 bucket** | Destination for log files |
| **S3 prefix** | Organize by account/region |
| **SSE-KMS** | Customer-managed key encryption |
| **Log file validation** | SHA-256 hashing + digest files for tamper detection |
| **CloudWatch Logs** | Stream events to CW Logs for real-time alerting |

## CloudTrail Insights & Lake

- **Insights**: detects unusual spikes/dips in API call volume (e.g. 10x ConsoleLogin spike).
- **Lake**: SQL-based querying of CloudTrail events, up to **7 years** retention. Organization-wide event data stores.

## Log file integrity validation

- Creates **digest files** with SHA-256 hashes alongside log files.
- Verify with `aws cloudtrail validate-logs` — detects modification or deletion after delivery.

## CloudWatch Logs integration

- Stream events to a **CloudWatch Logs log group**. Create **metric filters** to count specific API calls.
- Create **CloudWatch Alarms** on metrics → SNS notification. Example: alarm on 5+ failed console logins in 5 min.

## Multi-account strategy

- **Organization trail**: management account creates, members can't disable.
- **Centralized S3 bucket**: all trails deliver to one bucket. Bucket policy allows cross-account writes.
- **Best practice**: org trail + centralized S3 + CloudWatch Logs streaming.

## Encryption

| Option | What |
|---|---|
| **SSE-S3** | Default, AWS-managed, no extra cost |
| **SSE-KMS** | Customer-managed key, adds API cost per encrypt/decrypt |

## Pricing

| Component | Cost |
|---|---|
| Management events (first trail) | Free |
| Data events (S3/Lambda) | $0.10/100k events |
| CloudTrail Insights | $0.35/100k events analyzed |
| CloudTrail Lake | $0.50/GB ingested |

## Exam domains

- [x] **Secure (30%)** — audit logging, log integrity, KMS encryption, org trails
- [x] **Resilient (26%)** — multi-region trails, centralized S3, CloudTrail Lake
- [x] **High-Performing (24%)** — CloudWatch Logs for alerting, Insights
- [x] **Cost-Optimized (20%)** — data event filtering, S3 lifecycle policies

## Key gotchas

1. **Management events enabled by default** — 90-day history without a trail
2. **Trails don't retroactively capture events** — create before the incident
3. **Data events NOT enabled by default** — must opt in, expensive at scale
4. **Log file integrity ≠ encryption** — tamper detection vs confidentiality
5. **Org trails only by management account**
6. **Log delivery is async** — ~15 min delay; CW Logs for real-time

## Related services

- [[CloudWatch]] — metric filters, alarms, real-time alerting
