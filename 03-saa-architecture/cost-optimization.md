# Cost Optimization

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Cost optimization delivers the required performance, availability, and security at the **lowest possible price**. The goal is to eliminate waste, over-provisioning, and idle or orphaned resources, and to pay the right price for the right resource. It is the fourth pillar of the AWS Well-Architected Framework and in the SAA exam the 20% Cost-Optimized Architectures domain. Cloud billing is usage-based, so you optimize cost by matching purchasing decisions and capacity choices to the actual workload pattern.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - You measure actual utilization with Cost Explorer, budgets, and usage reports.
  - You right-size resources (pick the instance type that matches real CPU, memory, and network use instead of the biggest available).
  - You switch idle and predictable workloads to cheaper options and commit to Reserved/plan-based pricing.
- Why does it work?
  - AWS charges per hour/second and per GB. Unused capacity is pure waste. Pricing models (on-demand, savings plans, reserved, spot) each reward a different trade-off between flexibility and discount, so you pick per workload.
- What is the main idea behind it?
  - **Right-size, then right-price, then eliminate waste.** Five pillars: rightsizing, purchasing options (savings plans/reserved/spot), storage tiering and lifecycle, serverless and managed services, and continuous monitoring of idle/underused resources. Cost is an architectural input, not an afterthought.

## 3. Trade-offs

What do you gain?
- Directly lower cloud bills, potentially 30-70% with committed/spot purchasing
- Free budget for innovation or for improving other pillars
- Predictable spend through budgeting and anomaly detection

What do you sacrifice?
- **Commitment flexibility**  savings plans and reserved instances discount in exchange for 1-3 year commitments, right-sizing the wrong model locks in waste
- **Risk**  spot instances are cheap but can be reclaimed at 2-minute notice, so they need interruption-tolerant workloads
- **Engineering time**  analyzing usage, tagging, and tuning takes effort
- Aggressively optimizing cost can hurt performance or availability (a tiny instance, fewer replicas, cold-tier storage) when balanced incorrectly

## 4. AWS Services That Work With This Concept

- **AWS Cost Explorer**  visualize and analyze spend, forecast, and find under-utilized resources
- **AWS Budgets + Cost Anomaly Detection**  alerts and automated anomaly detection on spend
- **AWS Savings Plans**  commit to $/hour usage of compute for up to 36% steady-state savings
- **Reserved Instances**  commit to specific instance families (Compute, EC2, RDS) for up to 72% discounts
- **Amazon EC2 Spot**  up to 90% discount for interruption-tolerant, stateless workloads
- **Amazon S3 lifecycle policies + S3 Intelligent-Tiering**  move data to cheaper tiers (Standard-IA, One Zone-IA, Glacier, Glacier Deep Archive) automatically
- **AWS Trusted Advisor / Compute Optimizer**  rightsizing recommendations and utilization analysis
- **AWS Auto Scaling**  scale down during low demand to stop paying for idle capacity
- **AWS Fargate / Lambda**  managed, usage-based compute with no idle servers to pay for
- **Amazon EBS (gp3) / S3 (non-retrieval tiers)**  lower unit costs with right-sized volumes

## 5. When to Use

Use this concept when:
- Workloads are stable and predictable (right for savings plans and reserved capacity)
- There are idle, untagged, or over-provisioned resources you can identify and eliminate
- Data can move to colder storage tiers based on access patterns
- You run batch, stateless, fault-tolerant workloads suited to spot instances
- Exam questions mention "reduce costs," "right sizing," "savings plans vs on demand," "spot instances," "lifecycle policies," or "identify idle resources"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- Saving money harms an SLA-critical availability, latency, or durability requirement
- Commitment discounts are risky because the workload is short-lived, highly variable, or going away
- The "optimization" is really just dropping a replica or a service you actually need
- Introducing more tooling (tagging, monitoring, lifecycle tuning) costs more than it saves for a tiny workload
- Spot instances are unacceptable because your workload cannot tolerate interruption (stateful, time-critical, low-fault-tolerance)