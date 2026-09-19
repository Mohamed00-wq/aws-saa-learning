# AMI Course — Amazon Machine Images

## 1. Purpose

An **AMI** (Amazon Machine Image) is a **golden template** that defines every new EC2 instance: the operating system, pre-installed software, root filesystem, config, and boot metadata. You always choose an AMI when launching an instance — it's the recipe that makes instances identical, repeatable, and quick to start.

**AMI = EBS snapshot(s) + block device mapping + launch permissions + boot attributes.**

## 2. How it works

- An AMI is a pointer to one or more **EBS snapshots** plus a **block-device mapping** (which volumes attach at launch) and launch metadata
- When you launch an instance, EC2 copies the AMI's root volume from snapshots into a fresh EBS volume and boots it
- Creating an AMI from a running instance reverses this: EC2 snapshots each attached volume and registers an image
- AMIs are **regional** — each region has its own AMIs (must copy to use elsewhere)
- AMI must match the **instance type architecture**: x86 vs arm64 (Graviton), and virtualization type (modern = HVM)

```
instance (configured + apps installed) → aws ec2 create-image → AMI (snapshots)
AMI → launch → N identical instances (same OS, same apps, same config)
CopyImage → same AMI in another Region → launch there (DR)
```

## 3. When to use

- You need **many identical instances** (fleets, ASGs) — bake once, reuse forever
- **Golden image** strategy: bake security hardening, CIS benchmarks, agents (CloudWatch, SSM, EDR) so every instance starts compliant
- **Fast startup** — a fully baked AMI is launch-ready in under a minute (vs installing at boot)
- **Immutable/rolling deployments** — build new AMI version, roll ASG via Instance Refresh
- **Disaster recovery** — copy AMI to another region for regional failover
- **Custom software**, licensing, or preinstalled dependencies that are painful to script

## 4. When NOT to use

- You need flexibility per-instance and config-at-boot is fine — use **User Data** scripts instead of baking
- **Containers** (ECS/EKS/Fargate/App Runner) — the container image (ECR) is the "baked artifact", not the AMI
- Frequently changing application code — don't rebuild AMIs per deploy; bake the OS/base, deploy app code separately
- Serverless (Lambda) — nothing to image
- One-off experiments — just use an AWS-provided AMI + User Data

## 5. Important features

- **Sources**: AWS-provided, public/community, AWS Marketplace (paid + licensed OS), shared from another account, or your own
- **Create from instance** (`create-image`); **copy across regions** (`CopyImage`, can re-encrypt with a different KMS key)
- **Share** with specific accounts/orgs (explicit) or make public (launch permissions)
- **Encryption** — snapshot encryption with AWS-managed or customer-managed KMS keys
- **Boot modes / virtualization** — modern HVM (recommended, enhanced networking/GPU) vs legacy PV
- **EC2 Image Builder** — managed pipeline to build, test (e.g. with Inspector), and distribute AMIs automatically
- **User Data** — bootstrap scripts run at first boot, work *with* AMIs (bake + bootstrap combo)
- **Tags & naming** — version, app, environment for lifecycle management
- **Deregister + snapshot cleanup** — manage versions (keep last N)

## 6. Limitations

- **Regional only** — must copy across regions (takes time, storage cost for snapshots)
- **Architecture-locked** — x86 AMI can't launch Graviton/arm64 instances (and vice versa)
- **Snapshot storage costs** — AMIs bill for underlying EBS snapshots; old versions add up
- **Stale images** — OS/apps in a golden AMI age; needs a patch/rebuild pipeline
- **Machine-specific data** — must clean hostnames, logs, SIDs (Windows: run Sysprep) before creating
- **Not the delivery mechanism for fast-moving app code** — better for base OS + agents
- Create-image during heavy load can affect instance (consider `--no-reboot` carefully — consistency trade-off)

## 7. Trade-offs

- **Bake (Golden AMI) vs Bootstrap (User Data)**: fast, consistent, compliance-clean startup vs flexible, slower boot, no versioned-image storage
- **Golden AMI vs Containers**: EC2 fleets/Windows/GPU apps vs containerization where image lives in ECR/ECS
- **One AMI the fleet uses vs per-app AMIs**: simpler ops vs isolation of versions/apps
- **Frequent AMI rebuild vs occasional**: keeps patches fresh but costs pipeline + snapshot storage
- **Public AMIs**: convenient but trust-bound — use official/verified or build your own for security

## 8. Architecture

Golden-image pattern for a compliant, scalable fleet:

```
EC2 Image Builder pipeline:
  base AMI → install/configure (Ansible/shell) → test (Inspector, smoke tests)
  → register Golden AMI v1.2 → copy to other regions
Launch Template → references Golden AMI → Auto Scaling Group
  → every instance identical & compliant → elastic scaling = scale OUT by versioned images only
Deploy new app version? Build AMI v1.3 → ASG Instance Refresh → rolling replacement
DR? Golden AMI already copied to disaster-recovery Region → launch there
```

## 9. SAA-C03 Perspective

The exam tests AMI at the **architecture pattern** level:

- AMIs are **regional** — cross-region launch requires **copying** the AMI/snapshots first
- **AMI must match instance architecture** — x86 vs arm64 (Graviton) mismatch is a classic distractor
- **Golden AMI + ASG (Launch Template)** = compliant, compliant, immutable fleet — the "Madge/Tutorials Dojo favorite"
- **Immutable deployment**: new golden AMI → ASG Instance Refresh (not SSHing into running instances)
- **EC2 Image Builder** — managed pipeline for build/test/share of AMIs
- **AMI consistency during backup/unbounded**: snapshot/EBS vs Image Builder best-practice
- Know that **launching = EBS volumes hydrated from AMI snapshots**; backup strategy = snapshot AMI/EBS

Exam trap: "launch same fleet in another region" → answer involves **copying the AMI to that region**, never "just use the AMI ID directly". "Golden image for compliance" → bake hardening into AMI and let ASG use it.