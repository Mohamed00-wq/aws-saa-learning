# AWS Audit Manager Course — Continuous Compliance Evidence Collection

## 1. Purpose

AWS Audit Manager **continuously audits AWS usage** to simplify risk & compliance assessment: it automates **evidence collection**, maps your AWS resources to the requirements of **industry frameworks/regulations**, and produces **audit-ready assessment reports**. It replaces the manual, spreadsheet-driven "grab the logs before the auditor arrives" process. For SAA the keyword is **"continuous compliance / evidence collection / audit reports / frameworks (SOC 2, ISO, PCI, HIPAA, GDPR)"**.

> ⚠️ **2026 availability change (SAA-relevant):** AWS Audit Manager is **no longer open to new customers as of April 30, 2026**. Existing customers continue to use the service normally. (Study it as a concept/comparison for the exam's compliance-architecture questions.)

## 2. How it works

- **Framework** — a predefined (or custom) collection of **controls**, each mapped to AWS best practices for a compliance standard. Prebuilt frameworks include **SOC 2/3, ISO 27001, PCI DSS, HIPAA, GDPR, NIST CSF, CIS Controls/BS v1.2–1.4, FedRAMP Moderate, AWS Foundational Security Best Practices** (and more)
- **Assessment** — you scope accounts/Regions to a framework; Audit Manager continuously collects evidence against each control
- **Evidence sources** — **AWS CloudTrail** (API activity), **AWS Config** (resource config rules), **AWS Security Hub** (CSPM findings), **AWS License Manager**, plus **manually-uploaded evidence** (policy docs, training records, architecture diagrams). Evidence is timestamped and tied to the control it supports
- **Evidence folders & delegation** — you/team review evidence, comment, mark status; **delegate reviews** to specific people
- **Evidence finder** — search collected evidence and export CSV; generate **assessment reports** (summary + organized evidence folders) ready for auditors
- **Multi-account** — via **AWS Organizations**: assessments run over multiple accounts and consolidate into a **delegated administrator account**
- Integrates with third-party **GRC systems** (SQL/API to export evidence)

```
Compliance standard (SOC2/ISO27001/PCI...) → prebuilt framework (controls mapped to data sources)
  → scoped ASSESSMENT (accounts, Regions, evidence sources: CloudTrail/Config/Security Hub/manual)
  → continuous automated evidence collection → evidence folders → review/delegate
  → audit-ready assessment report
```

## 3. When to use

- **Preparing for / maintaining compliance** — SOC 2, ISO 27001, PCI-DSS, HIPAA, GDPR, FedRAMP audits
- **Internal audit & governance** — prove controls operate effectively (security, change management, backup/DR, licensing)
- **Continuously collecting evidence** so audit time is quick and predictable (no more log-hunting)
- **Multi-account org** — centralized evidence collection across AWS Organizations accounts
- **Integrating evidence into your existing GRC system**

## 4. When NOT to use

- **Automated compliance *checks/config monitoring*** → AWS Config / Security Hub (Audit Manager collects evidence *for audits*; Config rules are a data source to it)
- **Threat detection** → GuardDuty
- **Live security investigations** → Detective
- **Patching/monitoring posture in real-time** → you need Config/Security Hub, not an audit workflow
- **New AWS sign-ups of Audit Manager** — closed to new customers since April 30, 2026; for equivalents use Config rules/SCPs + Security Hub evidence pipelines or third-party GRC

## 5. Important features

- **Prebuilt frameworks** — SOC 2/3, ISO 27001, PCI DSS, HIPAA, GDPR, NIST CSF, CIS, FedRAMP Moderate, AWS Foundational Security Best Practices, ACSC/CCCS, license frameworks, and more
- **Automated evidence collection** from CloudTrail, Config, Security Hub (CSPM), License Manager + **manual uploads**
- **Custom frameworks & controls** (or customize prebuilt) to match internal policies/data sources
- **Assessment scoping & multi-account** (AWS Organizations, delegated administrator), delegation & review workflow
- **Evidence finder** (search + CSV), **audit-ready assessment reports**
- **Third-party GRC integrations**; CloudFormation/SDK support

## 6. Limitations

- **Not a security monitor** — it's a compliance-*evidence* product (depends on CloudTrail/Config/Security Hub for automated evidence)
- **Procedural/manual controls** still need human-provided evidence; "frameworks don't guarantee you pass the audit"
- **Closed to new customers since April 30, 2026** (existing customers continue) — plan alternatives for new deployments
- Pricing is per-assessment/per-control territory (paid service); report export cost components

## 7. Trade-offs

- **Audit Manager vs AWS Config / Security Hub** — continuous *audit evidence & reports for compliance* (Audit Manager) vs *configuration rules/checks & consolidated findings* (Config/Security Hub). Audit Manager *consumes* Config rules and Security Hub findings as evidence
- **vs GuardDuty/Detective** — compliance/risk assessment vs threat detection/investigation (different goals)
- **vs manual GRC spreadsheets/custom pipelines** — automated continuous evidence & delegation vs spreadsheets/manual collection (Audit Manager removes the manual part)
- **Prebuilt vs custom framework** — fast start with industry standards vs building your own control sets

## 8. Architecture

```
ISO 27001 / SOC 2 program:
  ┌─ CloudTrail (API activity) ┐
  ├─ Config rules (config CSPM) ├─► Audit Manager assessment (scoped org, delegated admin)
  ├─ Security Hub (findings)    │      ├─ evidence folders per control → reviewers/comments
  └─ manual (policies, training)┘      └─ report package → auditors
  Automated collection runs continuously; report at any time
```

## 9. SAA-C03 Perspective

- **"Continuously audit AWS for compliance / collect evidence automatically"** → **AWS Audit Manager**
- **"SOC 2 / ISO 27001 / PCI DSS / HIPAA / GDPR audit preparation"** → Audit Manager frameworks
- **"Audit-ready evidence & reports from CloudTrail/Config/Security Hub"** → Audit Manager
- **"Check resource configs against rules"** → **AWS Config**; "consolidate findings/dashboards" → **Security Hub**
- **"Detect threats"** → GuardDuty; "investigate findings" → Detective

Exam traps: "Audit Manager detects threats" → **no, compliance evidence collection**; "Audit Manager = Config" → **it uses Config as a data source, but is an audit/reporting workflow product**; "all controls are automated" → **procedural controls need manual evidence**; **Audit Manager is closed to new customers since April 30, 2026** — know that as a current-state fact and pair alternatives (Config/SCPs/Security Hub).