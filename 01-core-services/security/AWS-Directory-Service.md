# AWS Directory Service Course — Managed Microsoft AD / AD Connector / Simple AD

## 1. Purpose

AWS Directory Service provides **managed Microsoft Active Directory** in the cloud so directory-dependent workloads (Windows EC2, RDS SQL Server, FSx for Windows, SharePoint, WorkSpaces, IAM Identity Center SSO) can authenticate users/computers **without you running and patching domain controllers**. Three offerings: **AWS Managed Microsoft AD** (full Microsoft AD as a service), **AD Connector** (proxy to your existing on-prem AD), and **Simple AD** (cheap standalone Samba-based directory). For SAA it's the answer to **"managed Active Directory / join EC2 to a domain / SSO vs your on-prem AD / RDS SQL Server authentication"**.

## 2. How it works

- **AWS Managed Microsoft AD** — Microsoft **Active Directory (Windows Server 2019)** managed by AWS, deployed across **multiple AZs** (2+ domain controllers, DNS included, daily snapshots); you create/read users & groups; can **scale out with additional domain controllers**
  - Full AD features: **Group Policy (GPOs)**, LDAP/TLS (Secure LDAP), **trust relationships** (forest trusts to your on-prem AD), MFA, PowerShell, schema extensions, ADR/Recycle Bin, group-managed service accounts
  - Integrates with AWS apps: **Amazon EC2** (domain join), **Amazon RDS for SQL Server**, **Amazon FSx for Windows File Server**, **Amazon WorkSpaces/WorkDocs/QuickSight**, **AWS IAM Identity Center** (identity source), Elastic Beanstalk, AWS Management Console SSO
  - **Hybrid Edition** — extends/connects your *existing self-managed AD* into an integrated identity environment
- **AD Connector** — a **directory gateway / proxy**: keeps your **existing on-prem Microsoft AD** authoritative, forwards auth requests to your domain controllers (no directory sync, no caching of directory objects). Add one service account in AD, then AWS apps (WorkSpaces, EC2 Windows **seamless domain join**, WorkDocs, QuickSight) authenticate against your on-prem AD. Supports **MFA via RADIUS**
- **Simple AD** — standalone, **Samba 4-based** Microsoft-AD-*compatible* directory (cheaper). **Small (≤500 users) / Large (≤5,000 users)**. Basic features: users/groups, GPOs subset, Kerberos SSO, daily snapshots, EC2 join. **Missing:** MFA, AD trusts, AD Administrative Center, PowerShell, recycle bin, group-managed service accounts, schema extensions; incompatible with RDS SQL Server/Oracle, FSx, IAM Identity Center, WorkSpaces apps

> ⚠️ **2026 availability note:** **Simple AD is no longer open to new customers** — AWS directs new customers to AWS Managed Microsoft AD or AD Connector ("Simple AD availability changes").

```
AWS Managed Microsoft AD (in AWS): full AD, multi-AZ, GPO/trusts/MFA, joins EC2/RDS SQL/FSx/WorkSpaces, SSO
AD Connector: your on-prem AD stays authoritative → proxy/gateway to AWS apps or with RADIUS MFA
Simple AD: standalone Samba-4 directory (cheap, basic) — no longer for new customers
Integration: IAM Identity Center, EC2, RDS SQL Server, FSx Windows, WorkSpaces, Console SSO
```

## 3. When to use

- **Directory-aware Windows workloads in AWS** — domain-join EC2, run **RDS for SQL Server**, **FSx for Windows File Server**, SharePoint, .NET apps
- **Need full Microsoft AD features managed** — GPOs, trusts (join an existing AD forest), MFA, schema extensions without running DCs
- **Extend/connect on-prem AD to AWS** — **Managed Microsoft AD (Hybrid)** or **AD Connector** (proxy) for SSO + AWS app access using existing AD credentials
- **SSO for AWS console/apps via Active Directory** — IAM Identity Center connected directory
- **Giving AWS apps existing AD auth** (quick prototype) — AD Connector (no sync, no new directory)

## 4. When NOT to use

- **Identity for pure-AWS / non-Microsoft stacks** with no Active Directory dependency → **IAM Identity Center / Cognito** (AD not needed)
- **Unmanaged / fully-custom directory on your own EC2** → run your own domain controllers (you take the ops burden)
- **Simple AD today** → closed to new customers; use Managed Microsoft AD or AD Connector
- **Non-AD authentication needs** (JWT/OAuth for web apps) → Cognito
- **Trust doesn't exist / you don't want AD semantics** — for basic object storage of users, a database/SSO product is simpler

