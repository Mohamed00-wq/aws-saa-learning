# AWS Transfer Family Course — Managed SFTP / FTPS / FTP / AS2 File Servers

## 1. Purpose

AWS Transfer Family is a **fully managed file-transfer service** that runs **SFTP, FTPS, FTP, and AS2 servers** so your **partners/customers/users can keep using their familiar file-transfer clients** (WinSCP, FileZilla, scripts) to exchange files — while the files land **natively in Amazon S3 or EFS** (not a file server you operate). It's the answer to **"managed SFTP/FTP server / B2B file exchange / AS2 EDI / keep existing client config"**.

## 2. How it works

- **Create a server** (one server = endpoint + protocol) with **endpoint type** — **Public** (internet-accessible; SFTP only) | **VPC-Internet** | **VPC-Internal** (all broadly SFTP/FTPS/AS2) — provisioned in your VPC with **PrivateLink** where chosen
- **Protocols** — **SFTP v3** (SSH), **FTPS** (TLS, port 21 control + data ports 8192–8200), **FTP** (plaintext — credentials sent in clear; discouraged/isolate creds), **AS2** (B2B structured-data over HTTPS + signing/MIC), and **browser-based web-app** uploads/management
- **Identity providers** — **service-managed** (SSH keys), **AWS Managed Microsoft AD**, **custom** via **Lambda/API Gateway** (external DB / LDAP / any IdP supported)
- **Backing storage** — S3 (IAM maps users to bucket/prefix; logical home directories, parameterized/home-dir-per-user) or **EFS** (POSIX ownership); FSx for ONTAP supported too
- **Post-upload automation** — **managed workflows** (copy, tag, decrypt, run Lambda step on file arrival/partial) and **connectors** for **outbound** SFTP to partners/on-prem
- **Security & ops** — IAM roles, KMS/SSE at rest, TLS in transit, CloudWatch logs/metrics, CloudTrail audit, Route 53 custom hostnames, WAF for web apps; HA/autoscaling managed; 24/7

```
Partners (SFTP/FTPS/AS2 clients — unchanged config) → AWS Transfer Family server endpoint
  ├─ Auth: service-managed keys | AWS Managed Microsoft AD | Lambda/API-GW custom
  ├─ data → Amazon S3 (IAM scoped) or EFS (POSIX access)
  ├─ managed workflows: tag/copy/decrypt/Lambda on upload → trigger processing
  └─ connectors for outbound SFTP to partner servers; audit + alert via CloudWatch/CloudTrail
```

## 3. When to use

- **Replace/migrate a self-managed FTP/SFTP server onto AWS** without forcing partners to change clients
- **B2B file exchange & EDI** — AS2 (retail/healthcare/finance), SFTP between organizations
- **Regulated industries** file dropoffs/collections — banking, insurance, healthcare (HIPAA-friendly), retail
- **Files that need to be processed by AWS** on arrival — land in S3 → step via workflows → analytics (Athena), ML, etc.
- **External users need FTP-family access to an S3/EFS dataset with enterprise auth** (AD, LDAP, MFA via custom IdP)

## 4. When NOT to use

- **Store-to-store / one-way bulk migration or scheduled replication** → **AWS DataSync** (agent-based, fast, incremental)
- **Ongoing protocol access/caching for on-prem apps in a hybrid setup** → **Storage Gateway (File Gateway)**
- **Authenticated app-to-app uploads in your own code** → S3 **presigned URLs** (simpler/cheaper, less infra)
- **Physical/offline big-data movement** → Snowball / other AWS gears
- **Real-time event streaming** → Kinesis/MSK (Transfer Family is batch file transfer)

## 5. Important features

- **Server protocols** — SFTP v3, FTPS (TLS), FTP, **AS2** (certificate-based signing/MIC for B2B), plus **browser-based web app** transfers and a custom Web App (WAF-supported)
- **S3 or EFS backing natively** — IAM to S3 (users via permissions, logical directories, home-directory-per-user), POSIX to EFS; files become objects → Athena/Comprehend/Translate/ML processing directly
- **Identity providers** — service-managed (SSH keys) | **AWS Managed Microsoft AD** | **custom Lambda/API Gateway** (bring any IdP/MFA flow)
- **Managed workflows** — post-upload steps (copy, tag, custom file-processing/Lambda step, decrypt) triggered on complete/partial upload
- **Outbound connectors** — transfer to external/on-prem SFTP endpoints from S3
- **Automation & HA** — autoscaling endpoints, multi-AZ availability, Route 53 hostnames, **CloudWatch & CloudTrail**, AWS WAF for web apps, **KMS** encryption, PrivateLink endpoints
- **Zero client change migration** — keep existing firewall ports/config; FTP/FTPS data ports 8192–8200

## 6. Limitations

- **Not a general SSH server / shell access** — Transfer Family is file transfer only (no interactive shell)
- **FTP is unencrypted** — avoid; isolate FTP credentials from SFTP/FTPS if enabled
- **Endpoint types differ per protocol** — e.g., Public endpoints serve **SFTP only**; FTP/FTPS/AS2 typically need VPC-hosted endpoints (plan networking)
- Provisioned **server/endpoint cost** (per-server hourly, data transfer charges); workflow/connectors add more pieces
- Custom identity provider needs your team to build (Lambda/API GW if you don't use service-managed/AD)

## 7. Trade-offs

- **Transfer Family vs DataSync** — humans/partners using FTP/SFTP/AS2 Protocols into S3/EFS (Transfer Family) vs automated store-to-store bulk migration/replication with agent + schedule (DataSync)
- **Transfer Family vs Storage Gateway** — managed FTP/SFTP endpoints for external users over the internet (Transfer Family) vs NFS/SMB/File-cached access for on-prem/internal workloads (Gateway)
- **Transfer Family vs S3 presigned URL** — full managed multi-protocol server with auth/IdP/workflows for partners vs simple programmatic URL-based uploads (build it yourself)
- **SFTP vs AS2** — SSH-file-transfer with keys vs signed EDI documents over HTTPS (AS2); pick by trading-partner requirements

## 8. Architecture

```
Partner file exchange:
  Partner SFTP client → Transfer Family (VPC-hosted endpoint, service-managed/AD auth)
    → S3 bucket (per-user logical home dirs, IAM scoping)
  Upload completes → managed workflow: decrypt → copy to processing prefix → Lambda validates
  → analytics/mart via Athena; CloudWatch alert on failures; CloudTrail audit trail
As2 leg for EDI partners: certificates imported in ACM; profiles agreements on the same server
```

## 9. SAA-C03 Perspective

- **"Managed SFTP/FTPS/FTP/AS2 server; existing partners/clients don't change"** → **Transfer Family**
- **"B2B/EDI file exchange (AS2), healthcare/retail/finance drop-offs"** → **Transfer Family**
- **"Store files natively in S3/EFS, trigger serverless processing on upload"** → Transfer Family + managed workflows + Lambda
- **"Migrate on-prem/other-cloud bulk files, scheduled incremental online sync"** → **DataSync**
- **"Hybrid on-prem apps want NFS/SMB access"** → **Storage Gateway**; "app-to-app URL uploads" → **presigned URLs**

Exam traps: Distinguish file-movers — **Transfer Family** (FTP/SFTP/AS2 server endpoints for users/partners) vs **DataSync** (agent-based server-to-server sync) vs **Storage Gateway** (gateway protocol access) vs **Snowball** (offline). Transfer Family is NOT a shell; FTP is insecure; Public endpoints = SFTP only.