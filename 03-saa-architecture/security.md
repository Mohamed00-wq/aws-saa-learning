# Security

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Security protects your data, systems, and access from unauthorized use, disclosure, and attack. The goal is **defense in depth** and **least privilege**, so that even if one layer is breached, other layers still protect you and the blast radius is minimal. In the SAA exam it is the largest domain (30%, Design Secure Architectures). Cloud is a shared responsibility model, AWS secures the infrastructure, you secure what you put on it, so architecture choices for identity, network, encryption, and monitoring are yours.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - Every request is authenticated (who are you?) and authorized (what may you do?) before it touches data. Network boundaries, firewalls, and security groups filter what can even reach your resources. Data is encrypted at rest and in transit. Every action is logged and monitored.
- Why does it work?
  - Multiple independent controls mean an attacker must defeat each one. Even a compromised identity is limited by least-privilege policies and network segmentation.
- What is the main idea behind it?
  - Think in layers: **identity (IAM, Cognito, identity federation)** -> **network (VPC, security groups, NACLs, WAF, firewalls)** -> **data (KMS, S3 encryption, TLS)** -> **monitoring (CloudTrail, GuardDuty, Security Hub, Config)**. Apply **zero trust**: no request is trusted by default, least privilege everywhere, encrypt everything, and detect what you cannot stop.

## 3. Trade-offs

What do you gain?
- Confidentiality, integrity, and availability of your data and systems
- Compliance with regulations (GDPR, HIPAA, PCI) and audit trails to prove it
- Early detection of intrusions through monitoring and automation

What do you sacrifice?
- **Operational friction**  more IAM roles, network controls, and approvals slow down development
- **Cost**  KMS keys, WAF, centralized logging, and dedicated tools add spend
- **Performance**  encryption and WAF inspection add latency
- Overly aggressive controls (blocking everything, perfect per-request authz) can hurt usability and create engineering delay

## 4. AWS Services That Work With This Concept

- **AWS IAM**  users, roles, groups, policies, MFA, and least-privilege access control
- **Amazon Cognito**  authentication for apps (social, username/password, MFA, identity federation)
- **Amazon VPC**  subnets, security groups, NACLs, gateways, VPC peering, and private networking
- **AWS WAF / AWS Shield**  web application firewall and DDoS protection
- **AWS KMS / CloudHSM**  encryption key management and hardware-backed keys
- **Amazon S3 (versioning, encryption, block public access)**  secure object storage
- **AWS Secrets Manager / Parameter Store**  safe storage of credentials
- **AWS CloudTrail**  API activity audit log across the account
- **Amazon GuardDuty**  threat detection on malicious/unauthorized behavior
- **AWS Security Hub**  central security posture and findings aggregation
- **AWS Config**  continuous configuration compliance and drift detection
- **AWS Network Firewall / AWS Firewall Manager**  perimeter protection and central policy management
- **AWS Organizations / SCPs**  guardrails across all accounts

## 5. When to Use

Use this concept when:
- You hold any sensitive or customer data (PII, financial, health), so encryption and access control are mandatory
- Regulatory or compliance obligations require audits, least privilege, and monitoring
- The application is internet-facing and needs WAF, DDoS protection, and strong identity
- You grant access to third parties, contractors, or many internal teams where scoped IAM roles and boundaries matter
- Exam questions mention "least privilege," "defense in depth," "encrypt everything," "SCP guardrails," "shared responsibility," or "detect and respond to threats"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- A hero/root account with broad permissions is being used casually (never production, rotate and lock it down immediately)
- You are considering skipping encryption for "internal only" traffic, there are few real exceptions and costs are low
- The control (such as a WAF with huge rule packs on a tiny app) costs more in latency and budget than the risk it mitigates
- You are putting secrets in code, environment variables exposed in logs, or committing .env files, there is no valid reason it stays
- You need to ship a quick internal prototype with no sensitive data, still use baseline controls (IAM + default encryption), but avoid over-engineering the full defense-in-depth stack