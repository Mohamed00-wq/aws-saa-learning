# VPC Course  Virtual Private Cloud

## 1. Purpose

A VPC is your **logically isolated virtual network** inside AWS. It spans all AZs in a Region, is defined by a **CIDR block** (e.g. `10.0.0.0/16`), and plays the same role as your own data-center network  but as a managed service. Everything you deploy (EC2, RDS, ELB, Lambda-in-VPC) lives inside a VPC. It gives you full control over IP ranges, subnets, routing, and security while taking zero hardware maintenance off your plate.

```
10.0.0.0/16 (VPC, region-wide)
├── Public  subnet A (10.0.1.0/24)  az-a
├── Private subnet A (10.0.2.0/24)  az-a
├── Public  subnet B (10.0.3.0/24)  az-b
└── Private subnet B (10.0.4.0/24)  az-b
```

## 2. How it works

- A **VPC** = region-level container with one or more CIDR blocks (IPv4 `/16`–`/28` optional IPv6)
- **Subnets** slice the VPC CIDR and live in **exactly one AZ** (AZ-scoped)
- **Route tables** attached to subnets decide where traffic goes (most-specific prefix wins)
- **Internet access** is provided by components you add: **IGW** (public in+out), **NAT Gateway** (private out-only), **Egress-only IGW** (IPv6 out-only)
- **Security layers**: NACL (stateless, subnet) + Security Group (stateful, instance)  traffic must pass both
- **Private access to AWS services**: VPC Endpoints (Gateway for S3/DynamoDB, Interface via PrivateLink for everything else)

Creating a public subnet requires three things: an IGW attached to the VPC, a route `0.0.0.0/0 → igw-...`, and instances with public IPs + SG allow.

## 3. When to use

- Basically **always**  any EC2/RDS/ELB workload needs a VPC
- You need **isolation and IP control** (CIDR planning, no overlap with on-prem)
- **Hybrid networking**  connect to on-prem via VPN or Direct Connect
- **Multi-tier apps**  separate public (web) and private (app/db) subnets
- **Compliance/security**  private subnets, network ACLs, no public egress, flow logs
- **Multi-VPC architectures**  peering, Transit Gateway, central egress patterns

## 4. When NOT to use

- **Serverless-only apps** (pure Lambda + API Gateway + DynamoDB/S3)  *but* Lambda in a VPC is needed for RDS/private resources, so it still matters
- Fully-managed services you don't put inside your network (S3, DynamoDB, managed SaaS)  just use public endpoints or VPC endpoints
- AWS-managed/templated setups (default VPC, Elastic Beanstalk default) where you can live with the default network
- Rule: you can't escape VPC design on the exam  it underpins nearly every architecture question

## 5. Important features

- **CIDR & subnets**  plan ranges to avoid overlap **5 AWS-reserved IPs per subnet** (first 4 + last: `.0 .1 .2 .3 .255`), secondary CIDRs can be added later
- **Internet Gateway (IGW)**  1:1 with VPC, horizontally scaled, redundant, NATs public IPv4 IPv6 passes through directly
- **NAT Gateway**  private-subnet outbound internet 5 Gbps scaling to 100 Gbps, 1M→10M pps **AZ-scoped (deploy one per AZ for HA)** IPv4 (NAT64/DNS64 can do IPv6 now) can attach Security? No  route via route tables
- **Egress-only IGW**  outbound-only IPv6 for private subnets
- **Route tables**  longest-prefix match one per subnet main vs custom
- **Security Groups & NACLs**  the two firewall layers (stateful vs stateless)
- **VPC Endpoints**  **Gateway** (S3 + DynamoDB, free, uses prefix lists in route tables) vs **Interface** (PrivateLink, everything else, has ENI/pricing)
- **VPC Peering**  2 VPCs, **not transitive**, no overlapping CIDRs
- **Transit Gateway**  hub-and-spoke for many VPCs + on-prem, transitive
- **VPN / Direct Connect**  hybrid connectivity DX for stable high-bandwidth, VPN over internet
- **VPC Flow Logs**  capture accepted/dropped traffic per ENI (great for debugging + compliance)
- **VPC endpoints for shared responsibility**  keeping traffic on AWS network (never leaves it)

## 6. Limitations

