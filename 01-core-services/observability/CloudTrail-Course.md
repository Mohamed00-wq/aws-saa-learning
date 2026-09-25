# CloudTrail Course  API Auditing & Governance (Audit 🔎)

## 1. Purpose

CloudTrail is AWS's **audit log**: it records **every API call** in your account  console clicks, CLI, SDK, and service-to-service calls  with *who, what, when, from where, and the result*. It answers **"who changed what, and when?"** and is the foundation of security investigations, compliance, governance, and incident response. For SAA, any scenario involving *auditing, compliance, governance, or "who did that"* points to CloudTrail.

## 2. How it works

- Every action in your account generates an **event** (JSON) captured by CloudTrail
- **Event types**:
  | Type | What it captures (plane) | Default |
  |---|---|---|
  | **Management events** | Control plane  `RunInstances`, `CreateUser`, `DeleteBucket` | **Enabled (free, 90-day Event history)** |
  | **Data events** | Data plane  S3 `GetObject`/`DeleteObject`, Lambda `Invoke` | **Opt-in (high volume, paid)** |
  | **Network activity events** | VPC endpoint traffic metadata | Opt-in |
  | **Insights events** | Unusual API patterns (rate/error spikes) | Opt-in |
- **Two consumption layers**:
  - **Event history**  free, **90 days**, console/CLI searchable no setup (auto-on)
  - **Trails**  deliver events to **S3** (+ CloudWatch Logs/EventBridge) **persistent**, configurable (management, data, Insights), cross-region/org
- ~**15-min async delivery** to S3 **EventBridge** give ~15s real-time-ish, **CloudWatch Logs** streams for real-time alerting

```
Console / CLI / SDK / services → CloudTrail events
   ├─ Event history (free, 90 days, searchable)
   └─ Trail → S3 (SSE-S3/KMS, log validation digest) ─┬─ CloudWatch Logs → metric filter → alarm → SNS
                                                     ├─ EventBridge → Lambda remediation
                                                     └─ Athena/SIEM analysis of S3 files
   Insights → ML-detected anomalies → S3/EventBridge
```

## 3. When to use

