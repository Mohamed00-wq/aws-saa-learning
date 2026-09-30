# AWS SAA Learning

Personal knowledge base for preparing the **AWS Certified Solutions Architect – Associate (SAA-C03)** exam, built as a linked set of Markdown notes rather than a single linear document.

The guiding idea: an exam scenario rarely asks "what does EC2 do?", it asks *"given these requirements, which combination of services is right?"*. So the notes are organised so you can move from a **service** to the **architecture principle** it serves, then to a **hands-on lab** that proves it.

Everything here is study material and personal notes — treat AWS's own documentation as the source of truth when something looks stale.

## Repository Layout

| Folder | Contents | Status |
|---|---|---|
| [`01-core-services/`](01-core-services/) | 65 service notes for the AWS services that appear in most SAA scenarios | Complete |
| [`02-specific-services/`](02-specific-services/) | 17 notes on narrower, situational services | Complete |
| [`03-saa-architecture/`](03-saa-architecture/) | 12 design-principle notes (the Well-Architected pillars and core concepts) | Complete |
| [`04-architecture-labs/`](04-architecture-labs/) | 23 hands-on labs with CLI commands, architecture diagrams and policies | Scaffolded — every file is an empty placeholder |

```
.
├── 01-core-services/       # Services you must know cold
│   ├── compute/  container/  databases/  integration/
│   ├── Networking/  observability/  security/  storage/
├── 02-specific-services/   # Services that show up in narrower scenarios
│   ├── analytics/  compute/  devops/  integration/
│   ├── management/  media-ai/  migration/
├── 03-saa-architecture/    # Design principles and trade-offs
└── 04-architecture-labs/    # Lab scaffolding (README / architecture / commands)
    ├── 01-secure-architectures/
    ├── 02-resilient-architectures/
    ├── 03-high-performing-architectures/
    └── 04-cost-optimized-architectures/
```

## How to Use This Repo

**Scenario-driven** (recommended — this is how the exam works):

1. Read the scenario and extract the requirements: availability target, data durability, latency, cost, compliance, expected load pattern.
2. Jump to the relevant principle in [`03-saa-architecture/`](03-saa-architecture/) — e.g. *decoupling* for spiky or asynchronous workloads, *caching* for read-heavy latency, *elasticity* for unpredictable demand.
3. Use the principle note's **"AWS Services That Work With This Concept"** section to pick candidate services.
4. Read the candidate service notes in [`01-core-services/`](01-core-services/) and compare them via the **Trade-offs** section.
5. Validate the pattern in the matching lab under [`04-architecture-labs/`](04-architecture-labs/).

**Service-driven** — when you want to revise a specific service end to end, work through its note top to bottom.

## Note Templates

Each note follows a fixed skeleton so you always know where to look. Three variants are in use; new notes should pick the one matching their folder.

### Service notes (`01-core-services/`) — the 9-section template

Used by 61 of the 65 core-service notes.

```markdown
# <Service> Course  <subtitle>

## 1. Purpose          # What it is and the problem it solves
## 2. How it works     # Mechanics, plus a plain-text request/data-flow diagram
## 3. When to use      # Scenarios that call for this service
## 4. When NOT to use  # The managed/serverless alternative that fits better
## 5. Important features
## 6. Limitations
## 7. Trade-offs       # Service vs service, e.g. EC2 vs Lambda
## 8. Architecture     # Reference pattern diagram + design notes
## 9. SAA-C03 Perspective  # How the exam tests this, and common distractors
```

Worked example: [`EC2-Course.md`](01-core-services/compute/EC2-Course.md).

### Specific-service notes (`02-specific-services/`) — the compact template

Used by all 17 notes in this folder.

```markdown
# <Service>  <subtitle>

## Purpose
## Main use cases
## Key features
## When to use
## Important limitation
## SAA relevance   # Exam phrasing -> correct answer mapping
```

Worked example: [`CloudFormation.md`](02-specific-services/devops/CloudFormation.md).

### Architecture notes (`03-saa-architecture/`) — the concept template

```markdown
# <Concept>

## 1. Purpose
## 2. How It Works
## 3. Trade-offs                     # What you gain vs what you sacrifice
## 4. AWS Services That Work With This Concept
## 5. When to Use
## 6. When NOT to Use
```

Worked example: [`well-architected-framework.md`](03-saa-architecture/well-architected-framework.md).

> A handful of notes (`AWS-X-Ray`, `AWS-Backup`, `AWS-Snow-Family`, `FSx`) use a looser per-service structure with sections such as *What it is*, *Core concepts*, *Pricing*, *Related services*, *Key gotchas* and *Exam domains*. These are legacy shapes — prefer the three templates above for anything new.

## 01 — Core Services

The services that show up in the majority of SAA scenarios. Start here.

### Compute
[EC2](01-core-services/compute/EC2-Course.md) · [AMIs](01-core-services/compute/AMIs-Course.md) · [Auto Scaling](01-core-services/compute/Auto-scaling-Course.md) · [Elastic Load Balancing](01-core-services/compute/ELB-Course.md) · [Lambda](01-core-services/compute/Lambda-Course.md) · [AWS Batch](01-core-services/compute/AWS-Batch.md) · [Elastic Beanstalk](01-core-services/compute/Beanstalk.md)

