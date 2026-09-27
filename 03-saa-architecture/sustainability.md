# Sustainability

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Sustainability is the **sixth pillar of the AWS Well-Architected Framework** (added in December 2021). It focuses on the **environmental impact of workloads**, especially energy consumption and efficiency. The goal is to maximize the utilization of every resource you deploy, minimize waste, reduce the carbon footprint of running in the cloud, and do it without sacrificing the other pillars. For an architect, sustainability is treated as a **non-functional requirement** alongside security, reliability, and cost. For the SAA exam it is the newest and smallest domain, but its key practices show up in questions about right-sizing, lifecycles, and region choice.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - You measure and understand the workload's actual demand, utilization, and lifecycle, then design to match real usage instead of worst-case forecasts.
  - You choose Regions with access to **renewable energy / low-carbon grids**, and you align capacity to demand (scale down, right-size, use serverless and spot).
  - You reduce data waste (compress, tier to colder storage, archive old data) and adopt efficient software and modern hardware.
- Why does it work?
  - Energy use tracks **deployed and powered resources**. Every idle, oversized, or redundant resource is wasted energy. Aligning supply with demand and removing waste directly cuts both carbon and cost.
- What is the main idea behind it?
  - Sustainability best practices mirror cost optimization, because efficient utilization helps both. Six focus areas drive the pillar: **region selection**, **alignment to demand**, **software and architecture**, **data management**, **hardware and services**, and **process and culture**. Design principles include understanding impact, establishing sustainability goals, maximizing utilization, anticipating new and more efficient hardware, and reducing downstream impact.

Shared responsibility note: AWS is responsible for **sustainability of the cloud** (renewable energy, data center efficiency). Customers are responsible for **sustainability in the cloud** (workload design, utilization, data storage).

## 3. Trade-offs

What do you gain?
- **Lower energy use and carbon footprint** for the workload
- **Lower cost**, since most sustainability actions (right-size, scale-down, tier storage, remove idle) also save money
- Better optics and compliance with ESG and green reporting requirements
- Alignment with modern AWS guidance and exam expectations

What do you sacrifice?
- **Effort and analysis**  you must instrument utilization and review storage/data lifecycles
- **Possible conflicts with other pillars**  chasing minimum footprint could threaten performance (under-provision) or reliability (removing redundancy)
- **Latency vs green Regions**  choosing a low-carbon Region may be farther from users, raising latency
- **Migration cost**  adopting newer, more efficient instance generations and services takes re-engineering time

## 4. AWS Services That Work With This Concept

- **AWS Customer Carbon Footprint Tool**  estimates and reports the carbon emissions of your AWS usage over time
- **AWS Well-Architected Tool**  Sustainability pillar questions and reviews, plus the Sustainability Lens
- **AWS Compute Optimizer**  rightsizing recommendations that maximize utilization (less idle = less energy)
- **Amazon EC2 Auto Scaling / AWS Fargate / Lambda**  align capacity to demand, remove idle compute
- **Amazon EC2 Spot Instances**  use spare capacity for fault-tolerant work with no extra energy
- **Amazon S3 lifecycle policies / Intelligent-Tiering**  move cold data to lower tiers (Glacier, Deep Archive) to avoid powering infrequently-used storage
- **Amazon EBS snapshots / data lifecycle**  delete stale snapshots and unused volumes
- **Instance families**  newer generation Arm-based (Graviton) and more efficient instance types lower energy per unit of work

## 5. When to Use

Use this concept when:
- The workload has **low utilization** (idle instances, oversized types) and you want to right-size it, this helps both cost and sustainability
- Data grows without review, lifecycle policies can archive or expire old objects
- You are choosing a **Region** and can consider carbon intensity alongside latency and cost
- The organization has ESG or carbon-reduction goals that must be reflected in architecture
- Exam questions mention "sustainability," "carbon footprint," "reduce energy," "Customer Carbon Footprint Tool," or the "sixth pillar"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The top priority contract or SLA forbids scaling down or removing redundant capacity, sustainability must not degrade reliability
- Latency is critical and the only low-carbon Region is geographically far from users
- The strongest answer to a question is clearly a **cost-specific** mechanism (Savings Plans, spot pricing), sustainability is related but the exam wants the direct cost action
- The workload is experimental and has negligible footprint, adding lifecycle tooling is more waste than saving
- Pilars conflict and business value of a faster/more reliable design outweighs the small carbon delta, trade-offs are explicit in the framework