- **Security investigation**  "who deleted the bucket / changed the SG / got denied?"
- **Compliance & governance**  evidence of control-plane activity for auditors (PCI, SOC, etc.)
- **Failed-access analysis**  detect brute force / anomalous `ConsoleLogin` / denied API attempts
- **Change management**  correlate resource creation/deletion with who did it
- **Real-time alerting**  stream to CloudWatch Logs → metric filters → alarms (e.g., 5 failed logins in 5 min) or EventBridge → auto-remediate
- **Multi-account governance**  **Organization trail** from the management account covers **all accounts** (members can't disable)
- **Data plane security**  audit S3 object access / Lambda invocations via **data events** (opt-in)
- **Long-term retention + analytics**  trails → S3 (lifecycle to Glacier) → Athena/SIEM

## 4. When NOT to use

- **Operational metrics / performance monitoring** → **CloudWatch**
- **Configuration drift / resource state compliance** → **AWS Config**
- **Real-time application logs / user behavior in your app** → CloudWatch Logs (not CloudTrail)
- **Millisecond forensic replay of network traffic** → **VPC Flow Logs** (CloudTrail doesn't capture packets)
- **Monitoring every data event always-on** → cost explodes **enable selectively** (advanced event selectors, thresholds, buckets of interest)
- **Queryable long-term audit for new deployments (post May 2026)** → **CloudTrail Lake is closed to new customers** use **trails → S3 + Athena/SIEM** or CloudWatch instead
- **Application trace (per-request timing)** → **X-Ray**

## 5. Important features

- **Management events**  on **by default** read-only or write-only filter covers IAM, EC2, S3 control plane, ConsoleLogin and other non-API events free 90-day Event history
- **Data events**  opt-in S3 object-level, Lambda invoke, DynamoDB (some), Managed Blockchains use **advanced event selectors** to capture only what matters (limit cost) ~$0.10/100k
- **Insights events**  ML baselines detect **unusual API volume/error-rate spikes** (e.g., 10x login spike) ~$0.35/100k analyzed view 90 days or store to Lake/event stores
- **Trails**  **single-region** or **multi-region** (all Regions → one bucket  best practice) **organization trail** (management account, applies to all member accounts, members can't modify/delete)
- **Log file integrity validation**  **SHA-256 digest files** alongside logs `aws cloudtrail validate-logs` detects tampering/modification/deletion post-delivery
- **Encryption**  **SSE-S3** default (free) or **SSE-KMS** (customer-managed, per-API cost)
- **CloudWatch Logs integration**  stream trail events → metric filters → alarms → SNS also **EventBridge** (near-real-time, ~15s) for automated responses
- **CloudTrail Lake** (existing customers only)  columnar (**ORC**) event data stores, **SQL queries**, org-wide aggregation, immutable read-only stores, up to **7 years** retention config/Audit Manager/non-AWS data via channels **NOTE: closed to new customers since May 31, 2026**
- **Aggregated data events**  summaries of access patterns for filtering focus
- **Delivery options**  S3 prefix, SSE, lifecycle rules (manage cost) SNS notification on delivery

## 6. Limitations

- **Web event history is 90 days**  beyond that you need a **trail → S3** (trails don't **retroactively** capture create before the incident!)
- **Management events are free data events cost**  opt-in and selective to control spend
- **~15 min async S3 delivery**  use CloudWatch Logs/EventBridge for near-real-time reaction
- **Global services**  some events (IAM, S3, Route 53) are **global**, logged in **us-east-1** (design the trail accordingly)
- **Not packet capture**  no payload/network data (VPC Flow Logs for that)
- **CloudTrail Lake closed to new customers (May 31, 2026)**  plan on trails → S3 + Athena/CloudWatch for new workloads
- **No KMS-key permission leaks**  bucket/key policies must allow CloudTrail writes (misconfig = silent delivery drop)
- **Org pill management account only**  member accounts can't create org trails
- **Insights adds cost**  enable on sensitive/important trails only
- **Event size/frequency**  noisy accounts generate massive S3 objects lifecycle + partitioning matter

## 7. Trade-offs

- **CloudTrail vs CloudWatch**  *who did what (audit)* vs *how is it performing (observability)* complementary
- **CloudTrail vs AWS Config**  one-off events vs **continuous resource state/drift** (Config records configuration changes over time, CloudTrail records each API call)
- **CloudTrail vs VPC Flow Logs**  API-level audit vs **network-level** traffic metadata
- **Event history vs Trail**  free 90-day browse vs persistent configurable S3 delivery
- **Management vs Data events**  free by-default control-plane vs **expensive opt-in** data-plane
- **CloudWatch Logs vs EventBridge delivery**  rich filtering/alarming vs code-less automated responses (~15s)
- **CloudTrail Lake vs S3+Athena**  managed SQL/immutable stores vs **self-serve on your own S3** (Lake closed to new customers  lean S3+Athena)
- **SSE-S3 vs SSE-KMS**  free defaults vs control/audit of key usage (KMS adds per-API cost)
- **Multi-region vs single-region trail**  coverage vs cost/simplicity (multi-region + org = recommended)
- **Data events everywhere vs selective**  complete visibility vs cost use advanced selectors + buckets-of-interest

## 8. Architecture

Reference governance patterns:

```
Best-practice multi-account audit:
  Management account → Organization trail (all accounts, all regions)
     → centralized S3 bucket ├─ SSE-KMS + log-file integrity validation
                              ├─ CloudWatch Logs → metric filters → alarm (5 failed logins/5min) → SNS
                              └─ (optional) Athena queries on S3 / SIEM ingestion

Security investigation:
  Console/CLI event → Event history search (90 days, last-resort trail→S3) → correlate userIdentity,
  sourceIP, requestID (share with support), errorCode='AccessDenied' patterns

Real-time response:
  Trail → EventBridge rule (e.g. aws.iam DeleteUser) → Step Functions/Lambda → remediate + notify

User: X
     │
     │ StopInstances
     ↓
   EC2
     │
     ↓
 CloudTrail
     │
     ├── Who?      X
     ├── What?     StopInstances
     ├── Which?    EC2 instance i-123456
     └── When?     10:35:21


```

## 9. SAA-C03 Perspective

CloudTrail is a **Secure (30%) and governance** staple (Domains 1, 3):

- **"Audit how AWS APIs are being used / who made changes"** → **CloudTrail**
- **"Compliance / governance evidence"** → **trails → S3**
- **"Monitor failed logins / anomalous API activity"** → **CloudWatch Logs metric filters + Insights**
- **"Track S3 object access / Lambda invocations"** → **data events (opt-in, advanced selectors)**
- **"All regions, all accounts logging"** → **multi-region + organization trail**
- **"Detect tampering of logs"** → **log file integrity validation (SHA-256)**
- **"Long-term searchable audit"** → trails → **S3 + Athena** (and CloudTrail Lake for existing customers)
- **Management events default (free, 90 days)** **data events not default and cost money**
- **ConsoleLogin is a management event** global services log to **us-east-1**
- **~15 min delivery**  CloudWatch/EventBridge for near-real-time

Exam traps: "CloudTrail catches everything out of the box" → **data events are opt-in** "retroactive capture" → **create trail early** "tamper detection = encryption" → **it's integrity (digests), not confidentiality** "CloudTrail Lake for new accounts (2026+)": **closed to new customers May 31, 2026  use trails + S3 + Athena** "CloudTrail vs AWS Config"  state vs action.