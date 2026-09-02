# EC2 Pricing Models

## What it is

EC2 pricing models define **how you pay for compute capacity**. The same workload running on the same instance type can cost dramatically different amounts depending on which pricing model is selected. Choosing the right model is one of the most direct cost-optimization levers in AWS and maps directly to the **Design Cost-Optimized Architectures** exam domain.

Every EC2 instance is billed under exactly one pricing model regardless of what AMI it came from or what workload it runs. Models can be mixed within the same account and even within the same Auto Scaling Group.

---

## Key concepts

### On-Demand Instances

- Pay for compute by the **second** (with a 60-second minimum) — no commitment.
- Most flexible: launch and terminate at any time.
- Most expensive per-hour rate.
- Best for: unpredictable workloads, development/testing, short-lived experiments, workloads that cannot tolerate interruption.
- No upfront payment, no long-term commitment.
- Default pricing model — this is what you get if you don't specify anything else.

**Exam note:** On-Demand is the cost baseline. All other models are measured against it.

### Reserved Instances (RI)

- **1 or 3 year commitment** to a specific instance type and region in exchange for a discount up to ~72% compared to On-Demand.
- Not a separate instance type — it is a billing discount applied to matching running instances. You do not "launch" a Reserved Instance; you purchase the reservation, and it automatically applies to any matching running instance in your account.

**Standard RI:**
- Maximum discount (up to ~72%).
- Least flexible: you commit to a specific instance family, region, and tenancy.
- Cannot be changed after purchase (except modifying some attributes within limits).

**Convertible RI:**
- Smaller discount (up to ~66%).
- More flexible: you can change the instance family, size, OS, and tenancy during the term.
- Useful if you anticipate workload changes but still want savings.

**Payment options:**
- **All Upfront** — pay the full RI cost upfront. Maximum discount.
- **Partial Upfront** — pay a portion upfront, the rest in monthly installments.
- **No Upfront** — no upfront payment, but smaller discount.

**RI Marketplace:** you can sell unused Standard RIs on the Reserved Instance Marketplace to other AWS users. Convertible RIs cannot be sold.

**Exam trap:** RIs are scoped to a specific region and availability zone (or instance tenancy). A Regional RI applies discounts across all AZs in the region. An AZ-scoped RI applies only to instances in that specific AZ.

### Savings Plans

A newer, more flexible commitment model that offers discounts similar to RIs but with less rigidity.

**Compute Savings Plans:**
- Commit to a fixed **$ per hour** spend for 1 or 3 years.
- The discount applies automatically across **instance families, regions, tenancy, and OS**.
- Most flexible savings option — if your workload shifts from m5 in us-east-1 to c5 in eu-west-1, the Savings Plan still applies.
- Discount: up to ~66% compared to On-Demand (slightly less than a Standard RI but much more flexible).

**EC2 Instance Savings Plans:**
- Commit to a specific **instance family** (e.g. M5) in a specific region.
- Tighter commitment than Compute Savings Plans, but larger discount (up to ~72%).
- Can still change instance size, OS, and tenancy within the family.

**Exam distinction:** if the question asks about flexibility across instance families/regions, the answer is Compute Savings Plans. If it asks about maximum discount with some flexibility (instance size), the answer is EC2 Instance Savings Plans.

### Spot Instances

- Access to **unused AWS EC2 capacity** at a discount of up to ~90% compared to On-Demand.
- AWS can **reclaim** (interrupt) a Spot instance with approximately **2 minutes notice** when it needs the capacity back.
- No SLA for availability — instances may be interrupted at any time.

**Spot interruption behavior:**
1. AWS sends an interruption notice 2 minutes before reclaiming the instance.
2. The instance metadata service provides a two-minute warning endpoint.
3. Your application can use this warning to save state, finish current work, or gracefully shut down.
4. After the 2-minute window, the instance is terminated (or stopped, depending on interruption behavior setting).

**Spot Blocks** (deprecated) previously allowed you to reserve Spot capacity for a fixed duration (1–6 hours). This is no longer available for new requests.

**Best workloads for Spot:**
- Batch processing
- Big data analytics (EMR, Spark)
- Containerized workloads (ECS, EKS)
- CI/CD pipelines
- Fault-tolerant, stateless, or checkpointable applications
- Rendering and transcoding
- Machine learning training (checkpointable)

**Not suitable for:**
- Databases
- Real-time applications
- Workloads that cannot tolerate interruption
- Stateful applications without external persistence

**Spot Fleet:** a collection of Spot (and optionally On-Demand) instances managed together. You define a target capacity and Spot Fleet handles launching, replacing interrupted instances, and optionally mixing instance types for diversification. Diversifying across multiple instance types and AZs increases the likelihood of maintaining capacity.

**Capacity Blocks:** a way to reserve Spot capacity for a future time window. You specify a future start time, duration, and desired capacity. AWS guarantees the capacity will be available when needed. This is useful for planned batch jobs that need Spot-level pricing with guaranteed availability at a specific time.

