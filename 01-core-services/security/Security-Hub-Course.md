# Security Hub Course  AWS Security Hub

## 1. Purpose

Security Hub gives you a **single dashboard for your entire security posture** across all accounts and regions: aggregated findings, security best-practice checks (controls), compliance standards, and automated response. In 2026 it is split into two complementary services:

- **Security Hub (CSPM)**  continuous posture checks against your environment + security-relevant config ("posture").
- **Security Hub**  a unified experience that **correlates** findings from CSPM, GuardDuty, Inspector, Macie, and partners into **exposure findings**, with an **attack-path graph** and automated workflows.

## 2. How it works

- **Enable** per region (or org-wide via delegated administrator + Organizations), then:
  - **CSPM checks** evaluate resources against **standards**  AWS Foundational Security Best Practices (FSBP), CIS AWS Foundations, PCI DSS, NIST, AWS Resource Tagging  backed by AWS Config rules + custom rules. Each check = a **control** with Passed/Failed/Unknown, feeding **security scores** per standard.
  - **Aggregation**: findings, controls, and scores are aggregated from linked regions into an **aggregation region** ("single pane of glass").
- Any integrated service (GuardDuty, Inspector, Macie, IAM Access Analyzer, Firewall Manager, partners) **imports findings** in a standardized format (**ASFF/OCSF**) into Security Hub.
- Security Hub **correlates** imported findings to create **exposure findings** + attack path graphs exposes **unused access** analysis and least-privilege policy recommendations.
- **Automation rules** and **custom actions** route findings to EventBridge (ticketing: Jira/ServiceNow, SNS, Lambda, remediation).
- Findings are retained **90 days (active)** / **30 days (archived)** after last update.

## 3. When to use

- **Multi-account / multi-region** visibility of security state and compliance scores in one place.
- Compliance reporting (PCI DSS, CIS, NIST, SOC) with out-of-the-box standards + score tracking.
- Centralized **continuous controls monitoring** (CSPM)  "is my environment configured securely?"
- Centralizing GuardDuty + Inspector + Macie findings and automating response (MTTR).
- **Unused access / least-privilege** analysis at scale.

## 4. When NOT to use

- **Threat detection** itself  that's GuardDuty (Security Hub aggregates, doesn't detect network threats).
- **Vulnerability scanning**  that's Inspector (Security Hub ingests the findings).
- Small single-account, single-region setups where a few dashboards suffice (still optional costs stop), but it scales best at org level.
- Real-time blocking/IPS  Security Hub is detection/posture, not enforcement (automation rules trigger actions elsewhere).

## 5. Important features

- **Standards & controls**: FSBP, CIS, PCI DSS, NIST, tagging per-control status + **security scores** (updated within ~24h).
- **Cross-region aggregation** to an aggregation region one delegated admin (not admin+member simultaneously).
- **Exposure findings + attack-path graph**: sees how an attacker could chain resources.
- OCSF/ASFF normalized finding format 60+ AWS + partner integrations, **EventBridge** for automation.
- **Custom actions** → ticketing/chat/SOAR (Jira, ServiceNow, Slack, PagerDuty…).
- **Unused access**: finds roles/users/keys/perms unused in a 90-day lookback suggests scoped-down policies.
- Sample control findings, workflow statuses, suppression, and wizard for tour/onboarding.

## 6. Limitations

- **Finding retention**: active 90 days / archived 30 days  configure export (S3) for long-term audit.
- **Regional** service  aggregation region setup is mandatory for a global view.
- **Cost** based on number of controls + findings ingested (not free beyond trial).
- Backed by **AWS Config** (implied costs for Config rules in CSPM).
- Detection breadth = what integrations you enable it **won't detect** what isn't wired in.
- Score recomputation lag (≈24h) and scores reset when the aggregation region changes.

## 7. Trade-offs

- **Security Hub vs CloudWatch/EventBridge DIY**: pre-built normalization + standards vs free-form rules Security Hub centralizes cross-account signals others can't easily.
- **Security Hub vs third-party CSPM (Wiz, Prisma, CrowdStrike)**: AWS-native, cheap, integrated with ASFF/OCSF vs multi-cloud + broader context Security Hub can still *export* findings to them.
- **CSPM only vs full Security Hub**: posture checks only vs unified exposure/detection correlation  AWS recommends enabling both.
- **Standards breadth vs cost**: enable only standards you must report on (each adds controls/findings).

## 8. Architecture

```
Accounts × Regions ──► Security Hub (delegated admin aggregator region)
     ▲                        │
     │ findings (ASFF/OCSF)   ├─► dashboards: security scores, exposure, attack-path graph
 GuardDuty / Inspector /      ├─► automation rules / custom actions
 Macie / IAM Access Analyzer  ├─► EventBridge ──► SNS / Lambda / Step Functions
 / Firewall Manager / partners└─► (optional) S3 export for long-term retention
     ▲
     │ controls  ◄── AWS Config (FSBP, CIS, PCI DSS, NIST)
```

- Enable with delegated admin + aggregation region wire EventBridge → ticketing/Lambda remediation.
- Turn on GuardDuty, Inspector, Macie alongside for exposure correlation.

## 9. SAA-C03 Perspective

- "Single place to view/aggregate security findings across accounts & regions" → **Security Hub**.
- "Security best-practice compliance checks / standards" → Security Hub **CSPM** (controls, scores).
- Know findings flow: GuardDuty/Inspector/Macie → Security Hub → EventBridge → response automation.
- **GuardDuty vs Security Hub vs Inspector** split is heavily tested  detection vs aggregation vs vulnerability scan.
- Automation rules + custom actions → **EventBridge** (ticketing, auto-remediation).

Exam trap: "which service detects a brute-force from an unfamiliar IP?" → GuardDuty "where do you consolidate that finding and check compliance?" → Security Hub.