### Containers
[ECS](01-core-services/container/ECS-Course.md) · [EKS](01-core-services/container/EKS-Cloud-Course.md) · [ECR](01-core-services/container/ECR-Course.md)

### Databases
[RDS](01-core-services/databases/RDS-Course.md) · [Aurora](01-core-services/databases/Aurora-Course.md) · [DynamoDB](01-core-services/databases/DynamoDB-Course.md) · [ElastiCache](01-core-services/databases/ElastiCache-Course.md) · [MemoryDB](01-core-services/databases/MemoryDB.md) · [Redshift](01-core-services/databases/Redshift-Course.md) · [OpenSearch](01-core-services/databases/OpenSearch-Service.md) · [Neptune](01-core-services/databases/Neptune.md) · [DocumentDB](01-core-services/databases/DocumentDB.md) · [Keyspaces](01-core-services/databases/Keyspaces.md) · [QLDB](01-core-services/databases/QLDB.md) · [DMS](01-core-services/databases/DMS-Course.md)

### Integration & messaging
[API Gateway](01-core-services/integration/API-Gateway-Course.md) · [SQS](01-core-services/integration/SQS-Course.md) · [SNS](01-core-services/integration/SNS-Course.md) · [EventBridge](01-core-services/integration/EventBridge-Course.md) · [Kinesis](01-core-services/integration/Kinesis-Course.md) · [Step Functions](01-core-services/integration/Step-Functions-Course.md) · [Amazon MQ](01-core-services/integration/Amazon-MQ.md) · [Amazon MSK](01-core-services/integration/Amazon-MSK.md) · [AppSync](01-core-services/integration/AppSync.md)

### Networking
[VPC](01-core-services/Networking/VPC-Course.md) · [Route 53](01-core-services/Networking/Route53-Course.md) · [CloudFront](01-core-services/Networking/CloudFront-Course.md) · [AWS Global Accelerator](01-core-services/Networking/AWS-Global-Accelerator.md) · [Hybrid Connectivity](01-core-services/Networking/Hybrid-Connectivity-Course.md)

### Observability
[CloudWatch](01-core-services/observability/CloudWatch-Course.md) · [CloudTrail](01-core-services/observability/CloudTrail-Course.md) · [X-Ray](01-core-services/observability/AWS-X-Ray.md)

### Security, identity & compliance
[IAM](01-core-services/security/IAM-Course.md) · [KMS](01-core-services/security/KMS-Course.md) · [Secrets Manager](01-core-services/security/Secrets-Manager-Course.md) · [Cognito](01-core-services/security/Cognito-Course.md) · [WAF](01-core-services/security/WAF-Course.md) · [Shield](01-core-services/security/Shield-Course.md) · [GuardDuty](01-core-services/security/GuardDuty-Course.md) · [Security Hub](01-core-services/security/Security-Hub-Course.md) · [Inspector](01-core-services/security/AWS-Inspector.md) · [Macie](01-core-services/security/Amazon-Macie.md) · [Detective](01-core-services/security/Amazon-Detective.md) · [CloudHSM](01-core-services/security/CloudHSM.md) · [Directory Service](01-core-services/security/AWS-Directory-Service.md) · [Firewall Manager](01-core-services/security/AWS-Firewall-Manager.md) · [Audit Manager](01-core-services/security/AWS-Audit-Manager.md) · [Artifact](01-core-services/security/AWS-Artifact.md)

### Storage
[S3](01-core-services/storage/S3-Course.md) · [EBS](01-core-services/storage/EBS-Course.md) · [EFS](01-core-services/storage/EFS-Course.md) · [FSx](01-core-services/storage/FSx.md) · [Storage Gateway](01-core-services/storage/Storage-Gateway-Course.md) · [AWS Backup](01-core-services/storage/AWS-Backup.md) · [DataSync](01-core-services/storage/AWS-Data-Sync.md) · [Snow Family](01-core-services/storage/AWS-Snow-Family.md) · [Transfer Family](01-core-services/storage/AWS-Transfer-Family.md) · [AppFlow](01-core-services/storage/Amazon-AppFlow.md)

## 02 — Specific Services

Narrower services that appear in targeted scenarios or as distractors.

### Analytics
[Glue](02-specific-services/analytics/AWS-Glue.md) · [Athena](02-specific-services/analytics/Athena.md) · [Lake Formation](02-specific-services/analytics/Lake-Formation.md) · [Data Exchange](02-specific-services/analytics/AWS-Data-Exchange.md) · [Compute Optimizer](02-specific-services/analytics/AWS-Compute-Optimizer.md)

### DevOps & IaC
[CloudFormation](02-specific-services/devops/CloudFormation.md) · [CDK](02-specific-services/devops/CDK.md) · [Service Catalog](02-specific-services/devops/Service-Catalog.md)

