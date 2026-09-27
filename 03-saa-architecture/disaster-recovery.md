# Disaster Recovery

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Disaster recovery (DR) protects against **large-scale, site-wide failures** such as the loss of an entire AWS Region, a data center disaster, or a major external outage. While high availability handles individual component failures, DR answers the question "how do we get the business running again in another location if everything here is gone?" The goal is to keep data safe and restore service within acceptable **RTO (Recovery Time Objective)** and **RPO (Recovery Point Objective)** targets.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - You copy data, applications, and infrastructure to a secondary region (backup, replication, or pre-deployed resources).
  - On disaster, you activate the backup environment and **fail over** to it.
  - When the primary region recovers, you **fail back** to it (or stay on the new region).
- Why does it work?
  - Because the secondary location is isolated from the disaster, your data and workloads survive even when the primary site does not.
- What is the main idea behind it?
  - Balance **RPO**, how much data you can afford to lose (measured in time), against **RTO**, how fast you must be back up (measured in time). Cheaper strategies recover slowly and lose more data, expensive strategies recover fast with data loss near zero.

The classic DR strategy ladder, from slowest/cheapest to fastest/most expensive:

| Strategy | RTO/RPO | How it works |
|---|---|---|
| Backup and Restore | Hours, hours | Backups (often S3/Glacier) copied to another region, rebuild + restore |
| Pilot Light | Minutes to tens of minutes | Minimal core runs in DR region (DB replicas), app servers start on demand |
| Warm Standby | Minutes | Reduced-capacity scaled-down copy runs in DR region, scaled up on failover |
| Multi-site Active-Active | Near zero | Full production runs in both regions, traffic balanced between them |

## 3. Trade-offs

What do you gain?
- Survival of region-wide outages instead of complete business loss
- Meet legal and business requirements for data retention and continuity
- Control over exactly how much data loss (RPO) and downtime (RTO) you accept

What do you sacrifice?
- **Cost**  every step up the ladder (standby, warm standby, active-active) costs more in running resources
- **Complexity**  replication pipelines, failover playbooks, and testing require real operational effort
- **Data consistency concerns**  cross-region replication is usually asynchronous, so you may lose recent writes (higher RPO)
- DR is rarely "set and forget"  runbooks and recovery tests must be exercised regularly or the plan fails when you need it

## 4. AWS Services That Work With This Concept

- **AWS Backup**  central backup management, supports cross-region backup copies
- **Amazon S3 Cross-Region Replication (CRR)**  replicate objects to another region with versioning and lifecycle
- **Amazon RDS / Aurora Cross-Region Replicas**  asynchronous read replicas ready to promote in the DR region
- **DynamoDB Global Tables**  multi-region, fully managed, active-active replication
- **Amazon EBS Snapshots / AMIs**  copy snapshots across regions to rebuild compute
- **AWS Elastic Disaster Recovery (DRS)**  continuous replication of servers (EC2 and on-prem) for fast recovery into AWS
- **Amazon Route 53**  DNS-based failover from primary to recovery region
- **S3 Glacier / Amazon S3 (backup tiers)**  low-cost long-term backup storage

## 5. When to Use

Use this concept when:
- You must survive the loss of an entire AWS Region or on-premises site
- Compliance, contracts, or the business require defined RTO and RPO targets
- Data is irreplaceable and must be protected against site loss (backup to another region)
- Exam questions mention "region failure," "disaster recovery," "RTO/RPO," "pilot light," "warm standby," or "multi-site active-active" (know the strategy ladder and its ordering by cost and speed)

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The application is not critical enough to justify paying for a second region, then backups in a different AZ or simple snapshots may be enough
- You actually only need to survive AZ-level failures, which is **high availability / fault tolerance** done cheaper within one region
- Your strict single-writer database workloads cannot tolerate asynchronous replication, so cross-region reads would go stale
- The "disaster" you are preparing for is really a slow degradation, where scaling/self-healing within the region is the right tool