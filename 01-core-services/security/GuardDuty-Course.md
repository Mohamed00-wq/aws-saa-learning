# GuardDuty Course — Amazon GuardDuty

## 1. Purpose

GuardDuty is a **continuous threat detection** service that monitors your AWS accounts, workloads, and data for malicious or suspicious activity — account compromise, instance/container compromise, reconnaissance, and S3 bucket compromise. It feeds findings into Security Hub and EventBridge for alerting and automated response. Detection-only: it finds threats; you decide what to block.

## 2. How it works

- Ingests and analyzes telemetry streams with **ML + threat intelligence**:
  - **CloudTrail management events** (user/API activity) and **CloudTrail S3 data events**
  - **VPC Flow Logs** (network traffic) and **Route 53 Resolver DNS query logs**
  - **EKS audit logs** (EKS Protection), **runtime events** from an installed GuardDuty agent (Runtime Monitoring for EC2/ECS/EKS) — not available on EKS Hybrid Nodes; EKS Auto Mode integrates
  - Malware Protection for **S3** (scan uploaded objects), **EC2/EBS**, and **ECS**
- Produces **findings**: severity-rated (Low–Critical) with finding type, affected resource, indicators.
- **Extended Threat Detection**: correlates multi-stage attack sequences across data sources into one **Critical-severity finding** (`AttackSequence:…`) for EKS/ECS/EC2 instance groups, IAM credential misuse, and S3 data compromise.
- Findings stream to **Amazon EventBridge**, **Security Hub**, and can be exported to **S3**; use **SNS/Lambda** for real-time alerting.

## 3. When to use

- Baseline **continuous threat detection** for accounts, EC2, containers (EKS/ECS), Lambda, RDS, and S3.
- Compliance programs that require continuous monitoring + documented detections.
- **Multi-account organizations** — enable once with delegated admin, auto-enroll accounts.
- Combined with **Security Hub** to centralize findings and **Inspector** for vulnerability scanning.
- Malware detection without running your own AV (S3 objects, EC2/ECS workloads).

## 4. When NOT to use

- It **detects, doesn't prevent** — not a substitute for WAF/SG/NACL/Network Firewall or for patching.
- Real-time packet-level forensics or raw traffic inspection (no access to full packet payloads).
- Bare-metal packet capture/malware on non-standard setups; runtime monitoring requires agent compatibility.
- Small, single-account hobby use where cost of data-source fees > value (still often worth enabling).

## 5. Important features

- **Data sources**: CloudTrail (mgmt + S3 data events), VPC Flow Logs, DNS query logs, EKS audit logs, runtime agent events, malware scans.
- **Protection plans**: EC2, S3, EKS, ECS, RDS, Lambda + **Malware Protection** + **Runtime Monitoring**.
- **Extended Threat Detection** (attack sequence findings) — ML/AI-linked multi-stage attacks.
- **Findings management**: severity, suppression rules, trusted IP lists (optional), filter/export, GC auto-analysis.
- **Multi-account**: delegated administrator via Organizations; centralized view.
- **Integrations**: Security Hub (enriched exposure findings), EventBridge (automation: SNS, Lambda, SSM remediation), S3 export of findings.
- 2026: available as a capability of the **enhanced Security Hub**; findings auto-enriched with context.

## 6. Limitations

- **Detection-only** — no blocking/mitigation built in; you wire response actions.
- **Pricing** scales with monitoring volume (data sources, malware-scan GB, Runtime Monitoring vCPU-hours) and the optional 30-day trial then per-feature fees.
- Findings can have **delayed/aggregate** detection windows; not sub-second.
- **Runtime Monitoring not available** for EKS Hybrid Nodes.
- Doesn't scan assets outside AWS (on-prem only via coupled services, e.g. outposts coverage is limited).

## 7. Trade-offs

- **GuardDuty vs manual CloudTrail+Flow Log analytics**: turnkey ML + threat intel vs DIY rules and ETL — GuardDuty wins on time-to-deploy and coverage.
- **GuardDuty vs third-party SIEM/EDR (Splunk, CrowdStrike, Wiz)**: AWS-native, cheap, integrated findings vs richer multi-cloud/incident response; many export GuardDuty findings to a SIEM anyway.
- **Foundational vs Runtime Monitoring**: broader signal vs deeper agent-based process/file/network telemetry (more coverage but operational overhead + agent).
- **S3 malware scanning cost vs risk**: per-GB scan fees vs protection from malicious uploads/downloads.

## 8. Architecture

```
Member accounts (CloudTrail, Flow Logs, DNS, S3 data events, agents)
        │
        ▼
GuardDuty ── delegated admin / aggregation ──► findings
        │
        ├──► Amazon EventBridge ──► SNS / Lambda / Step Functions (auto-remediation)
        ├──► AWS Security Hub (centralized + exposure correlation)
        └──► S3 export (archive/long-term, Athena queries)
```

- Enable with delegated admin in Organizations; enable Runtime Monitoring + Malware Protection on critical workloads.
- Route high-severity findings to an incident-response Lambda (isolate instance, revoke key, snapshot).

## 9. SAA-C03 Perspective

- "Continuously monitor accounts/instances/buckets for suspicious activity" → **GuardDuty** (ML + threat intel findings).
- Detection for **EC2/S3/EKS/ECS/Lambda/RDS + containers → EKS/ECS audit + runtime**.
- **GuardDuty vs Inspector**: GuardDuty = *behavioral threat detection* (CloudTrail/Flow/DNS/runtime); Inspector = *vulnerability scanning* (CVEs in image/package). Know the split.
- Findings → **EventBridge/Security Hub/SNS** for response; guardrails via IAM/SCP.
- GuardDuty + Security Hub + Inspector are the "security automations" trio on the exam.

Exam trap: a question about *detecting* suspicious S3/EC2 activity — GuardDuty; about *finding known CVEs/package vulnerabilities* — Inspector.