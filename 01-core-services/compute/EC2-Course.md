# EC2 Course  Elastic Compute Cloud

## 1. Purpose

EC2 (Elastic Compute Cloud) is AWS's service for **renting virtual servers** in the cloud. Instead of buying and maintaining physical hardware, you launch "instances" (VMs) that run your operating system and applications, paying only for what you use. It's the building block for most non-serverless AWS architectures.

## 2. How it works

When you launch an instance you choose:

- **AMI**  the machine image = OS + pre-installed software template
- **Instance type**  the hardware mix of vCPU / memory / storage / network
- **Networking**  VPC, subnet, public/private IP
- **Security Groups**  virtual firewall rules (allow-only, stateful)
- **Key pair**  SSH/WinRM credentials
- **IAM role**  permissions for the instance to call AWS services (no hardcoded keys)
- **EBS volume or instance store**  storage

Billing starts when the instance **runs** and stops when **stopped/terminated** (EBS volumes bill separately).

```
Users → Internet → Security Group → EC2 instance (in a subnet) → App → Database
```

## 3. When to use

- Need full control over the OS, kernel, installed software
- Porting an existing on-prem VM to AWS (lift-and-shift)
- Custom/legacy/server-bound applications
- Workloads needing specific CPU, RAM, GPU, or networking
- Persistent, always-on applications (web/app servers, batch, monolithic apps)
- GPU/ML training (P, Inf, Trainium instances), HPC (high-perf workloads)

## 4. When NOT to use

- If a managed service already solves it with less overhead:
  - **Lambda**  short, event-driven, spiky-traffic, no server management
  - **RDS/Aurora**  you need a database without patching/admin
  - **ECS/EKS on Fargate**  containers without managing EC2
  - **Elastic Beanstalk**  simple deploy of web apps
  - **Glue/Athena/Redshift**  batch analytics instead of custom servers
- Rule: if you don't need server-level control, choose managed/serverless.

## 5. Important features

- **Instance types** (pick by workload):
  - `T`  burstable, low/steady baseline (development, web servers)
  - `M`  general purpose (balanced CPU/memory)
  - `C`  compute optimized (compute-heavy, batch, ML training)
  - `R/X`  memory optimized (in-memory caches, databases)
  - `I/D/H`  storage optimized (high I/O, big data)
  - `P/G/Inf/Trn`  GPU / ML (AWS Inferentia, Trainium)
  - Graviton/ARM (`g` suffix)  better price/performance on most workloads
- **Purchasing options**:
  - On-Demand  pay per second, flexible, no commitment
  - Reserved / Savings Plans  1–3 yr commitment, up to ~72% cheaper
  - Spot  up to 90% off, capacity can be reclaimed (interruptible workloads)
  - Dedicated Hosts/Instances  for licensing/compliance
- **EBS**  persistent block storage (detach/reattach, snapshot to S3)
- **Amazon Machine Images (AMIs)**  golden images for fast launches
- **Auto Scaling + ELB**  scale instance count, spread traffic
- **Placement Groups**  Cluster (low latency), Spread (HA), Partition (fault isolation)
- **Elastic IP**  static public IPv4 (you can reassign quickly on failover)
- **Instance lifecycle**  pending → running → stopping → stopped → terminated (hibernation preserves RAM to EBS)
- **Instance store**  fast ephemeral NVMe storage (data lost on stop/terminate)
- **Metadata service (IMDS)**  instance fetches its own config/credentials
- **Elastic Fabric Adapter (EFA)**  low-latency HPC networking

## 6. Limitations

- **You manage the OS**  patching, security, maintenance (your half of shared responsibility)
- **Not elastic by default**  no auto-scaling unless you configure it
- **Operational overhead**  more than managed/serverless services
- **Instance failure is possible**  single EC2 = single point of failure (use ASG across AZs)
- **Account limits**  soft limits on vCPU per family/region (can request increases)
- **Instance store is temporary**  lose data on stop/terminate
- **Regional/AZ-bound**  an instance lives in one specific AZ
- **Cost runs while running**  idle instances still bill

## 7. Trade-offs

- **EC2 vs Lambda**: control & long-running vs serverless & auto-scaled, spiky-free
- **On-Demand vs Reserved vs Spot**: flexibility vs cost vs interruption risk
- **EBS vs Instance Store**: persistent vs fast/ephemeral
- **Bigger instance vs more instances (scale up vs scale out)**: single large vs many small, HA requires scale-out
- **Graviton (ARM) vs x86**: ~40% better price/performance but app must be ARM-compatible
- **Compute savings vs memory cost**: right-size instance type for the workload ratio

## 8. Architecture

Reference pattern for a resilient EC2 architecture:

```
Internet → ALB → [ASG: EC2 (az-a) | EC2 (az-b)] → RDS/Aurora (Multi-AZ)
            ↘ (SGs only allow ALB) 
```

- Multi-AZ Auto Scaling Group for HA and cost control
- ALB as single entry point (health-checks, terminates TLS)
- Security Groups: least privilege (ALB→app→db chain)
- IAM roles instead of access keys for instance credentials
- EBS snapshots / AMIs for backup and DR
- Spot instances (with mixed-instances policy) for non-critical capacity
- For serverless-first workloads prefer Lambda + API Gateway + SQS/DynamoDB

## 9. SAA-C03 Perspective

The exam tests **EC2 decision-making**, not memorization. Know:

- **Instance families** and when to choose each (T/M/C/R/X/I/P/G)
- **Purchasing options**  scenario questions: pick Spot for fault-tolerant/cheap (batch, ML), On-Demand for unpredictable, Reserved/Savings Plans for steady 24/7
- **ASG + ELB**  target tracking scaling, cooldowns/warm-up time, health checks, AZ rebalancing
- **Placement groups**  Cluster vs Spread vs Partition scenarios
- **EC2 vs Lambda vs ECS/Fargate vs Beanstalk**  pick the right compute per scenario
- **EBS vs instance store**  durability, snapshot, throughput vs IOPS
- **Hibernation**  stop/resume while preserving in-memory state (see sample question trend)
- **IAM roles** for instances, and **SG/NACL** flow for securing the app tier
- **Dedicated hosts** for licensing **Elastic IP reassignment** as DR failover tool

Frequent exam trap: they ask you to *select a compute service*  answer EC2 only when server-level control or a specific CPU/GPU requirement is stated.