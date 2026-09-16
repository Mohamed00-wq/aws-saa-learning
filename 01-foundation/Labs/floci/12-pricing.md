# Lab 12 — EC2 Pricing Models with Floci (Conceptual)

> Pricing is account/billing state — not something floci charges you for. But floci's EC2 API still exposes the pricing surfaces: Reserved Instances purchases, Spot pricing, and capacity reservations. This lab exercises the **control-plane** side of every pricing model so you can see them in your local AWS.

**Reference notes:** `services/EC2Pricing-models.md`

> **Key context:** in real AWS, pricing models are billing discounts & contract commitments applied to the *same* instance types, AMIs, and architectures. The choice is orthogonal to what you build.

## 1. On-Demand (baseline — no commands needed)

- Pay per second (60s min), no commitment, launch/terminate anytime.
- Most expensive. Default for everything.

```bash
aws ec2 run-instances --image-id ami-00000000 --instance-type t2.micro --query 'Instances[0].[InstanceId,InstanceType]' --output text 2>/dev/null || echo "default pricing model = On-Demand"
```

## 2. Reserved Instances — purchase surface

```bash
aws ec2 describe-reserved-instances-offerings \
  --instance-type t3.micro \
  --product-description "Linux/UNIX" \
  --query 'ReservedInstancesOfferings[0].[ReservedInstancesOfferingId,InstanceType,Duration,MetricType]'
```

Purchase one (1 year, no upfront):

```bash
aws ec2 purchase-reserved-instances-offering \
  --reserved-instances-offering-id <offering-id-from-above> \
  --instance-count 1

aws ec2 describe-reserved-instances
```

**What this really is (real AWS):** a **billing discount**, not a launch type. It applies to any matching running instance. Discount up to ~72% for Standard (fixed family/region/tenancy), ~66% for Convertible (can swap family, must keep total value ≥).

## 3. Savings Plans — different contract, same goal

```bash
# Savings Plans are managed in the console/billing API:
# aws savingsplans create-savings-plan / describe-savings-plans
aws savingsplans describe-savings-plans 2>/dev/null || echo "savingsplans endpoint may be billing-only in floci"
```

**Concepts to lock in:**

| Plan | Discount (1/3yr) | Flexibility |
|---|---|---|
| **Compute SP** | up to ~66% | across families, regions, size, tenancy, **+ Lambda + Fargate** |
| **EC2 Instance SP** | up to ~72% | fixed family + region; can change size/OS/tenancy |

**Exam line:** flexibility across families/regions → **Compute SP**. Max discount with size flexibility → **EC2 Instance SP**. SPs **auto-apply** — no manual association.

## 4. Spot — request & pricing surface

Spot = unused AWS capacity at up to ~90% discount. Warnings 2 min before reclaim via instance metadata.

```bash
aws ec2 describe-spot-price-history \
  --instance-types t2.micro t3.micro \
  --product-description "Linux/UNIX" \
  --start-time "2026-01-01T00:00:00" \
  --end-time "2026-01-01T01:00:00" \
  --query 'SpotPriceHistory[*].[InstanceType,SpotPrice,Timestamp]'
```

Spot Fleet / request:

```bash
aws ec2 request-spot-fleet \
  --spot-fleet-request-config '{
    "TargetCapacity": 2,
    "IamFleetRole": "arn:aws:iam::000000000000:role/aws-ec2-spot-fleet-tagging-role",
    "LaunchSpecifications": [
      { "ImageId": "ami-00000000", "InstanceType": "t2.micro" }
    ],
    "AllocationStrategy": "capacityOptimized"
  }' 2>/dev/null || echo "Spot Fleet needs the fleet role + may be partial in floci — concept below"
```

**Interruption behavior:** Stop, Hibernate, or Terminate (default Terminate). Allocation strategies: `lowest-price` (cheap, risky) · `diversified` · `capacity-optimized` (most available, fewest interruptions — **recommended**).

**Capacity Blocks** = reserved Spot for a future window (planned batch, guaranteed).

## 5. Dedicated Hosts vs Dedicated Instances

Dedicated **Hosts** = physical server, sockets/cores visible → per-socket/per-core **licensing** (Windows/SQL/SAP). Most expensive. Dedicated **Instances** = dedicated hardware but no visibility/control. Cheaper, billed per instance.

```bash
# Host purchase surface:
aws ec2 describe-host-reservation-offerings --instance-family t3 2>/dev/null || true
# or describe_hosts after allcoate
aws ec2 allocate-hosts --instance-type t3.micro --availability-zone us-east-1a --quantity 1 2>/dev/null || echo "host allocation may need floci EC2 parity"
```

**Exam answer for licensing:** physical-licensing/compliance → Dedicated **Hosts** (not Instances).

## 6. The ASG mix pattern (production cost design)

```
Baseline (24/7):   Reserved Instances / Savings Plans   → always-on capacity
Moderate scale:    On-Demand                            → normal growth waves
Burst / spikes:    Spot                                 → interruptible, cheapest
```

ASG `InstancesDistribution` sets the order (e.g. `SpotAllocationStrategy: capacity-optimized`, On-Demand base + %):
see [09-auto-scaling.md](09-auto-scaling.md).

## 7. Other cost optimizations to remember

- **Right-sizing** — correct instance type (over-provision = waste)
- Idle **EIP charges** (allocate + don't use = hourly fee)
- EBS: **gp3 over gp2/io2** where possible
- **Stop unused dev/test outside hours** (no compute charge while stopped)
- **AWS Compute Optimizer** — ML recommendations for size + model

## Exam hooks

1. **Savings Plans auto-apply** to matching usage — no manual step
2. **RIs don't auto-renew** — On-Demand billing after expiry
3. Convertible RI: can swap families but **no downgrade** (total value stays ≥)
4. On-Demand capacity reservation = capacity assurance in an AZ, **NOT a discount**
5. RI + Savings Plans can't double-discount the same instance
6. Compute SPs cover **Lambda + Fargate**
7. Spot: 2-min reclaim warning via metadata = graceful checkpointing

## Gotchas to remember

1. Same workload, same instance type — different model = different price. Models mix freely within an account/ASG.
2. Spot not for DBs / real-time / non-interruption-tolerant workloads
3. EIP idle = money; release when unused