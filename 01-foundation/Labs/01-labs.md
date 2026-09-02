# Lab 01 — EC2 & VPC Foundation (Conceptual Walkthrough)

> **Goal:** Understand how to build a complete network stack from scratch — VPC → Subnet → Internet Gateway → Route Table → Security Group — launch an EC2 instance inside it, assign an Elastic IP, and explore EC2 pricing models. This walkthrough explains each step conceptually so you understand the "why" before you touch the CLI.

**Exam domains covered:** Design Secure Architectures (30%) · Design Resilient Architectures (26%) · Design Cost-Optimized Architectures (20%)

---

## Part 1 — Discover available AMIs

**What you need to understand:**
Before launching any instance, you need an AMI — the blueprint. When listing AMIs, you should filter by:
- **Owner:** filter to `self` (your own AMIs) or specific AWS accounts. Without this, you'll pull community and marketplace AMIs too, which can be overwhelming and potentially untrusted.
- **Fields to inspect:** `ImageId` (the AMI ID, e.g. `ami-0abc123`) and `Name` (human-readable label). Other useful fields: `State` (must be `available`), `RootDeviceType` (EBS-backed vs instance store), `VirtualizationType` (hvm vs paravirtual — always prefer HVM).

**Why this matters for the exam:** Choosing an AMI is a prerequisite for launching EC2 instances. Understanding AMI types (AWS-provided, marketplace, community, custom) and their trust implications is tested in the Secure Architectures domain.

---

## Part 2 — Create an SSH key pair

**What you need to understand:**
- A key pair consists of a **public key** (stored by AWS) and a **private key** (downloaded by you, once).
- The private key is used for SSH authentication — AWS does not store it, and you cannot retrieve it after creation.
- The private key file must have restricted permissions (read-only for the owner). SSH clients refuse to use keys with overly open permissions (e.g. `chmod 644`). The correct permission is `400` (read-only by owner).
- **For the exam:** key pairs are an EC2-specific authentication mechanism. For applications needing AWS API access, use IAM roles instead.

**Security note:** never share private key files. Treat them like passwords. If compromised, delete the key pair and create a new one, then update all instances that used the old key.

---

## Part 3 — Build a dedicated VPC from scratch

**What you need to understand:**
- A **VPC (Virtual Private Cloud)** is your isolated network in AWS. Every AWS resource that needs networking lives inside a VPC.
- **CIDR block** defines the IP address range (e.g. `10.0.0.0/16` gives you 65,536 IP addresses). The CIDR cannot be changed after creation.
- **Default VPC vs custom VPC:** every account comes with a default VPC pre-configured with a public subnet, internet gateway, and route table. For the exam and production, you should use a **custom VPC** — the default VPC is for quick experimentation, not production.
- **Naming/tags:** always tag your VPC with a descriptive name. In production environments with many VPCs, untagged resources are impossible to manage.

**Why `/16`?** A `/16` CIDR is a common choice because it gives you room to create many subnets (e.g. `10.0.1.0/24`, `10.0.2.0/24`, etc.) without running out of IP space. A `/24` VPC (256 IPs) is too small for most workloads.

---

## Part 4 — Create a public subnet inside the VPC

**What you need to understand:**
- A **subnet** is a subdivision of your VPC's IP range. Each subnet lives in exactly one **Availability Zone** (AZ).
- **Public subnet vs private subnet:**
  - A **public subnet** has a route to an Internet Gateway — instances in it can reach the internet (and be reached from it).
  - A **private subnet** has no route to an Internet Gateway — instances can only communicate within the VPC (or via NAT Gateway for outbound-only internet access).
- **CIDR within the VPC:** the subnet's CIDR must fall entirely within the VPC's CIDR range. A `10.0.1.0/24` subnet (256 IPs) inside a `10.0.0.0/16` VPC is valid.
- **AZ affinity:** a subnet is tied to a specific AZ. EBS volumes, ENIs, and other AZ-scoped resources launched in that subnet will be in the same AZ. For high availability, always use subnets in at least 2 AZs.

**Exam note:** the number of usable IPs in a subnet is less than the CIDR suggests. AWS reserves 5 IPs per subnet (first 4 + last 1) for networking purposes: the network address, the VPC router, the DNS server, the reserved future use, and the broadcast address. A `/24` subnet gives you 251 usable IPs, not 256.

---

## Part 5 — Internet Gateway

**What you need to understand:**
- An **Internet Gateway (IGW)** is a horizontally scaled, redundant AWS-managed component that enables communication between your VPC and the internet.
- **Two-step process:** creating the IGW is not enough — you must also **attach** it to your VPC. This is a common exam and lab trap: "I created an IGW but my instance has no internet access" — because you forgot to attach it.
- An IGW is required for instances in a public subnet to have public IP reachability.
- An IGW also performs **NAT** for instances that have public IPs — it translates the instance's private IP to a public IP for outbound traffic.