### Media & AI
[ML Managed Services](02-specific-services/media-ai/ML-Managed-Services.md) · [AI Dev Tools](02-specific-services/media-ai/AI-Dev-Tools.md) · [MediaConvert](02-specific-services/media-ai/AWS-MediaConvert.md) · [Elastic Transcoder](02-specific-services/media-ai/Elastic-Transcoder.md) · [Device Farm](02-specific-services/media-ai/Device-Farm.md)

### Application build & delivery
[Amplify](02-specific-services/compute/AWS-Amplify.md) · [ACM](02-specific-services/integration/ACM.md)

### Management & migration
[Health Dashboards](02-specific-services/management/Health-Dashboards.md) · [Migration Hub](02-specific-services/migration/AWS-Migration-Hub.md)

## 03 — Architecture Concepts

Design principles. These are what turn "which service?" into "which *design*?" — and they map almost one-to-one onto the six Well-Architected pillars.

Start with the [Well-Architected Framework](03-saa-architecture/well-architected-framework.md); it is the umbrella note and links out to the others.

| Concept | Note |
|---|---|
| Operational Excellence | [operational-excellence.md](03-saa-architecture/operational-excellence.md) |
| Security | [security.md](03-saa-architecture/security.md) |
| Scalability | [scalability.md](03-saa-architecture/scalability.md) |
| Elasticity | [elasticity.md](03-saa-architecture/elasticity.md) |
| Fault tolerance | [fault-tolerance.md](03-saa-architecture/fault-tolerance.md) |
| High availability | [high-availability.md](03-saa-architecture/high-availability.md) |
| Disaster recovery | [disaster-recovery.md](03-saa-architecture/disaster-recovery.md) |
| Decoupling | [decoupling.md](03-saa-architecture/decoupling.md) |
| Performance efficiency | [performance.md](03-saa-architecture/performance.md) |
| Cost optimization | [cost-optimization.md](03-saa-architecture/cost-optimization.md) |
| Sustainability | [sustainability.md](03-saa-architecture/sustainability.md) |

The reliability pillar is covered by three separate notes because SAA scenarios distinguish them: **fault tolerance** (no downtime, built-in redundancy), **high availability** (recovers quickly, measured by uptime SLA), and **disaster recovery** (regional failures, measured by RTO/RPO).

## 04 — Architecture Labs

Each lab is a folder containing four files:

| File | Purpose |
|---|---|
| `README.md` | Scenario brief, requirements, and what the lab demonstrates |
| `architecture.md` | The target design, with a diagram and the reasoning behind each choice |
| `commands.md` | The AWS CLI steps to build and tear the architecture down |
| `*.json` | Policy, trust or filter-policy documents where the scenario needs one |

> **Status:** the structure and scenario list exist, but no lab has been written yet. Every `README.md`, `architecture.md`, `commands.md` and `.json` file in this tree is **empty** (0 bytes), and `01-secure-architectures/data-protection/` is an empty directory that git will not track until it contains a file. Treat this folder as the next thing to fill in.

### 01 — Secure architectures
`data-protection` · `iam` · `kms` · `nacl` · `security-groups` · `vpc-security`

### 02 — Resilient architectures
`backup-recovery` · `decoupling` · `fault-tolerance` · `high-abailability` · `messaging` · `multi-az`

### 03 — High-performing architectures
`auto-scaling` · `caching` · `cdn` · `perfomence` · `scalability` · `storage-database-selection`

### 04 — Cost-optimized architectures
`cost-aware-architecture` · `pricing-models` · `right-sizing` · `servless` · `storage-optimization`

> Directory names `perfomence` and `servless` are typos carried over from the AWS course outline. Left as-is to match the source material — rename if you'd rather keep the tree clean.

## Conventions

- **Markdown only** — no build step, no tooling, no dependencies. Open the files in any editor or browse them on GitHub.
- **Plain-text ASCII diagrams** inside fenced code blocks, rather than images or Mermaid, so they stay readable in a terminal and diff cleanly.
- **`→` for flow, `↘` for branches** in architecture sketches.
- **Bolded term followed by an explanation** in lists, e.g. `**Multi-AZ** — two or more AZs serving the same app`.
- **Troubleshooting habits baked in**: notes state what a service *cannot* do, so you can eliminate it from a scenario quickly.
- **`-Course.md` suffix** marks the first-pass core-service notes; newer notes drop it. Naming is not fully consistent yet.

## Security Notes for the Labs

`.gitignore` already excludes the common sources of accidental credential commits: `.aws/`, `*.pem`, `*.env`, `credentials` and `config`. Keep it that way.

- Never commit account IDs, ARNs with real account numbers, or access keys. Use `<ACCOUNT_ID>` placeholders.
- Lab cleanup is part of the lab — the snapshots, EBS volumes, NAT gateways and Elastic IPs created by these patterns are billed continuously. Tear them down.
- Prefer a dedicated sandbox account with a budget alarm over your primary account.

## Contributing

This is a personal study repo, so the workflow is informal: write a note when you hit a gap, keep the section skeleton, and link new architecture concepts back to the service notes that support them.


## License

[MIT](LICENSE) © 2026 Mohamed
