# EC2 — Elastic Compute Cloud 

![EC2](/images/icons/arch/Arch_Amazon-EC2_64.svg)

## What it is

EC2 is the foundational compute building block in AWS — it provides resizable virtual servers (instances) in the cloud. If S3 is where data lives at rest and DynamoDB is where structured data lives, EC2 is where you run code that needs full control over the operating system, networking, and software stack.

EC2 is not the answer for everything (Lambda, ECS, Fargate, and Beanstalk are alternatives for specific workloads), but it remains the most flexible compute option and the one most other services integrate with. Nearly every architecture on the exam involves EC2 in some capacity, whether directly or as the target behind an ELB.

## Instances

- Always launched **from an AMI**; identified by an immutable **instance ID** (`i-0abc...`).
- After launch, the instance is independent of the AMI — changes don't modify the AMI, deleting the AMI doesn't affect running instances.

## Instance types

| Family | Use | Examples |
|---|---|---|
| **General (t, m)** | Balanced | t3.micro, m5.large |
| **Compute (c)** | CPU-heavy | c5.large |
| **Memory (r, x, z)** | In-memory DBs | r5.large |
| **Storage (i, d, h)** | High local I/O | i3.large |
| **Accelerated (p, g, trn, inf)** | GPU/ML | p4d.24xlarge |

- **Size** (nano→xlarge) sets vCPUs and RAM.
- **t-series** use a CPU **credit** model — burst above baseline using credits, throttled when exhausted; unlimited mode bursts for extra cost.

## Networking

- Every instance gets a primary **private IP** + primary **ENI**.
- **Public IP** is auto-assigned, released on stop/stop and changes each restart.
- **Elastic IP (EIP)**: static public IP, persists across stop/start, reassignable. **Charged when idle** (allocated but not attached) — key cost trap.

## Security Groups vs NACLs

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Additive (Allow only, no Deny) | Numbered, first match wins, explicit **Deny** supported |

On the exam: SG for instance-level access control; NACL when subnet-level stateless filtering / explicit deny is mentioned.

## ENIs

Virtual network card in a VPC. Primary ENI (eth0) can't be detached and is deleted on termination. Secondary ENIs are detachable (failover/management). **AZ-scoped** — can't attach across AZs.

## Placement groups

- **Cluster**: same rack, single AZ → lowest latency/highest throughput for HPC. All fail together if rack fails.
- **Spread**: distinct racks → max 7 instances per AZ, critical instances that must not fail together.
- **Partition**: logical partitions on rack floor, replicas across failure domains (HDFS, Kafka, Cassandra).

## Hibernation

Saves RAM to the **encrypted root EBS volume** to resume in place (processes/net connections intact). Requires EBS-backed, encrypted root volume.

## User Data

Script that runs once at first boot. Stored unencrypted (use Secrets Manager/SSM for secrets). Max **16 KB**.

## Lifecycle & billing

```
pending → running → stopping → stopped → pending → running
                       ↓
                  terminating → terminated
```

- **stopped**: no compute charge, but EBS volumes still billed; instance store data lost.
- **terminated**: root volume deleted by default (`DeleteOnTermination=true`); **non-root volumes preserved by default**.

## IMDS (169.254.169.254)

Provides instance metadata (type, AZ, IAM role creds, user data). **IMDSv2** requires a `PUT` token first — prevents **SSRF** credential theft. If a question mentions SSRF credential theft, enable IMDSv2.

## Exam domains

- [x] **Secure (30%)** — SGs, NACLs, IMDSv2, instance profile roles
- [x] **Resilient (26%)** — spread groups, lifecycle, hibernation
- [x] **High-Performing (24%)** — instance type selection, cluster groups for HPC
- [x] **Cost-Optimized (20%)** — pricing models, idle EIP charges, t-series credits

## Key gotchas

1. SGs are **stateful**, NACLs are **stateless** — must allow response explicitly in NACL
2. SGs have no Deny rules; NACLs do
3. Deleting an AMI **doesn't delete snapshots**
4. **Idle EIPs** incur charges
5. **Root volume** deleted on termination, non-root preserved (by default)
6. **Instance store** data lost on stop/terminate
7. Hours of high CPU in t-series unlimited mode can exceed a fixed-size instance cost
8. **Hibernation requires encrypted root volume**
9. SGs can reference other SGs ("allow 3306 from sg-app")
10. Enable **IMDSv2** to stop SSRF credential theft


## Related services

- [[AMI]] — template instances are launched from
- [[EBS]] — persistent block storage
- [[Auto-scaling]] — launch/terminate on demand
- [[ELB]] — distributes traffic across instances
- [[VPC]] — subnets, routing context
- [[IAM]] — roles via instance profiles
- [[S3]] — AMIs, user data, app assets