**Why not just "plug into the internet"?** The IGW is the controlled entry/exit point. Without it, your VPC is completely isolated. This isolation is a security feature — nothing gets in or out without explicit routing.

---

## Part 6 — Route Table

**What you need to understand:**
- A **route table** is a set of rules (routes) that determine where network traffic is directed.
- Every VPC has an implicit, uneditable **main route table**. If a subnet is not explicitly associated with a route table, it uses the main one.
- **For internet access:** you need a route with destination `0.0.0.0/0` (all traffic) and target = the Internet Gateway. This tells the VPC: "any traffic that doesn't match a more specific route goes to the internet."
- **Three-step process:** create the route table → add the `0.0.0.0/0` route pointing to the IGW → associate the route table with your subnet.
- **Without the association**, the route table exists but has no effect on any subnet. The subnet uses the main route table by default (which typically has no internet route).

**Exam note:** route tables are evaluated from most specific to least specific. A route to `10.0.0.0/16` (local) is more specific than `0.0.0.0/0` (internet), so VPC-internal traffic stays within the VPC and only unknown destinations go to the IGW.

---

## Part 7 — Security Group

**What you need to understand:**
- A **Security Group (SG)** is a stateful virtual firewall that controls inbound and outbound traffic at the **instance level** (ENI level).
- **Stateful:** if you allow inbound HTTP (port 80), the response is automatically allowed outbound — you do not need to add an outbound rule for the return traffic. This is because the SG "remembers" the connection.
- **Default behavior:** all traffic is denied unless explicitly allowed. An empty SG blocks everything.
- **Inbound vs outbound:** you must explicitly create rules for each direction. For a web server, you'd typically allow inbound 80/443 and outbound all (so it can reach databases, external APIs, etc.).
- **SG rules are additive:** there are no explicit Deny rules in SGs — only Allow rules. If traffic matches any Allow rule, it is permitted. If it matches no rule, it is implicitly denied.

**For this lab:** creating a SG with only port 22 (SSH) open means you can SSH into the instance from your machine, but nothing else can reach it over the network (until you add more rules).

**SG vs NACL recap:**
| Property | Security Group | NACL |
|---|---|---|
| Level | Instance (ENI) | Subnet |
| Stateful | Yes | No |
| Default | Deny all | Allow all |
| Rules | Allow only | Allow and Deny |
| Evaluation | All rules evaluated | Rules evaluated in order |

---

## Part 8 — Launch the EC2 instance

**What you need to understand:**
Launching an instance combines all the pieces you've built:
- **AMI:** the OS template.
- **Instance type:** hardware profile (for a lab, `t2.micro` or `t3.micro` is sufficient — eligible for free tier).
- **Key pair:** for SSH access.
- **Subnet:** determines the AZ and IP range.
- **Security group:** the firewall rules.
- **Count:** how many identical instances to launch.

After launch, the instance goes through the lifecycle: `pending` → `running`. You can verify the state and find the public/private IP.

**Important:** the instance's **private IP** is assigned from the subnet's CIDR range and is permanent while the instance is running. The **public IP** (if auto-assigned) may change on stop/start — use an Elastic IP for a permanent public IP.

---

## Part 9 — Elastic IP

**What you need to understand:**
- An **Elastic IP (EIP)** is a static, persistent public IP address allocated to your account.
- It persists across stop/start cycles — unlike an auto-assigned public IP, which changes when you stop and restart the instance.
- **Two-step process:** allocate the EIP → associate it with your instance.
- **Cost implication:** AWS charges for **idle Elastic IPs** — if an EIP is allocated but not associated with a running instance, you pay an hourly fee. This is a key cost-optimization point: release EIPs you're not using.

**When to use an EIP:**
- When you need a stable public IP for DNS records (e.g. pointing `api.example.com` to a fixed IP).
- When clients or services cache the IP and changing it would break connectivity.
- For instances that need to be reachable at a consistent address.