### Dedicated Hosts

- A **physical server** fully dedicated to your account — not shared with any other customer.
- Provides visibility into the physical sockets and cores, which is required for certain **software licensing** models that are tied to physical hardware (e.g. per-socket or per-core licenses for Windows Server, SQL Server, SAP).
- Most expensive option — not a cost optimization strategy.
- Also useful for compliance requirements that demand physical isolation.
- You can control instance placement on the host (e.g. ensure all instances of a type are on the same socket).

**Exam note:** if a question mentions "licensing requirements tied to physical hardware" or "compliance requiring dedicated servers," the answer is Dedicated Hosts, not Dedicated Instances.

### Dedicated Instances

- Run on hardware that is dedicated to your AWS account — no other customer's instances share the same physical host.
- **But** you do NOT have visibility or control over the physical server (unlike Dedicated Hosts).
- Cheaper than Dedicated Hosts.
- Useful for compliance requirements that require hardware isolation but do not need physical socket visibility.

**Dedicated Hosts vs Dedicated Instances:**
| Property | Dedicated Hosts | Dedicated Instances |
|---|---|---|
| Physical server visibility | Yes (sockets, cores) | No |
| Instance placement control | Yes | No |
| Per-socket/per-core licensing | Yes | No |
| Cost | Most expensive | Expensive (but less than Hosts) |
| Billed per | Host | Instance |

---

## Architecture deep dive

### How pricing models interact with other services

**Auto Scaling Groups:**
- You can configure an ASG to use a mix of pricing models. A common pattern is:
  - **Reserved Instances or Savings Plans** for the baseline (minimum) capacity — these run 24/7 and provide the lowest cost for steady-state.
  - **On-Demand** for moderate scaling — for predictable traffic increases.
  - **Spot Instances** for burst capacity — for short-term demand spikes that can tolerate interruption.
- ASG allocation strategies let you specify the order in which pricing types are used (e.g. prioritize Spot first, fall back to On-Demand).

**Spot Fleet allocation strategies:**
- **lowest-price** — launches from the Spot pool with the lowest current price. Simple but may lead to frequent interruptions if that pool becomes popular.
- **diversified** — distributes capacity across multiple pools (instance types/AZs). Reduces interruption risk.
- **capacity-optimized** — launches from the pool with the most available capacity. Lowest interruption rate, increasingly recommended by AWS.

### Compute Optimizer

AWS Compute Optimizer is a service that uses machine learning to analyze your historical usage patterns and recommend the most cost-effective EC2 instance types and pricing models. It considers:
- Actual CPU, memory, network, and disk utilization.
- Whether you should switch to a different instance type (right-sizing).
- Whether you should switch pricing models (e.g. from On-Demand to Savings Plans).

This is not a pricing model itself, but it helps you choose the right one.

### Cost optimization beyond pricing models

The exam tests cost optimization broadly, not just pricing models:
- **Right-sizing:** choosing the correct instance type for the workload. A m5.2xlarge when the workload only needs m5.large is wasting money.
- **Elastic IP idle charges:** an allocated but unassociated EIP incurs a per-hour charge.
- **EBS volume type:** using io2 when gp3 would suffice is a common waste.
- **S3 storage classes:** moving infrequently accessed data to S3 Glacier.
- **Stopping unused resources:** dev/test instances should be stopped outside business hours.
- **Spot for batch:** using Spot for non-critical batch processing instead of On-Demand.

---

## Exam domain(s)

- [ ] Design Secure Architectures (30%)
- [ ] Design Resilient Architectures (26%)
- [ ] Design High-Performing Architectures (24%)
- [x] **Design Cost-Optimized Architectures (20%)** — this is THE primary exam domain for pricing models

---

## Advanced gotchas & edge cases

1. **Savings Plans automatically apply to matching usage.** Unlike RIs (which need to match specific instance attributes), Compute Savings Plans automatically apply to any compute usage that matches the commitment rate. No manual association needed.

2. **Spot prices vary by pool and time.** AWS does not publish Spot prices in advance. They fluctuate based on supply and demand. The only reliable way to reduce interruptions is to diversify across instance types and AZs.

3. **Spot interruption behavior can be set to Stop, Hibernate, or Terminate.** By default, interrupted Spot instances are terminated. You can change this to Stop (preserving EBS data) or Hibernate (preserving RAM state to EBS).

4. **Savings Plans can cover Lambda and Fargate, not just EC2.** Compute Savings Plans also apply to Lambda duration and Fargate usage, making them more versatile than RIs.

5. **Reserved Instances do not auto-renew.** When an RI term expires, you are billed at On-Demand rates. The exam may test whether you know to plan for RI expiration.

6. **Convertible RIs can be exchanged for different instance families, but the total value must be equal or higher.** You cannot downgrade to a cheaper instance type.

