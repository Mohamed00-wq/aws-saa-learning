# EC2 Pricing Models

## What it is
The different ways AWS lets you pay for EC2 compute capacity. Choosing the right pricing model is a core cost-optimization decision — the same workload can cost dramatically different amounts depending on which model it runs on. This sits squarely in the Cost-Optimized Architectures exam domain.

## Key concepts
- **On-Demand** — pay per hour/second, no commitment, most expensive per-hour rate
- **Reserved Instances (RI)** — 1 or 3-year commitment for a discount up to ~72%; Standard (cheaper, less flexible) vs Convertible (more flexible, smaller discount); payment options: All Upfront, Partial Upfront, No Upfront
- **Spot Instances** — unused AWS capacity at up to ~90% discount; can be reclaimed by AWS with a short interruption notice
- **Savings Plans** — commit to a fixed $/hour spend for 1 or 3 years; more flexible than RIs since the discount applies across instance families/regions automatically
- **Dedicated Hosts** — a full physical server allocated entirely to you, with visibility into sockets/cores — used for licensing tied to physical hardware
- **Dedicated Instances** — run on hardware dedicated to your account, but without visibility/control over the physical server itself

## How it works
Every EC2 instance you launch is billed under one of these models regardless of which AMI it came from or what workload it runs. On-Demand is the default and most flexible but costliest baseline. Reserved Instances and Savings Plans trade commitment (1 or 3 years) for a steep discount, and are best suited to steady, predictable workloads — e.g. a production database running continuously. Spot Instances trade reliability for the deepest discount: AWS can interrupt a Spot instance with roughly a two-minute warning if it needs that capacity back for On-Demand or Reserved customers, so Spot is only appropriate for fault-tolerant, interruption-tolerant workloads (batch processing, big data analysis, video rendering, CI/CD jobs, stateless workers). Dedicated Hosts and Dedicated Instances exist mainly for compliance and software licensing requirements that depend on physical core/socket counts, not for cost savings — they're typically the most expensive option.

These models can be mixed within the same account and even the same Auto Scaling Group (e.g. a baseline of Reserved Instances plus Spot Instances for burst capacity) to balance cost and reliability.

## Exam domain(s)
- [ ] Design Secure Architectures (30%)
- [ ] Design Resilient Architectures (26%)
- [ ] Design High-Performing Architectures (24%)
- [x] Design Cost-Optimized Architectures (20%)

## Lab notes
What I actually did hands-on (screenshots, steps, gotchas).

## Exam-style questions
**Q1.** A company runs a batch video-transcoding job that can tolerate being stopped and restarted at any point, and wants to minimize cost as much as possible. Which pricing model should they use?
- A) On-Demand Instances
- B) Reserved Instances
- C) Spot Instances
- D) Dedicated Hosts

<details>
<summary>Answer</summary>
C is correct — Spot Instances offer the deepest discount and are ideal for fault-tolerant, interruptible workloads like batch transcoding. On-Demand and Reserved cost more without a corresponding benefit here, and Dedicated Hosts address licensing/compliance, not cost savings.
</details>

**Q2.** A company runs a production database that must be available 24/7 for the next 3 years with no interruptions. Which pricing model provides the best cost savings while guaranteeing capacity is never reclaimed?
- A) Spot Instances
- B) On-Demand Instances
- C) Reserved Instances (3-year term)
- D) Dedicated Instances

<details>
<summary>Answer</summary>
C is correct — Reserved Instances suit steady-state, predictable workloads and offer a significant discount (up to ~72%) without the interruption risk of Spot. On-Demand is safe but far more expensive over 3 years; Dedicated Instances address hardware isolation, not cost optimization for this scenario.
</details>

## Related services
- [[EC2]] — the compute service these pricing models apply to
- [[Auto-scaling]] — often combines multiple pricing models (e.g. Reserved baseline + Spot for burst) within one group
- [[AMI]] — the template launched regardless of which pricing model the resulting instance uses