- **Per-region**  VPCs don't span regions (need peering/Transit Gateway/global accel across regions)
- **CIDR planning is early/final**  growing VPC CIDR needs secondary blocks overlapping CIDRs break peering
- **5 reserved IPs eaten per subnet**  real design constraint
- **Default quotas**  up to 5 VPCs/region (default) IGW, NAT GWs per AZ, flows  all have limits (soft, increasable)
- **NAT GW is AZ-scoped**  a shared single NAT breaks on AZ failure (HA requires per-AZ deployment)
- **Peering isn't transitive**  no A→C routing via B (that's Transit Gateway)
- **IPv4 costs now**  public IPv4 addresses are billed NAT GW bills hourly + per-GB
- **No control of AWS core**  you manage the network config, AWS manages the underlying physical network

## 7. Trade-offs

- **Public vs private subnet**  direct internet (simpler, exposed) vs NAT egress (secure, cost + complexity)
- **Internet vs VPC Endpoints**  IGW/NAT (universal but public-path + per-GB NAT cost) vs Gateway Endpoint (free, private, but S3/DDB only) vs Interface endpoints (private, but per-ENI/hourly cost)
- **NAT Gateway vs NAT Instance**  managed (scaleable, billed by GB) vs DIY EC2 (cheaper at low volume, you patch/failover)
- **VPC Peering vs Transit Gateway**  simple 1:1 (cheap, no transitive) vs hub-and-spoke (scales, per-hour cost + transitive)
- **Direct Connect vs VPN**  dedicated secure line ($$, stable) vs internet VPN (cheap, variable)
- **One big VPC vs many small**  simplicity vs blast-radius isolation (multi-account/org patterns favor many)
- **Cloud-managed (default VPC) vs custom**  zero effort vs full control

## 8. Architecture

Reference 3-tier, multi-AZ VPC pattern:

```
                    VPC 10.0.0.0/16
   ┌───────────────────────────────────────────────┐
   │ Public-A 10.0.1.0/24   Public-B 10.0.3.0/24  │
   │   IGW ← 0.0.0.0/0      NAT-GW (EIP)          │
   │   ALB/Web tier                               │
   ├───────────────────────────────────────────────┤
   │ Private-A 10.0.2.0/24  Private-B 10.0.4.0/24 │
   │   App tier (ASG) → NAT-GW per AZ for updates │
   │   RDS Multi-AZ (port 3306, SG allows app)    │
   │   Route pl-s3... → Gateway Endpoint (S3)     │
   └───────────────────────────────────────────────┘
```

- 3-tier isolation: web (public) / app (private) / db (private) with SG references per tier, not IPs
- NAT GW per AZ in public subnets so app can patch without being reachable
- Gateway Endpoint for S3/DynamoDB  keeps costs down (no NAT per-GB) + no public path
- Route53 internal DNS for service discovery Flow Logs to CloudWatch/S3 for audit
- For multi-VPC: Transit Gateway hub, centralized egress VPC to save NAT costs

## 9. SAA-C03 Perspective

Networking is the **highest-yield topic** in Domain 1 (Secure Architectures). Know:

- **Subnet = one AZ** VPC = all AZs in the region 5 reserved IPs per subnet
- **Route tables**: longest-prefix match Gateway Endpoint prefix lists (pl-...) override `0.0.0.0/0` NAT
- **NACL stateless vs SG stateful**  allow explicit deny in NACL, ephemeral return ports, must pass both layers
- **IGW vs NAT GW vs Egress-only IGW vs Gateway/Interface Endpoint**  scenario routing questions
- **Private subnet outbound internet** → NAT Gateway (exam classic: "instances need patches but must not be reachable from internet")
- **One NAT GW per AZ** for HA single NAT GW = availability risk
- **Gateway Endpoint for S3/DynamoDB**  free + keeps traffic inside AWS Interface endpoints (PrivateLink) for other services
- **Peering not transitive** **overlapping CIDRs** break peering → Transit Gateway for scale
- **VPC Flow Logs**  debugging dropped traffic (confirm NACL vs SG)
- **Shared VPC / multi-account** patterns + centralized egress for cost

Exam trap: "instances in private subnet need internet for patches but no inbound" → **NAT Gateway**. "Private, free access to S3/DynamoDB" → **Gateway Endpoint** (not NAT, not PrivateLink-dependent). "Connect many VPCs + on-prem, transitive" → **Transit Gateway**, not peering.