7. **Spot instance termination notices are available via the instance metadata service.** Applications that check for the 2-minute warning can save state gracefully. This is a key resilience pattern.

8. **On-Demand capacity reservations** guarantee that you can launch On-Demand instances in a specific AZ, even during high-demand periods. This is different from RIs — it does not provide a discount but provides capacity assurance.

9. **RIs and Savings Plans stack for discounts.** If you have an RI for an m5.large in us-east-1a and the instance is running in us-east-1a, the RI discount applies. Savings Plans apply if the commitment rate is met. But you cannot double-discount the same instance.

10. **Dedicated Hosts can be purchased as On-Demand, Reserved, or Spot.** You can commit to a Dedicated Host for 1 or 3 years to reduce the already-high cost.

---

## Exam-style questions

**Q1.** A company runs a batch video-transcoding job that can tolerate being stopped and restarted at any point. They want to minimize cost. Which pricing model should they use?
- A) On-Demand Instances
- B) Reserved Instances
- C) Spot Instances
- D) Dedicated Hosts

<details><summary>Answer</summary>
**C** — Spot instances offer up to 90% discount and are ideal for fault-tolerant, interruptible workloads. The workload can tolerate stop/restart, so Spot interruption is acceptable. On-Demand and Reserved are more expensive without benefit here. Dedicated Hosts are for licensing/compliance, not cost savings.
</details>

**Q2.** A company runs a production database that must be available 24/7 for the next 3 years with no interruptions. Which pricing model provides the best cost savings while guaranteeing capacity is never reclaimed?
- A) Spot Instances
- B) On-Demand Instances
- C) Reserved Instances (3-year term)
- D) Dedicated Instances

<details><summary>Answer</summary>
**C** — Reserved Instances provide a significant discount (up to ~72%) for steady-state workloads with no interruption risk. Spot can be reclaimed. On-Demand is safe but far more expensive over 3 years. Dedicated Instances address hardware isolation, not cost optimization.
</details>

**Q3.** A Solutions Architect wants to reduce compute costs but needs flexibility to change instance types, regions, and operating systems over the next 12 months. Which option provides the best balance of savings and flexibility?
- A) Standard Reserved Instances
- B) Convertible Reserved Instances
- C) Compute Savings Plans
- D) Spot Instances

<details><summary>Answer</summary>
**C** — Compute Savings Plans provide up to ~66% discount with flexibility across instance families, regions, tenancy, and OS — the most flexible commitment option. Standard RIs (A) are the least flexible. Convertible RIs (B) are more flexible than Standard but less than Savings Plans. Spot (D) does not provide a guaranteed discount.
</details>

**Q4.** A company has a predictable, steady-state workload running 24/7 on 100 m5.xlarge instances in us-east-1. They plan to keep this workload for 3 years and want the maximum possible discount. What should they use?
- A) Compute Savings Plans
- B) EC2 Instance Savings Plans
- C) On-Demand Instances
- D) Spot Instances

<details><summary>Answer</summary>
**B** — EC2 Instance Savings Plans commit to a specific instance family (m5) in a specific region, offering up to ~72% discount — close to Standard RI levels with slightly more flexibility (instance size, OS, tenancy). Compute Savings Plans (A) are more flexible but offer a smaller discount (~66%).
</details>

**Q5.** A company's batch processing job runs on EC2 and currently uses On-Demand Instances. The workload can checkpoint its state to S3 and resume from the last checkpoint. How can they reduce costs without changing the application architecture?
- A) Switch to Reserved Instances
- B) Switch to Spot Instances
- C) Switch to Dedicated Hosts
- D) Switch to a larger instance type

<details><summary>Answer</summary>
**B** — the workload's ability to checkpoint and resume makes it a perfect fit for Spot Instances. If interrupted, it restarts from the last checkpoint with minimal wasted work. Reserved Instances (A) would save money but not as much as Spot for an interruptible workload.
</details>

**Q6.** A Solutions Architect notices that a fleet of EC2 instances is consistently using less than 15% CPU and 20% memory. Which cost optimization strategy should they recommend first?
- A) Purchase Reserved Instances
- B) Move to Spot Instances
- C) Right-size to a smaller instance type
- D) Enable hibernation

<details><summary>Answer</summary>
**C** — right-sizing (choosing a smaller instance type that matches actual usage) is the most fundamental cost optimization. If the workload only uses 15% CPU and 20% memory, a smaller instance type (e.g. from m5.xlarge to m5.large) would cut costs in half before any pricing model is even considered.
</details>

---

## Related services

- [[EC2]] — the compute service these pricing models apply to
- [[Auto-scaling]] — ASGs can combine multiple pricing models (Reserved baseline + Spot burst) within one group
- [[AMI]] — the template launched regardless of which pricing model the resulting instance uses
- [[Savings Plans]] — AWS Savings Plans console for managing Compute and EC2 Instance Savings Plans
- [[AWS-Compute-Optimizer]] — ML-based recommendations for right-sizing and pricing model selection
