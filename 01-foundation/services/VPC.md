# VPC — Virtual Private Cloud

## What it is

A VPC is your **isolated virtual network** inside AWS — logically private, spans all AZs in a region, and is defined by its **CIDR block** (e.g. `10.0.0.0/16`). Everything you deploy (EC2, RDS, ELB) lives inside a VPC. On the exam, VPC questions test whether you understand subnets, routing, internet egress, and the firewall layers (NACL + SG).

## Core building blocks

| Component | Purpose |
|---|---|
| **VPC** | Isolated network, region-wide, has a CIDR |
| **Subnet** | AZ-scoped slice of the VPC CIDR — each subnet lives in **one AZ** |
| **Route table** | Controls where traffic goes per subnet |
| **Internet Gateway (IGW)** | Enables public internet in/out for a VPC |
| **NAT Gateway / Instance** | Outbound-only internet for private subnets |
| **Security Groups** | Stateful instance firewall |
| **NACL** | Stateless subnet firewall |
| **Gateway endpoints** | Private access to S3/DynamoDB without internet |

## Subnets & routing

- Route tables are **associated with subnets**; a subnet can have only **one** route table.
- **Main route table** (default) applies to subnets without an explicit association.
- Public subnet: route `0.0.0.0/0 → igw-xxxx`. **Private subnet**: no IGW route.

```
Public subnet:   10.0.1.0/24 → route 0.0.0.0/0 → IGW
Private subnet:  10.0.2.0/24 → route 0.0.0.0/0 → NAT GW (in a public subnet)
```

- **Not all subnets in a route table are public** — only those with an IGW route + instances with public IPs/SGs allowing ingress.
- Same AZ subnets can still be public or private — yes.

## Internet access options

| Option | What it gives | Direction |
|---|---|---|
| **IGW** | Full bidirectional internet | Both |
| **NAT Gateway** (managed) | Outbound internet from private subnet | Out only |
| **NAT Instance** (self-managed) | Same as above but a manual EC2 — must disable source/dest check | Out only |
| **Egress-only IGW** (IPv6) | Outbound-only for IPv6 | Outer only |
| **Gateway Endpoint** | Private VPC→S3/DynamoDB, no NAT/IGW | Private |

- **NAT GW is redundant per AZ** — one per AZ for HA; needs an EIP + public subnet.
- **NAT can't be used for IPv6** — use Egress-only IGW instead.
- Gateway Endpoints: **free**, no internet path needed, use prefix lists (pl-xxxxx).

## Firewall layers (order matters)

1. **NACL** — stateless, subnet-level, first check (inbound + outbound rules)
2. **Security Group** — stateful, instance-level

- Traffic must pass **both** layers. NACL deny is evaluated first.
- SG stateful → responses auto-allowed; **NACL stateless** → must allow return traffic on ephemeral ports.

## VPC size & IP planning

- VPC CIDR: `/16` to `/28` per VPC; up to **5 VPCs** per region (default).
- **Reserved by AWS per subnet**: first 4 + last 1 IPs (e.g. `.0.1.2.3` + `.255`) plus... — actually 5 total: `+0`, `+1`, `+2`, `+3`, `+255` are unavailable.
- **Secondary CIDRs** can be added if you run out (they're just extra blocks).
- **IPv6**: optional — either AMAZON-provided or your own (BYOIP).

## Peering & connectivity

| Connection | Use |
|---|---|
| **VPC Peering** | Direct connect between 2 VPCs — **no transitive peering** |
| **Transit Gateway** | Hub-and-spoke — connects many VPCs/on-prem (transitive) |
| **VPN / Direct Connect** | Hybrid — on-prem to AWS |
| **Gateway/Interface endpoints** | Private access to AWS services |

- **Peering is not transitive**: A↔B and B↔C does **not** give A↔C.
- Peered VPCs must not have **overlapping CIDRs**.

## Exam domains

- [x] **Secure (30%)** — NACL + SG layers, endpoints, private subnets, no public egress
- [x] **Resilient (26%)** — multi-AZ subnets, NAT per AZ, cross-AZ redundancy
- [x] **High-Performing (24%)** — Gateway Endpoints for S3, Transit Gateway for scale
- [x] **Cost-Optimized (20%)** — NAT GW vs Endpoint pricing, Gateway Endpoint is free

## Key gotchas

1. Subnet = **one AZ**; VPC = **all AZs** in the region
2. **5 reserved IPs** per subnet (first 4 + last) — cannot use them
3. Route tables: subnet has only **one**; empty main RT = subnet pieces can be public
4. **NACL stateless** + SG stateful — both must allow traffic
5. **VPC Peering is not transitive**
6. **NAT Gateway can't do IPv6** — Egress-only IGW for that
7. Default VPC allows everything — custom VPC starts locked down
8. **Delete on IGW detach**: disassociate public IPs, then detach
9. **VPC Flow Logs** capture allowed/denied traffic at the ENI level — useful to see what NACL vs SG dropped

## Related services

- **EC2** — instances that live in subnets
- **Security Groups / NACLs** — firewall layers
- **Route53** — DNS resolution for VPC resources
- **ELB** — front-ends traffic outside/inside the VPC
- **CloudWatch** — Flow Logs for traffic visibility
- **S3 / DynamoDB** — reachable via Gateway Endpoints