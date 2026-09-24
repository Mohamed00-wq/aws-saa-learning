# AWS Firewall Manager Course — Centralized Firewall/Security Policy Across Accounts

## 1. Purpose

AWS Firewall Manager **centrally manages and enforces firewall & network-security policies across ALL accounts and resources in an AWS Organizations organization** from a single administrator account. You define a protection policy once; Firewall Manager **automatically applies it to existing resources AND any future ones** (new accounts, new VPCs, new resources), and reports/remediates non-compliant ones. For SAA it's the answer to **"centrally enforce WAF/Shield/Security Groups/Network Firewall/DNS Firewall across many accounts — including new accounts (Organizations)"**.

## 2. How it works

- **Setup** — prerequisites: **AWS Organizations** enabled, **Firewall Manager delegated administrator** (best practice: a *dedicated security account*, not the management account), **AWS Config** (who it relies on for resource inventory/audit), and the underlying services you'll control (AWS WAF, Shield Advanced subscription for Shield policies, AWS Network Firewall, Route 53 Resolver DNS Firewall)
- **Policies** — each policy defines a **policy type**, a **resource group** (account / resource type / tags), a **scope** (specific accounts or all accounts), and **remediation action**
- **Policy types**:
  - **AWS WAF policies** — deploy Web ACLs/rule groups across **CloudFront distributions, ALBs, API Gateways, AppSync, Cognito, App Runner, Verified Access**; enforces Managed Rules; supports Marketplace rules. Key behavior: **first & last rule groups are enforced centrally, while account teams may add rule groups in between** (hierarchical enforcement)
  - **AWS Shield Advanced policies** — auto-subscribe all (or subset of) member accounts to Shield Advanced; deploy protections; auto-subscribe future accounts
  - **Amazon VPC security group policies** — **Common SG** (replicate a baseline SG to every VPC/resource), **Content audit** (guardrails: allowed/disallowed rules; catch overly-permissive SGs), **Usage audit** (identify unused/redundant SGs)
  - **AWS Network Firewall policies** — deploy firewall endpoints + rules to VPCs (incl. creating new endpoints & route-table updates for non-compliant VPCs)
  - **Route 53 Resolver DNS Firewall policies** — associate rule groups with VPCs
  - **Network ACL (NACL) policies** and **third-party firewall policies** (e.g., Palo Alto, Fortinet from AWS Marketplace) centrally deployed/monitored
- **Compliance & reporting** — Firewall Manager continuously checks resources, flags non-compliant VPCs/accounts (e.g., VPCs missing Network Firewall), **auto-remediates** when enabled, and sends findings to **Security Hub**
- **DDoS at scale** — centralized **Shield Advanced** attack monitoring across the organization

```
Org (Organizations) ─► Firewall Manager delegated admin (security account)
  Policy: WAF / Shield Advanced / Security Groups / Network Firewall / DNS Firewall / NACL / 3rd-party
  ├─ scope: accounts ∪ tags ∪ resource types
  ├─ auto-applies to existing + NEW resources/accounts (continuous compliance)
  └─ non-compliant → report + auto-remediate; findings → Security Hub
```

## 3. When to use

- **Multiple accounts / a whole AWS Organization** needing consistent firewall posture (WAF, Shield, SGs, Network Firewall, DNS Firewall)
- **Enforcing security baselines on resources you don't control day-to-day** — teams create new accounts/ALBs/CloudFront distributions; Firewall Manager protects them automatically
- **Centralized DDoS protection** — Shield Advanced subscription + protections across the org for all current & future resources
- **Guardrails on security groups** — prevent overly-permissive rules anywhere in the org (content-audit SG policies)
- **Pairing with AWS WAF** for org-wide managed-rule enforcement ("first/last rule groups")

## 4. When NOT to use

