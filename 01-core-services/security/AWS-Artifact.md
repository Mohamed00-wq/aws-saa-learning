# AWS Artifact — Compliance Reports & Agreements

## What it is

AWS Artifact is a self-service portal for retrieving **compliance reports** and **legal agreements** from AWS. It's where you download SOC reports, PCI DSS attestations, ISO certificates, FedRAMP authorizations, and sign Business Associate Agreements (BAAs) or Data Processing Addendums (DPAs). No need to email AWS or wait weeks — Artifact gives you on-demand access.

For the SAA exam, Artifact is the answer when a question asks how to obtain AWS compliance documentation, how to sign a BAA for HIPAA, or how to download a SOC 2 report for an auditor. It's a simple service — know what it provides and where to find it.

## What Artifact provides

### Artifact Reports

| Report | What it covers |
|---|---|
| **SOC 1 / SOC 2 / SOC 3** | Security, availability, processing integrity, confidentiality, privacy controls |
| **PCI DSS** | Payment card industry compliance (relevant if handling credit cards) |
| **ISO 27001 / 27017 / 27018** | International information security standards |
| **FedRAMP** | Federal Risk and Authorization Management Program (government workloads) |
| **IRAP** | Australian Government security assessment |
| **MTCS** | Singapore Government security standard |
| **C5** | German Cloud Computing Compliance Criteria |
| **CSA STAR** | Cloud Security Alliance certification |
| **Cyber Essentials Plus** | UK Government security standard |

- Reports are available in **PDF** format.
- Updated periodically — Artifact always shows the **latest version**.
- Some reports require an **NDA** (Non-Disclosure Agreement) before download.

### Artifact Agreements

| Agreement | When you need it |
|---|---|
| **BAA (Business Associate Agreement)** | HIPAA-regulated workloads — required before processing PHI on AWS |
| **DPA (Data Processing Addendum)** | GDPR compliance — required when processing EU personal data |
| **Addendum for Service Information** | Supplemental info for certain service agreements |

- Agreements are presented for **electronic signature** in the Artifact console.
- Once signed, the agreement is stored in Artifact for future reference.
- **BAA covers all AWS services** that are HIPAA-eligible — sign once, use across all eligible services.

## NDA requirement

- Some compliance reports (especially SOC 1 and SOC 2 Type II) are classified as **confidential**.
- You must accept an **AWS NDA** in Artifact before downloading these reports.
- NDA is a one-time acceptance — once accepted, you can download NDA-protected reports freely.

## Multi-account access

- **Individual account access**: each IAM user accesses Artifact with their own AWS account.
- **No cross-account sharing**: you can't download a report from Artifact and share it externally (violates NDA).
- **AWS Organizations**: each member account can access Artifact independently.

## Where Artifact lives

- AWS Management Console → search "Artifact" in the services menu.
- No API — Artifact is **console-only** for reports and agreements.
- No CLI/SDK access to download reports (but reports can be programmatically accessed via S3 after download).

## Pricing

AWS Artifact is **free** — no charge for downloading reports or signing agreements.

## Exam domains

- [x] **Secure (30%)** — SOC/ISO/FedRAMP compliance documentation, BAA for HIPAA, NDA-protected reports
- [x] **Resilient (26%)** — centralized access to compliance evidence for audits
- [x] **High-Performing (24%)** — self-service portal, no manual AWS requests needed
- [x] **Cost-Optimized (20%)** — free service, no cost considerations

## Key gotchas

1. **Artifact is console-only** — no API or CLI access to download reports directly
2. **SOC 1 / SOC 2 Type II reports require NDA acceptance** — can't download without it
3. **BAA covers all HIPAA-eligible services** — sign once in Artifact, use across Lambda, EC2, RDS, S3, etc.
4. **Reports are updated periodically** — always download the latest version for audit submissions
5. **Artifact is per-account** — no cross-account report sharing; each account needs its own access
6. **IAM users need explicit Artifact permissions** — `artifact:*` actions in IAM policy to access the console
7. **Don't email NDA-protected reports** — use secure sharing methods; distribution violates NDA terms
8. **DPA is for GDPR** — BAA is for HIPAA; know which agreement matches which regulation
9. **FedRAMP reports are in Artifact** — not separate; download from the same portal as SOC/ISO
10. **Artifact doesn't replace compliance** — it provides evidence; you're still responsible for your own compliance posture

## Related services

- **AWS-Config** — tracks resource compliance against rules; Artifact provides AWS-side compliance evidence
- **AWS-Security-Hub** — aggregates security findings; Artifact shows AWS compliance certifications
- **IAM** — controls who can access Artifact in your account
- **KMS** — Artifact reports are not KMS-encrypted (stored in AWS-managed S3)