## 5. Important features

- **AWS Managed Microsoft AD (Standard/Enterprise/Hybrid)** — full AD: GPOs, **forest trusts**, **Secure LDAP (TLS)**, MFA/gMSAs, PowerShell, schema extension, multi-AZ HA, **scale-out DCs**, daily snapshots
- **Category-leading AWS integrations** — **EC2 domain join, RDS SQL Server, FSx for Windows, WorkSpaces/WorkDocs, QuickSight, AWS IAM Identity Center, Elastic Beanstalk, Managed Console SSO**
- **AD Connector** — proxy to existing on-prem AD (no sync), seamless EC2 domain join, RADIUS MFA, multi-AZ, works with WorkSpaces/QuickSight/WorkDocs/IAM Identity Center
- **Simple AD** — cheap Samba-4 standalone directory (dev/test, small teams) — *legacy, closed to new customers*
- **Multi-AZ + monitoring + snapshots** are AWS-managed; **LDAP/TLS** for secure directory access

## 6. Limitations

- **Managed Microsoft AD** — you don't own/see the DCs (org-level GPO/master admin is delegated; some Active Directory admin features require a *setup* delegated admin account); Region-scoped (setup per Region)
- **AD Connector** — requires an on-prem AD reachable via VPC/DPX/VPN; **not compatible with RDS SQL Server** (can't be used for SQL Server auth); no directory objects stored (authoritative on-prem)
- **Simple AD** — **no longer for new customers**; subset features only (no MFA/trusts/schema/RDS SQL/FSx/IAM Identity Center)
- **Hybrid/Connector** need connectivity planning (VPN or Direct Connect to on-prem DCs)

## 7. Trade-offs

- **Managed Microsoft AD vs Simple AD** — full AD semantics/GPO/trusts/RDS-SQL/FSx integration (Managed AD) vs cheaper basic Samba directory (Simple AD — legacy/closed)
- **Managed Microsoft AD vs AD Connector** — run a managed AD **in AWS** (users/GPOs live here, you manage them in AWS) vs **keep using your existing on-prem AD** (AWS is just a gateway; nothing to manage cloud-side, but on-prem AD is the dependency)
- **vs self-managed DCs on EC2** — managed multi-AZ HA/patching/backups vs full control + ops burden (licensing available for Managed AD)
- **vs IAM Identity Center (formerly SSO)** — AD as identity *source* vs IAM Identity Center as the user portal; they even combine (Identity Center + AD connector/managed AD)

## 8. Architecture

```
Hybrid Windows shop moving to AWS:
  On-prem AD forest ←(forest trust / Hybrid Edition)→ AWS Managed Microsoft AD (multi-AZ)
  ├─ EC2 Windows instances domain-joined (GPO applied)
  ├─ RDS SQL Server / FSx for Windows authenticate via Managed AD
  └─ IAM Identity Center uses Managed AD as identity source → SSO to console/apps
  (Alternative: AD Connector in place of Managed AD → proxy auth back to on-prem DCs, RADIUS MFA)
```

## 9. SAA-C03 Perspective

- **"Managed Active Directory for EC2/RDS SQL Server/FSx Windows/WorkSpaces"** → **AWS Directory Service (Managed Microsoft AD)**
- **"SSO with existing on-prem AD without duplicating the directory"** → **AD Connector** (proxy, no sync)
- **"Join Windows EC2 to a domain / Group Policy"** → Managed Microsoft AD
- **"RDS SQL Server needs a directory"** → Managed Microsoft AD (AD Connector/simple don't qualify)
- **"AWS-native identity, no Windows dependency"** → **IAM Identity Center** / Cognito

Exam traps: "Simple AD is like Managed AD" → **no, cheaper Samba-4 subset (and closed to new customers as of 2026)**; "AD Connector copies your AD into AWS" → **no, it's a proxy to on-prem (no data copied/synced)**; "AD Connector works with RDS SQL Server" → **no, use Managed Microsoft AD**; "IAM Identity Center must use AD" → **no, AD is one option among identity sources**. Memorize the three: **Managed AD = full managed AD in cloud · Connector = proxy to on-prem · (Simple AD = legacy standalone basic)**.