- **Single account / single region, hand-managed resources** → just configure WAF/Shield/SG directly (Firewall Manager adds org overhead)
- **Application-layer policies without Organizations** — Firewall Manager *requires* AWS Organizations + Config
- **Per-team flexibility everywhere** — heavily-customized per-app WAF tuning conflicts with central enforcement
- **You don't need multi-account centralization** — plain AWS WAF / AWS Shield Advanced / Security Groups are simpler

## 5. Important features

- **Central policy engine across AWS Organizations** — one admin account, auto-deploy to existing + future resources/accounts
- **Policy types** — AWS WAF (Web ACL + managed rules), **Shield Advanced**, **VPC Security Groups** (common/content-audit/usage-audit), **AWS Network Firewall**, **Route 53 Resolver DNS Firewall**, **NACLs**, **third-party firewalls**
- **Hierarchical WAF enforcement** — centrally-held **first & last rule groups**; account owners add rules in between
- **Automatic Shield Advanced subscription** for all/member accounts + auto-subscribe of new org accounts; centralized DDoS monitoring
- **Compliance reporting & auto-remediation** of non-compliant resources; **Security Hub integration**
- **Scoping** by account, resource type, and tags; managed lists; multi-Region policy application (per policy/Region)

## 6. Limitations

- **Prerequisites** — AWS Organizations, AWS Config enabled, Firewall Manager delegated admin, and underlying services (e.g., Shield Advanced subscription for Shield policies)
- **Charges** are on the *underlying* services (AWS WAF, Shield Advanced, Config, Network Firewall) + FMS itself
- **SG policies don't support SGs shared through AWS RAM**; shared-VPC nuances apply
- Central enforcement can **conflict with per-app custom rules** (workable via hierarchical WAF, but planning needed)
- WAF Classic Web ACLs must be migrated to AWS WAF (new API) to be managed

## 7. Trade-offs

- **Firewall Manager vs AWS WAF alone** — central multi-account org-wide deployment/remediation/Shield subscription (FMS) vs configuring WAF Web ACLs per account/app yourself (single-account or the WAF console)
- **vs Shield Advanced alone** — org-wide auto-subscription & centralized DDoS management (FMS) vs per-resource manual Shield Advanced protection
- **vs Config/SCPs** — FMS *deploys & remediates* firewall resources; SCPs/Config **restrict/audit** configurations (complementary — many run SCPs to *require* FMS protections)
- **FMS vs direct SG management** — baseline enforcement at org scale vs per-team SG control (content-audit policies bridge the gap)

## 8. Architecture

```
Org-wide minimum-security baseline:
  FMS delegated admin (dedicated security account)
  ├─ WAF policy: Managed rules + Anti-DDoS rule group as LAST group on all ALBs/CloudFront/API GW
  ├─ Shield Advanced policy: subscribe ALL accounts (cost $3k/mo org-wide), auto-protect EIPs/ALBs/CloudFront
  ├─ Network Firewall policy: firewalls + stateful rules in every VPC (new VPCs get them automatically)
  └─ SG content-audit: deny "0.0.0.0/0:22 from the internet" anywhere in the org
  Non-compliant → auto-remediate + findings to Security Hub
```

## 9. SAA-C03 Perspective

- **"Centrally enforce firewall rules (WAF/Shield/SG/Network Firewall/DNS Firewall) across many AWS accounts"** → **AWS Firewall Manager**
- **"Protect resources automatically even as new accounts/VPCs/resources are added (Organizations)"** → **Firewall Manager**
- **"Apply managed WAF rules organization-wide / subscribe all accounts to Shield Advanced centrally"** → FMS
- **"Let app teams add WAF rules but enforce core rules centrally"** → FMS hierarchical WAF (first/last rule groups)
- **"Single account / just one app"** → configure **AWS WAF / Shield Advanced** directly

Exam traps: "Firewall Manager works without Organizations" → **no, it requires Organizations (and Config)**; "FMS replaces AWS WAF" → **no, it *deploys* WAF/Shield/SG/NFW rules centrally**; "FMS runs per account independently" → **no, one delegated administrator for the whole org, auto-applied**; "security teams hand-manage every new account" → **FMS auto-protects new accounts as they join**.