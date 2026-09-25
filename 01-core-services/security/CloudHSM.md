# AWS CloudHSM Course  Dedicated, Single-Tenant Hardware Security Module (HSM)

## 1. Purpose

AWS CloudHSM provides **dedicated, single-tenant hardware security modules (HSMs)** hosted in AWS  gives you **FIPS 140-2/140-3 Level 3 validated** crypto hardware that **you fully control** (keys, users, policies). Unlike KMS, the **cryptographic keys never leave your HSM**, are managed by *you*, and CloudHSM supports the **industry-standard crypto APIs (PKCS#11, JCE, OpenSSL Providers, Microsoft CNG/KSP)**  for workloads that must keep absolute control of keys, meet compliance (e.g., PCI DSS key mgmt), or need key exportability/classic HSMs. For SAA the keyword is **"dedicated hardware security module / keep keys in your own HSM / PKCS#11 & JCE / regulatory key control"**, often compared to **KMS**.

## 2. How it works

- Approvision **HSM instances** per AZ (an HSM *cluster* typically has **2+ HSMs in multiple AZs** for HA), each exposed inside your VPC via **ENIs**
- Authenticate to the HSM with **HSM users** (CU/CO/Appliance) rather than IAM  access is cryptographically controlled by the HSM itself
- Generate, store, and use keys entirely **inside the tamper-resistant HSM** use with native **PKCS#11 / JCE / OpenSSL / Microsoft CNG** libraries (traditional HSM integration)
- **AWS CloudFormation/CLI** support for provisioning management via AWS Management Console or CLI
- **CloudHSM Classic** (the legacy service) has been **retired**  the current product is CloudHSM as described (with `cloudhsmv2` API)
- **Clusters & HA**  define a cluster (one in a Region), add HSMs per AZ CloudFormation creates a cluster with the first HSM and auto-detects others
- **Key management is yours**  including **backup/restore**: on-demand or policy-based backups of HSM state (keys, users) restore into the same or another cluster (backup restore API hardware type must be compatible)

```
VPC ── HSM cluster
  HSM-A (AZ-a) ─ ENI  \
                       └─► HSMs sync (HA) keys live only inside HSMs
  HSM-B (AZ-b) ─ ENI  /
Clients use PKCS#11 / JCE / OpenSSL Provider / CNG → ENIs
  ├─ keys never leave you own users (CU/CO), policies
  ├─ backups → on-demand/scheduled → restore (same/other cluster)
  └─ FIPS 140-2/140-3 Level 3 validated hardware
Integration with AWS services: KMS custom key stores (HSM-backed), ACM Private CA
```

## 3. When to use

- **Regulatory/contractual key control**  your org must physically own/manage keys (e.g., PCI-DSS key-management requirements, sovereign/regulatory mandates)
- **Legacy/enterprise crypto stacks**  applications written against **PKCS#11 / JCE / OpenSSL / Microsoft CNG** that need an on-prem-style HSM in the cloud
- **Key exportability**  use KMS custom key store backed by CloudHSM when you need to *move/export* keys (KMS alone keeps keys non-exportable)
- **High-assurance separation**  key/crypto isolation from the AWS "user cloud" (dedicated single-tenant hardware) is the selling point
- **FIPS Level 3 hardware compliance** while offloading the physical HSM lab to AWS

## 4. When NOT to use

- **Simple AWS-native key management, auto-rotation, CloudTrail integration with S3/KMS-integrated services (EBS, S3, Lambda env encryption, DB encryption)** → **use AWS KMS** (deep service integration, far less ops)
- **You want AWS to own/rotate key material per policy (envelope encryption default, AWS managed keys)** → KMS
- **You need user-identity IAM control rather than native HSM users** → KMS (with IAM)  CloudHSM auth is HSM-native
- **Small/dev workloads**  CloudHSM is per-HSM hourly cost ($1.60/hr/HSM+ in us-east-1) + cluster ops KMS (cheap, serverless) is better fit

## 5. Important features

- **Dedicated single-tenant HSMs** in your VPC via ENIs **HA across AZs with multiple HSMs**
- **FIPS 140-2/140-3 Level 3** validated hardware
- **Full control of keys & users**  HSM-native authentication (CU/CO), no IAM-based crypto
- **Standards crypto APIs**  **PKCS#11, Java JCE, OpenSSL (provider), Microsoft CNG/KSP**
- **Backups** on-demand or scheduled restore to same/another cluster (same hardware type)
- **AWS integration niches**  **KMS custom key store** (KMS key material physically stored in your HSM) and **ACM Private CA** (H2: CloudHSM-backed CA)
- **CloudFormation** provisioning no minimum commitments (pay per HSM-hour)

## 6. Limitations

- **Not integrated with most AWS services** (unlike KMS)  you integrate via SDK/APIs yourself no auto-rotation, no CloudTrail key events, no per-service key policies
- **You manage users, policies, and HA/cluster operations**  real operational burden and learning curve (this is by design  you own the crypto plane)
- **Per-HSM hourly cost + cluster ≥2 for HA** backups/complexity add up for small workloads
- **Keys/backups are Region/hardware-type bound** to some degree restoring needs compatible clusters/cluster id
- **No IAM-native access control over crypto ops**  HSM users governed inside the HSM

## 7. Trade-offs

- **CloudHSM vs AWS KMS (the big exam comparison):**
  | CloudHSM | AWS KMS |
  |---|---|
  | Single-tenant, dedicated HSM hardware | Shared managed service, AWS owns/rotates |
  | You own keys & users (control plane) | AWS manages key material & access via IAM |
  | FIPS 140-2/140-3 Level 3, keys never leave HSM | FIPS validation cap (140-2 L3 validation on request) |
  | Keys **exportable** (your call) | Keys **non-exportable** (design) |
  | Traditional APIs: PKCS#11/JCE/OpenSSL/CNG | AWS-native SDK/API, service integrations (S3, EBS, Lambda, Secrets etc.) |
  | You operate cluster/HA/backups | Fully managed, zero ops |

- **KMS custom key store (CloudHSM-backed KMS)** balances both  KMS API/UX on your own HSMs (best of both when you need it)
- **vs ACM/Private CA**  CloudHSM can back ACM Private CA H2 CA for stronger key control in a certificate chain
- **vs SSO/native cloud crypto**  the price of control is ops complexity choose CloudHSM for compliance/export requirements only

## 8. Architecture

```
Compliance-critical signing/encryption:
  CloudHSM cluster (2 HSMs in 2 AZs ← ENIs) inside VPC
  Apps: PKCS#11 (C), JCE (Java/Spring), OpenSSL Provider  talking to ENIs
  Backup policy -> S3? no  HSM backups to dedicated backup store restore on demand
  Optional: KMS custom key store mapped to this cluster for KMS-style use of your own HSMs
  ACM Private CA + CloudHSM for CA private keys under your control
```

## 9. SAA-C03 Perspective

- **"Dedicated / single-tenant HSM hardware / keys fully under my control"** → **CloudHSM**
- **"App must speak PKCS#11 / JCE / OpenSSL / CNG to an HSM"** → CloudHSM
- **"Keys must be exportable / I need to move key material"** → CloudHSM (or KMS custom key store on CloudHSM)
- **"Manage keys in a service integrated with S3/EBS/Lambda etc., AWS handles rotation"** → **AWS KMS**
- **"Envelope encryption / default AWS-native encryption management"** → KMS

Exam traps: "CloudHSM is fully integrated with S3/EBS/Lambda" → **no, KMS is CloudHSM is a raw HSM you integrate yourself** "AWS manages CloudHSM keys" → **no, you own the keys/users/policies** "CloudHSM keys are exportable by default like KMS" → **KMS keys are never exportable CloudHSM keys can be exported (your choice)** "CloudHSM = KMS" → **different models  know the trade-off table**. Mnemonic: **KMS = managed shared keys for AWS services · CloudHSM = your dedicated hardware crypto, your rules.**