# AWS Artifact Course — Compliance Reports & Agreements

## 1. Purpose

AWS Artifact is the **self-service portal for retrieving AWS compliance reports and signing legal agreements**. You download **SOC reports, PCI DSS attestations, ISO certificates, FedRAMP authorizations**, IRAP, C5, CSA STAR, and similar certifications on demand, and sign **BAAs (HIPAA)** or **DPAs (GDPR)** electronically — no emailing AWS, no multi-week turnaround. For the SAA exam it's the answer for **"where do I get AWS compliance evidence / download a SOC report for an auditor / sign a Business Associate Agreement for HIPAA"**.

## 2. How it works

- **Artifact Reports** — browse and download certification/report PDFs; Artifact always serves the **latest version**; periodic updates
- **Artifact Agreements** — presented for **electronic signature**; once signed, stored in Artifact for reference
- **NDA gating** — confidential reports (e.g., **SOC 1 / SOC 2 Type II**) require a **one-time NDA acceptance** before download
- **Access** — AWS Console only (search "Artifact"); IAM permissions (`artifact:*`) control who can use it; each AWS account accesses independently (**no cross-account report sharing / distribution**)

```
Console → Artifact → Reports (SOC/PCI/ISO/FedRAMP/Irap/C5/CSA STAR...) PDF downloads
                 → Agreements (BAA, DPA, addenda) → e-sign → stored for reference
NDA-protected reports → accept NDA once → download freely
Multi-account: per-account access, no cross-account sharing (NDA)
```

## 3. When to use

- **Providing AWS-side compliance evidence to an auditor** — SOC 1/2/3, PCI DSS, ISO, FedRAMP
- **Signing a BAA (HIPAA)** before running PHI workloads on AWS, or a **DPA (GDPR)** for EU/EEA personal data
- **Evaluating AWS compliance posture** for your own pre-work (which certifications cover the services you use)
- **Any "get AWS compliance documentation on demand" question** — no need for AWS Support requests

## 4. When NOT to use

- **Checking your own resource configurations against rules** → **AWS Config**
- **Evidence of *your* systems' security posture / findings** → Security Hub, Inspector, GuardDuty, Audit Manager (Artifact is AWS's own attestations, not your compliance status)
- **Trust center / third-party due-diligence portal** has replaced your download need → that's fine, Artifact is the AWS-native path

## 5. Important features

- **Reports** — SOC 1/2/3, **PCI DSS**, **ISO 27001/27017/27018**, **FedRAMP**, **IRAP**, **MTCS**, **C5**, **CSA STAR**, **Cyber Essentials Plus** (PDF, latest versions)
- **Agreements** — **BAA (HIPAA)**, **DPA (GDPR)**, service addenda; **e-sign directly** in the console
- **BAA covers all HIPAA-eligible services once signed** — sign once, use across Lambda/EC2/RDS/S3 etc.
- **One-time NDA acceptance** unlocks confidential reports
- **Free service** — no charge for reports or agreements
- Some reporting programs tied to **AWS Organizations** for enterprise agreements

## 6. Limitations

- **Console-only** — no API, CLI, or SDK for downloading reports directly (everything goes through the portal)
- **Per-account access** — no cross-account report sharing/distribution (NDA-bound); each account needs its own IAM permissions
- **Artifact only proves AWS's compliance** — it's evidence, not a substitute for your own compliance program
- Reports are updated periodically; always grab **latest version** for audit submissions (older certs may be superseded)

## 7. Trade-offs

- **Artifact vs AWS Config / Security Hub** — AWS's-own compliance *certifications* and *agreements* (Artifact) vs auditing *your* resources (Config) / aggregated findings & standards (Security Hub). You use Artifact alongside them, not instead
- **vs third-party TPRM / GRC platforms** — Artifact is the authoritative AWS source; platforms pull from it. No real alternative inside AWS
- **BAA vs DPA** — know which agreement matches which regulation: BAA = **HIPAA**, DPA = **GDPR**

## 8. Architecture

```
Workload: PHI on Lambda/EC2/RDS/S3 (HIPAA focus)
  AWS Artifact: sign BAA (covers all HIPAA-eligible services) → processing PHI is now compliant
  Auditors need evidence → user downloads SOC 2 Type II + HIPAA eligibility report
  → IAM policy granting artifact:* on the security/audit account
  GDPR workload → sign DPA in Artifact
```

## 9. SAA-C03 Perspective

- **"Download AWS compliance reports / SOC / FedRAMP / ISO"** → **AWS Artifact**
- **"Sign a BAA for HIPAA or DPA for GDPR"** → **AWS Artifact (Agreements)**
- **"Check my resources against rules"** → **AWS Config**; "consolidated findings/security standards" → **Security Hub**
- **"Continuous compliance evidence for audits"** → **AWS Audit Manager** (uses Config/Security Hub data + manual evidence)

Exam traps: "Artifact has an API/CLI" → **no, console-only**; "Artifact checks your account's compliance" → **no, it provides AWS's certifications & legal agreements**; "NDA is per-report" → **one-time acceptance covers NDA-protected reports**; "BAA per-service" → **sign once, covers all HIPAA-eligible services**; "Artifact replaces your compliance program" → **it's evidence, not a compliance solution**. Mnemonic: **Artifact = AWS's paperwork (certifications + agreements) · Config = your config compliance · Audit Manager = continuous audit evidence.**