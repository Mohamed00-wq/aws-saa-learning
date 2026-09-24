# Amazon Macie Course — S3 Sensitive Data Discovery & Protection

## 1. Purpose

Amazon Macie is a **fully managed data security / data privacy service** that **automatically discovers, classifies, and protects sensitive data** such as **PII, financial information, and credentials** stored in **Amazon S3**, using **machine learning and pattern matching**. It continuously evaluates your S3 data estate, generates actionable **findings**, and helps you meet compliance (GDPR, HIPAA, PCI-DSS) and data-residency requirements. For SAA the keyword pair is **"Sensitive data / PII / discover & classify" + "S3"**.

## 2. How it works

- **Enable Macie** (one click / API) → **Amazon S3 bucket inventory** of your data estate: buckets, objects, size, count, AWS account, Regions, tags, storage classes
- **Bucket-level security & access evaluation** — Macie continually evaluates bucket policies, detects **publicly accessible buckets**, **unencrypted buckets**, and buckets **shared with accounts outside your AWS Organizations**, and flags risky configurations
- **Two discovery modes:**
  - **Automated sensitive data discovery** — daily sampling of your bucket inventory; Macie retrieves representative objects and analyzes their contents continuously (recommended baseline; console "heat map" of sensitivity)
  - **Sensitive data discovery jobs** — one-time or scheduled (daily/weekly/monthly) deep, targeted scans of selected buckets/objects (by tags, prefixes, last-modified, etc.)
- **Detection = managed data identifiers + custom data identifiers + allow lists**
  - **Managed identifiers** — built-in detectors: credit card numbers, AWS secret access keys, passport numbers (per country), PII (names, addresses, IDs), financial, medical, credentials (a growing list for many countries/regions)
  - **Custom identifiers** — your own regex patterns, keywords, ignore-words (proprietary data, employee IDs, internal classification)
  - **Allow lists** — text/patterns to NOT flag (exceptions, sample data)
- **Output** — **sensitive data findings** (what/where) + **discovery results** (analysis log per object, incl. no-sensitive and error results) → **EventBridge**, **S3**, and **AWS Security Hub**
- **Multi-account** — a **Macie administrator** account manages up to 1,000 member accounts (up to 5,000 via AWS Organizations)
- **Formats supported** — .txt/.json/.csv/.tsv/.xml, .pdf, .doc/.docx, .xls/.xlsx, archives (.zip/.gzip/.tar), Parquet, Avro (must be in supported storage classes; object encryption must be accessible)

```
Macie admin (multi-account org)
  ├─ bucket inventory + continuous bucket security evaluation (public/encrypted/shared)
  ├─ automated discovery (daily sampling) &/or discovery jobs (one-time/scheduled)
  │    ├─ managed data identifiers (PII, cards, credentials...)
  │    ├─ custom identifiers (regex/keywords you define) + allow lists
  └─ findings → EventBridge / S3 / Security Hub → Lambda remediation, tickets
```

## 3. When to use

- **Discover where PII / sensitive data lives in S3** — data privacy compliance (GDPR, CCPA, HIPAA, PCI-DSS)
- **Continuous monitoring of data security posture** of S3 — exposed/unencrypted/mis-shared buckets at scale
- **Audit data before migration / analytics** — classify data so you can protect or separate sensitive sets
- **Automated response** — findings routed to EventBridge/Lambda (e.g., revoke public access, quarantine)
- **Organizations-wide S3 visibility** from a central Macie administrator account

## 4. When NOT to use

- **Detecting network/account threats** → GuardDuty (Macie is about *data*, not activity)
- **Scanning compute/containers for vulnerabilities** → Inspector
- **General S3 inventory/analytics needs** → S3 Inventory + Athena (Macie is focused on sensitive-data classification)
- **Not S3 data** (databases, files elsewhere) — Macie protects S3 objects specifically

## 5. Important features

- **ML + pattern matching** sensitive data discovery (PII, financial, credentials, geolocation-specific)
- **Automated sensitive data discovery** (continuous daily sampling) + **targeted discovery jobs** (one-time/recurring)
- **Bucket-level findings** — public access, unencryption, accounts outside org sharing, risky permissions
- **Managed + custom data identifiers and allow lists** to tailor detection
- **Findings → EventBridge, S3, Security Hub**; one-click reconcile in console
- **Macie administrator / multi-account** with member accounts; interactive **sensitivity heat map**
- **KMS integration** to analyze encrypted objects; GDPR/HIPAA/PCI-aligned

## 6. Limitations

- **S3-focused** (general purpose buckets; supported storage classes/formats) — not a general data-security scanner for other stores
- **Encrypted objects** must use keys Macie can access; unsupported formats/classes are skipped (logged as discovery results)
- **Sampling is sampling** — automated discovery may miss objects; deep jobs are the thorough option (more cost)
- Per-account/Region enablement; results repository (S3) must be configured for long-term storage

## 7. Trade-offs

- **Macie vs GuardDuty** — sensitive *data* (PII) & bucket posture in S3 (Macie) vs *activity*/threat detection across accounts (GuardDuty). Complementary security layers
- **Macie vs Inspector** — data-at-rest classification (Macie) vs compute/OS/container vulnerability scanning (Inspector)
- **Automated discovery vs jobs** — continuous cheap broad visibility vs targeted thorough scans when you need them
- **vs Security Hub** — generates findings Security Hub aggregates; Macie is a source, Security Hub the console

## 8. Architecture

```
Compliance posture:
  Enable Macie (admin account, org) → daily automated discovery across all S3
  Findings: "customer-data bucket publicly accessible + contains US passport numbers"
  → EventBridge → Lambda: apply blocking bucket policy / move object to quarantine prefix
  → Security Hub: aggregate for dashboard; job/member for deep scan of the affected prefix
```

## 9. SAA-C03 Perspective

- **"Discover / classify sensitive data (PII, cards, credentials) in S3"** → **Amazon Macie**
- **"Automate S3 sensitive-data discovery at scale for privacy/compliance"** → Macie
- **"Detected publicly accessible / unencrypted / externally-shared S3 bucket"** → Macie (bucket-level findings)
- **"ML + pattern matching to detect PII"** → Macie
- **"Detect threats/account compromise"** → GuardDuty; "vulnerability scanning of EC2/containers" → Inspector

Exam traps: Macie = **S3 data discovery**, not network/account threat detection (that's GuardDuty); "Macie scans all AWS data" → **no, S3 objects**; "you must build rules from scratch" → **managed identifiers are built-in; custom ones optional**; "findings only in console" → **exportable to EventBridge/S3/Security Hub**. Simple mnemonic: **GuardDuty = activity, Macie = data at rest in S3, Inspector = vulnerabilities.**