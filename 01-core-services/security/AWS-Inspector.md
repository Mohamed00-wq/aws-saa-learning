# Amazon Inspector Course — Automated Vulnerability & EC2/Container/Lambda Scanning

## 1. Purpose

Amazon Inspector is an **automated vulnerability management service** that **continuously scans your workloads for software vulnerabilities and unintended network exposure**, delivering a prioritized list of findings with **risk scores** for remediation. For SAA it's the answer to **"automated vulnerability scanning of EC2 instances, container images in ECR, and Lambda functions (packages + code) with SBOM export"**. Its value prop: agentless options, continuous coverage, and integration with Security Hub/EventBridge for automated response.

## 2. How it works

- **Scan targets & modes:**
  - **Amazon EC2** — **agentless scanning via EBS snapshots** (recommended; near real-time) and/or **agent-based** via the SSM agent; covers OS & package vulnerabilities (CVEs), configuration-related findings, **network reachability** (internet-exposed ports/services and their exposure)
  - **Amazon ECR container images** — **enhanced scanning**: on-*push*, on-scan, and continuous (nightly re-scan of existing images up to 30 days old); OS packages only; **ECR console shows findings too**
  - **AWS Lambda** — **standard scanning** (package/OS dependencies in function deployment packages) and **code scanning** (application-level: injection flaws, weak crypto, etc. via Amazon Q built-in detectors based on CWE)
  - **Beyond AWS** — CI/CD tooling; map exported by CWEs to **SBOM**
- **Vulnerability intelligence** — combines 50+ sources (public CVEs incl. **2023+**, ISA/IAR, NVD/OSV, per-distro advisories) and **zero-day notification rules** to flag high-risk items early; **contextual risk scores** rank issues by likelihood & impact in *your* environment (reachability, whether an EC2 instance is internet-facing, use in Lambda etc.)
- **SBOM Generator** — produces **Software Bill of Materials** (SPDX) for EC2/ECR/Lambda to S3 (supply-chain transparency)
- **Outputs** — findings to the Inspector console, **EventBridge**, **AWS Security Hub**; raw findings to S3; CIS benchmarks-style checks as applicable
- **Pricing/model** — **15-day free trial** of all features; then paid per-scanned-account/resource-hours; you opt in (disabled by default)

```
EC2 (agentless EBS snapshot or SSM agent) ─┐
ECR images (push/scan/continuous) ──────────► Inspector (50+ intel sources, risk scoring)
Lambda (package scan + code scan) ──────────┘   └─► findings: console / EventBridge / Security Hub / S3
  + SBOM Generator (SPDX) → S3 — supply chain transparency
  + network reachability findings (internet exposure)
```

## 3. When to use

- **Continuously scan EC2 for CVEs, misconfigurations, and network exposure** (visibility + posture)
- **Scan container images on push and re-scan continuously** (ECR enhanced scanning) — shift-left image hygiene before deployment
- **Scan Lambda dependencies and application code** for vulnerabilities before/along the release
- **Get an SBOM** (SPDX) of your workloads for supply-chain/audit needs
- **Automate response** — EventBridge on critical findings → quarantine instance / block port / page on-call with Security Hub as the single findings dashboard

## 4. When NOT to use

- **Detecting threats/account compromise** (phishing, bad actors, runtime detections) → GuardDuty / Detective
- **Sensitive data discovery in **S3** or elsewhere** → Macie (Inspector is about *vulnerabilities*, not data classification)
- **Compliance evidence at the organizational scale** → AWS Audit Manager / Config (Inspector findings can feed these, but it's not an audit product)
- **Static app-code analysis for custom, non-Lambda apps** → dedicated SAST tools (Inspector Lambda code scanning covers Lambda)

## 5. Important features

- **EC2 agentless scanning** (EBS snapshots; no agent footprint) + agent-based option, **network reachability** findings
- **ECR enhanced scanning** — on-push, on-scan, continuous nightly re-scans (≤30-day images), findings mirrored in ECR console
- **Lambda scanning** — standard (package/OS deps) + **code scanning** (Amazon Q detectors, CWE-mapped, injection/weak-crypto, etc.)
- **SBOM Generator** — SPDX export to S3 for EC2/ECR/Lambda
- **50+ vulnerability-intelligence sources incl. 2023+ CVEs, per-distro advisories, zero-day notification rules**
- **Contextual risk scoring** (reachability + exposure) to prioritize; **15-day free trial**; EventBridge + Security Hub + S3 integration

## 6. Limitations

- **OS/packages first-class; not deep app-context for arbitrary frameworks** (Lambda code scan covers specific detector classes)
- **ECR continuous re-scan** limited to images scanned within the last 30 days (older need manual/higher-tier handling)
- **Network reachability** limited to scenarios configurable/covered (agentless scope); some checks need the SSM agent or agentless snapshot permissions
- Not a threat-detection service (no behavioral detections) — pair with **GuardDuty** for that
- **Paid after 15-day trial**; per-account/per-scan-hour cost to manage across large fleets

## 7. Trade-offs

- **Inspector vs GuardDuty** — vulnerability *static/package/code* scanning & exposure (Inspector) vs *threat detection* on live activity (GuardDuty); complementary — both funnel findings to Security Hub
- **Agentless vs agent-based** — zero footprint, always-on, snapshot cost (agentless) vs deeper/faster on active instances (agent-based)
- **vs Macie** — workloads' vulnerabilities (Inspector) vs sensitive data-at-rest discovery (Macie)
- **vs WiSC / third-party scanners (Tenable, Qualys/Prisma Cloud, Snyk)** — native AWS, integrated, org-wide event-driven (Inspector) vs deep third-party feature sets & market integrations (bring-your-own where required)

## 8. Architecture

```
Secure pipeline:
  Pipeline: push image → ECR enhanced scan (Inspector on-push) → threshold gate → deploy
  Fleet: EC2 agentless scan → Inspector → critical finding → EventBridge → Lamabda auto-actions
    (patch via SSM Patch Manager, quarantine SG, notify) → Security Hub consolidated view
  Audit: SBOM Generator → SPDX → S3 for compliance
```

## 9. SAA-C03 Perspective

- **"Automatically scan EC2 instances for vulnerabilities"** → **Amazon Inspector**
- **"Scan container images in ECR (on push/continuous) for CVEs"** → **Amazon Inspector (ECR enhanced scanning)**
- **"Scan Lambda functions' packages and code for vulnerabilities"** → Inspector (Lambda scanning)
- **"Network reachability / which ports are exposed to the internet"** → Inspector
- **"Generate an SBOM for supply-chain compliance"** → Inspector SBOM Generator
- **"Detect threats/anomalous activity"** → **GuardDuty**; "sensitive data in S3" → **Macie**; "unify findings" → **Security Hub**

Exam traps: "Inspector detects a compromise/intrusion" → **no, it's vulnerability scanning (static) — GuardDuty handles active threats**; "Inspector scans non-AWS/arbitrary apps deeply" → **primarily EC2/ECR/Lambda with product scopes**; "it's free always" → **15-day trial then paid**; "Inspector replaces Security Hub" → **it's a source that feeds Security Hub**.