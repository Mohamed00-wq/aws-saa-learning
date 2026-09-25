# AWS Backup  Centralized Backup Service

## What it is

A **fully managed, centralized data-protection service** that automates backups across many AWS services from one console: define policies once, apply them to resources, get consistent scheduling, retention, and compliance. Replaces juggling per-service backup features (EBS snapshots, RDS automated backups, DynamoDB on-demand, EFS backups) with a single policy engine.

## Core concepts

- **Backup plans**  the policy. Each plan contains one or more **rules**:
  - **Schedule**: hourly, daily, weekly, or cron. Continuous backups + point-in-time recovery (PITR) for eligible resources.
  - **Backup window**: start window (default 8 hours) and completion window jobs are canceled if they don't finish in time.
  - **Lifecycle**: transition recovery points to **cold storage** after N days, and **expire/delete** after N days (up to 100 years).
  - **Target vault** + optional **cross-Region / cross-account copy**.
- **Backup vaults**  logical containers for recovery points. Encrypted with a **KMS key**, protected by **vault access policies** (who can create/restore/delete), and optionally hardened with **Vault Lock**. Can send SNS notifications on job state.
- **Backup Vault Lock**  the WORM (write-once-read-many) protection layer:

| Mode | Behavior |
|---|---|
| **Governance** | Prevents accidental/non-privileged deletion users with proper IAM privileges can still remove the lock |
| **Compliance** | After the **grace period**, immutable  cannot be altered/deleted by anyone, **including the root user or AWS** |

- **Incremental backups**: first backup is full, subsequent ones store only changes  frequent backups without full storage cost.
- **IAM role**: AWS Backup assumes a role with permissions to describe/snapshot the target resources.

## Supported resources

EC2, EBS, RDS, Aurora, **DynamoDB**, **EFS**, **FSx**, **S3**, Storage Gateway, VMware (on-prem via agent)  a single plan can target several resource types (by ARN or by **tag**).

## Security & compliance

- Vaults encrypted with the vault's **KMS key** (default `aws/backup` or a customer-managed CMK)  backups stay encrypted independently of the source resource.
- Recovery points are **immutable** and separate from source resources  deleting the EC2 instance doesn't delete its backups (retained per lifecycle).
- **Vault Lock (Compliance mode)** is the exam answer for ransomware protection and tamper-proof retention (e.g. regulator requires 7-year retention that even root can't delete).
- **AWS Backup Audit Manager**  frameworks + controls for automated compliance checks.

## Exam domains

- [x] **Secure (30%)**  KMS-encrypted vaults, vault access policies, Vault Lock governance vs compliance
- [x] **Resilient (26%)**  automated backup plans, cross-Region cross-account copies, PITR, DR/restore targets
- [x] **High-Performing (24%)**  backup windows, incremental backups, restore speed to warm vs cold tier
- [x] **Cost-Optimized (20%)**  lifecycle to cold storage, expiration/retention, per-resource plans instead of "backup everything"

## Key gotchas

1. **"Centralize/automate backups across services" ⇒ AWS Backup**  not per-service snapshot tools
2. **Vault Open ≠ Vault Lock**  locking is optional and explicitly enabled
3. **Compliance mode lock is irreversible after the grace period (min 72 hours)**  even root/AWS can't undo it pick governance if you need an escape hatch
4. **Lifecycle is applied to recovery points, not S3 objects**  vaults are not buckets S3 Lifecycle policies are a separate thing
5. **Cold storage requires 90+ days total retention**  don't expect instant deletion of cold-tiered backups
6. **Backup window: if the job misses the window, it's canceled**  size the window to your data volume
7. **Delete schedules ≠ instantly gone**  backups are retained and billed until lifecycle expiry
8. **Tag-based resource assignment**  new resources matching tags get protected automatically

## Related services

- **KMS**  encrypts every backup vault
- **Storage Gateway**  Volume Gateway volumes are backed up through AWS Backup
- **EFS / FSx / RDS / DynamoDB / EC2 / EBS / S3**  all supported source resources
- **SNS**  backup job notifications
- **Audit Manager**  compliance checks on backup coverage