**When NOT to use an EIP:**
- For most web applications behind an ALB (the ALB has its own DNS name; clients don't need instance IPs).
- For temporary or burst instances (Spot, ASG-managed) — EIPs are a manual, static resource that don't fit dynamic architectures.

---

## Part 10 — EC2 Pricing Models (conceptual overview)

Understanding pricing is a core cost-optimization skill:

| Model | Commitment | Discount | Risk | Best for |
|---|---|---|---|---|
| **On-Demand** | None | 0% (baseline) | None | Unpredictable workloads, dev/test |
| **Reserved (RI)** | 1–3 years | Up to ~72% | Locked to instance type/region | Steady, predictable production workloads |
| **Savings Plans** | 1–3 years | Up to ~66% (Compute) | Committed $/hr spend | Flexible steady workloads (any instance family/region) |
| **Spot** | None | Up to ~90% | Can be interrupted | Fault-tolerant batch jobs, CI/CD, rendering |
| **Dedicated Host** | On-Demand or RI | Varies | Physical server isolation | Licensing tied to physical hardware |

**Key insight for the exam:** the pricing model is orthogonal to the AMI, instance type, and architecture. You can run the same workload on any pricing model. The choice depends on: cost sensitivity, interruption tolerance, commitment horizon, and licensing requirements.

**Mixing models in production:**
- **Baseline capacity:** Reserved Instances or Savings Plans (always-on, predictable)
- **Burst capacity:** On-Demand (moderate scaling)
- **Fault-tolerant burst:** Spot (aggressive scaling, interruption-tolerant)

---

## Recap — What you built and why

```
VPC (10.0.0.0/16)
 └── Public Subnet (10.0.1.0/24) — in AZ X
      ├── Internet Gateway — attached to VPC
      ├── Route Table — 0.0.0.0/0 → IGW, associated with subnet
      ├── Security Group — port 22 inbound (SSH)
      └── EC2 Instance — t2.micro, with Elastic IP
```

Every piece serves a purpose:
- **VPC:** isolated network boundary
- **Subnet:** IP range + AZ placement
- **IGW:** internet connectivity
- **Route Table:** traffic direction
- **Security Group:** instance-level firewall
- **EC2:** the compute workload
- **Elastic IP:** stable public address

Without any one of these, the architecture breaks: no IGW = no internet. No route table association = no traffic flow. No SG rules = all traffic blocked. This is the foundation every AWS architecture builds on.

---

## Exam-style Questions

**Q1.** A company needs to run a batch processing job that can tolerate interruptions, at the lowest possible cost. Which pricing model fits best?
- A) On-Demand
- B) Reserved
- C) Spot
- D) Dedicated Host

<details><summary>Answer</summary>
**C** — Spot instances offer the deepest discount (up to 90%) and are ideal for fault-tolerant, interruption-tolerant workloads.
</details>

**Q2.** What is the key difference between an Elastic IP and a standard auto-assigned public IP?
- A) Elastic IP is faster
- B) Elastic IP is static and persists across stop/start; auto-assigned IP changes
- C) Elastic IP is free; auto-assigned IP costs money
- D) Elastic IP works with IPv6; auto-assigned does not

<details><summary>Answer</summary>
**B** — Elastic IP is a static public IP you own. Auto-assigned public IP changes when the instance is stopped and restarted. Elastic IPs also cost money when idle, so "free" (C) is wrong.
</details>

**Q3.** Why does an EC2 instance in a newly created VPC have no internet access by default, even after attaching an Internet Gateway?
- A) The IGW must be enabled per subnet
- B) You also need a route table with a route to `0.0.0.0/0` via the IGW, associated with the subnet
- C) The instance needs a public IP to reach the internet
- D) Both B and C

<details><summary>Answer</summary>
**D** — internet access requires both a route to the IGW (route table) and a public IP on the instance. The IGW alone does nothing without routing, and a public IP alone cannot route to the internet without an IGW.
</details>

**Q4.** If an Elastic IP is allocated but not associated with any running instance, what happens?
- A) Nothing — it's free until used
- B) AWS charges a per-hour fee for idle Elastic IPs
- C) The EIP is automatically released after 24 hours
- D) The EIP is associated with the next instance launched

<details><summary>Answer</summary>
**B** — AWS charges for idle Elastic IPs as a cost-optimization mechanism. Release unused EIPs to avoid unnecessary charges. They do not auto-release (C) or auto-associate (D).
</details>

**Q5.** A Solutions Architect creates a security group with no inbound rules and default outbound rules. Can an instance in this security group receive any traffic?
- A) Yes — all traffic is allowed by default
- B) No — with no inbound Allow rules, all inbound traffic is implicitly denied
- C) Yes — but only ICMP traffic
- D) No — the instance cannot communicate at all

<details><summary>Answer</summary>
**B** — Security Groups default to deny all inbound. Without explicit Allow rules, no inbound traffic reaches the instance. Outbound is allowed by default (the instance can initiate outbound connections), but responses are only allowed because Security Groups are stateful.
</details>

---

## Related Services

[[IAM]] · [[EC2]] · [[EBS]] · [[Auto-scaling]] · [[ELB]] · [[VPC]]
