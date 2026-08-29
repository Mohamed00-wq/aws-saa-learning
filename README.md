# AWS SAA-C03 Learning Journey

Hands-on study log for the AWS Certified Solutions Architect – Associate (SAA-C03) exam.

Every service gets its own markdown file with: what it is, key commands, how it works, lab notes, and exam-style practice questions. See [TEMPLATE.md](./TEMPLATE.md) for the format used across all files.

## Exam domains
| Domain | Weight |
|---|---|
| Design Secure Architectures | 30% |
| Design Resilient Architectures | 26% |
| Design High-Performing Architectures | 24% |
| Design Cost-Optimized Architectures | 20% |

## Roadmap & progress

- [ ] **01 · Foundation** — IAM, EC2, AMIs, EBS, EC2 Pricing Models, Auto Scaling, ELB
- [ ] **02 · Storage** — S3, EFS, FSx, Storage Gateway, AWS Backup, Snow Family
- [ ] **03 · Databases** — DynamoDB, Keyspaces, RDS, Aurora, DocumentDB, Neptune, ElastiCache, MemoryDB, Redshift
- [ ] **04 · Serverless & Compute** — Lambda, API Gateway, Step Functions, Elastic Beanstalk, ECS, ECR, EKS, Batch, Compute Optimizer
- [ ] **05 · Messaging & Integration** — SNS, SQS, Amazon MQ, EventBridge, AppSync, AppFlow, Amplify
- [ ] **06 · Networking (VPC)** — VPC, Route53, CloudFront, Global Accelerator
- [ ] **07 · Monitoring & Governance** — CloudWatch, CloudTrail, Service Catalog, Health Dashboards, AWS Artifact
- [ ] **08 · Security** — KMS, ACM, Cognito, Secrets Manager, Firewall Manager, Shield, WAF, CloudHSM, GuardDuty, Detective, Inspector, Macie, Security Hub, Audit Manager, Directory Service
- [ ] **09 · Migration & Transfer** — DMS, Migration Hub, Data Sync, Transfer Family
- [ ] **10 · Analytics & ML** — Kinesis, MSK, Athena, Glue, Lake Formation, OpenSearch, Data Exchange, ML Managed Services, AI Dev Tools, QLDB, Elastic Transcoder, MediaConvert, Device Farm

## Applied project

Alongside the notes, every concept feeds into one running project that builds up in stages:

```
IAM → S3 + DynamoDB → Lambda → API Gateway → Serverless App → VPC + Networking → High Availability → Hybrid Cloud
```



## License

MIT — see [LICENSE](./LICENSE).