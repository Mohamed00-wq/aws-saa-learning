# EC2 — Elastic Compute Cloud 

## What it is

EC2 is the foundational compute building block in AWS — resizable virtual servers (instances) in the cloud. It provides full control over OS, networking, and software stack. Nearly every architecture on the exam involves EC2 directly or as the target behind an ELB.

## Instances & types

- Launched **from an AMI**; immutable **instance ID** (`i-0abc...`). Independent of AMI after launch.

| Family | Use | Examples |
|---|---|---|
| **General (t, m)** | Balanced | t3.micro, m5.large |
| **Compute (c)** | CPU-heavy | c5.large |
| **Memory (r, x, z)** | In-memory DBs | r5.large |
| **Storage (i, d, h)** | High local I/O | i3.large |
| **Accelerated (p, g, trn, inf)** | GPU/ML | p4d.24xlarge |

- **t-series**: CPU **credit** model — burst above baseline, throttled when exhausted; unlimited mode for extra cost.

## Networking

- Primary **private IP** + primary **ENI**. **Public IP** auto-assigned, changes on restart.
- **Elastic IP (EIP)**: static public IP, persists across stop/start. **Charged when idle**.

## Security Groups vs NACLs

| | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| State | **Stateful** | **Stateless** |
| Rules | Allow only, no Deny | Numbered, first match, explicit **Deny** |

## ENIs, Placement groups, Hibernation

- **ENI**: AZ-scoped virtual network card. Primary deleted on termination; secondary detachable.
- **Cluster**: same rack, lowest latency (HPC). **Spread**: distinct racks, max 7/AZ. **Partition**: logical partitions (HDFS, Kafka).
- **Hibernation**: saves RAM to **encrypted root EBS** — resume with processes intact.

## User Data & Lifecycle

- Script runs once at first boot. Max **16 KB**. Stored unencrypted.

```
pending → running → stopping → stopped → pending → running
                       ↓
                  terminating → terminated
```

- **stopped**: no compute charge, EBS still billed. **terminated**: root volume deleted, non-root preserved.

## IMDS (169.254.169.254)

**IMDSv2** requires a `PUT` token — prevents **SSRF** credential theft. Enable when SSRF is mentioned.

## Exam domains

- [x] **Secure (30%)** — SGs, NACLs, IMDSv2, instance profile roles
- [x] **Resilient (26%)** — spread groups, lifecycle, hibernation
- [x] **High-Performing (24%)** — instance type selection, cluster groups for HPC
- [x] **Cost-Optimized (20%)** — pricing models, idle EIP charges, t-series credits

## Key gotchas

1. SGs **stateful**, NACLs **stateless** — NACL must allow response explicitly
2. SGs no Deny rules; NACLs do
3. Deleting AMI **doesn't delete snapshots**
4. **Idle EIPs** incur charges
5. **Root volume** deleted on termination, non-root preserved
6. **Instance store** data lost on stop/terminate
7. t-series unlimited mode can exceed fixed-size instance cost
8. **Hibernation requires encrypted root volume**
9. SGs can reference other SGs ("allow 3306 from sg-app")
10. Enable **IMDSv2** to stop SSRF credential theft

## Related services

- **AMI** — template instances are launched from
- **EBS** — persistent block storage
- **ASG** — launch/terminate on demand
- **ELB** — distributes traffic across instances
- **VPC** — subnets, routing context
- **IAM** — roles via instance profiles
