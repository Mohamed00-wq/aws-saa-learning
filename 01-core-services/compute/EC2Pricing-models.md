# EC2 Pricing Models

## What it is

Pricing models define **how you pay for compute**. Same workload on the same type costs very differently by model. One of the most direct cost-optimization levers — maps to **Design Cost-Optimized Architectures**. Models can mix within an account and ASG.

## On-Demand

- Pay per **second** (60s min), no commitment — launch/terminate anytime.
- Most flexible, **most expensive**. Default model; the cost baseline all others are measured against.
- Best: unpredictable workloads, dev/testing, workloads that can't tolerate interruption.

## Reserved Instances (RI)

- **1 or 3-year** commitment to a type+region for discount up to **~72%**. Not a launch type — a **billing discount** applied to matching running instances.
- **Standard RI**: max discount (~72%), least flexible (fixed family/region/tenancy), hard to change.
- **Convertible RI**: less discount (~66%), can change family/size/OS/tenancy.
- **Payment**: All Upfront (max discount) → Partial → No Upfront.
- **RI Marketplace**: sell unused Standard RIs (Convertible can't be sold).
- RIs are AZ/region-scoped.

## Savings Plans

Commit to a fixed **$/hour** for 1/3 years.

- **Compute Savings Plans**: up to ~66%. Applies across **instance families, regions, tenancy, OS** — most flexible. Also covers **Lambda + Fargate**. (Flexibility → Compute SP)
- **EC2 Instance Savings Plans**: up to ~72%. Commits to instance **family + region**, can change size/OS/tenancy. (Max discount with some flexibility → Instance SP)

**Exam:** flexibility across families/regions → Compute SP. Max discount with size flexibility → EC2 Instance SP.

## Spot Instances

- Unused AWS capacity at up to **~90%** discount.
- AWS can **reclaim with ~2 min notice**; no availability SLA. Interruption behavior: stop, hibernate, or terminate.
- Warning available via **instance metadata** — checkpoint/save state gracefully.
- **Best**: batch, big data (EMR/Spark), ECS/EKS, CI/CD, fault-tolerant/checkpointable, rendering, ML training.
- **Not for**: databases, real-time, workloads that can't tolerate interruption, stateful without persistence.
- **Spot Fleet**: manages target capacity + replaces interrupted instances; diversify types/AZs to reduce interruption.
- **Capacity Blocks**: reserve Spot for a future window (planned batch, guaranteed availability).
- **Allocation**: `lowest-price` (cheap, risky) · `diversified` (spread) · `capacity-optimized` (most available, lowest interruption — recommended).

## Dedicated Hosts

- **Physical server** dedicated to your account. Visibility into **sockets/cores** for per-socket/per-core **licensing** (Windows/SQL/SAP). Most expensive. Can be purchased On-Demand/Reserved/Spot.
- **Exam:** physical-licensing/compliance → Dedicated **Hosts** (not Instances).

## Dedicated Instances

- Dedicated hardware, **but no visibility/control** over physical server. Cheaper than Hosts. Billed per instance.

| | Dedicated Hosts | Dedicated Instances |
|---|---|---|
| Physical visibility (sockets/cores) | Yes | No |
| Instance placement control | Yes | No |
| Per-socket licensing | Yes | No |
| Cost | Most expensive | Less |
| Billed per | Host | Instance |

## ASG mix pattern

**Reserved/Savings Plans** for baseline (24/7) + **On-Demand** for moderate scaling + **Spot** for burst. ASG allocation strategy sets the order (e.g. Spot first, fall back On-Demand).

## Other cost optimization

- **Right-sizing** — correct instance type; over-provisioning wastes money
- Idle **EIP charges**
- EBS type — gp3 vs io2
- S3 storage classes / Glacier
- Stop unused dev/test outside hours
- Compute Optimizer (ML) recommends right-sizing + pricing model

## Exam domains

- [ ] Secure (30%)
- [ ] Resilient (26%)
- [ ] High-Performing (24%)
- [x] **Cost-Optimized (20%)** — THE primary domain for pricing models

## Key gotchas

1. **Savings Plans auto-apply** to matching usage — no manual association
2. Spot prices fluctuate by pool/time; diversify (types/AZs) to reduce interruption
3. Spot interruption can be **Stop/Hibernate/Terminate** (default Terminate)
4. **Compute Savings Plans cover Lambda + Fargate**, not just EC2
5. **RIs don't auto-renew** — billed On-Demand after expiry
6. Convertible RIs can swap families but **total value must stay equal/higher** — no downgrade
7. Spot 2-min notice via metadata service = resilience pattern
8. **On-Demand capacity reservations** = capacity assurance in an AZ, NOT a discount
9. RI + Savings Plans can't double-discount the same instance
10. Dedicated Hosts available On-Demand/Reserved/Spot

## Related services

- **EC2** — compute these models apply to
- **ASG** — combine models (Reserved baseline + Spot burst)
- **AMI**— template regardless of pricing model
- **Savings Plans** — console for managing SPs
- **AWS-Compute-Optimizer** — right-sizing + model recommendations
