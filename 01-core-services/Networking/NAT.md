# NAT — Network Address Translation

## What it is

NAT allows instances in **private subnets** to reach the internet (outbound) while remaining unreachable from the outside (no inbound). AWS offers two flavors: **NAT Gateway** (managed, recommended) and **NAT Instance** (self-managed EC2). For IPv6, use an **Egress-only Internet Gateway** instead.

## NAT Gateway vs NAT Instance

| | NAT Gateway | NAT Instance |
|---|---|---|
| **Managed by** | AWS | You (EC2 AMI) |
| **Performance** | Up to 45 Gbps | Depends on instance type |
| **Availability** | AZ-scoped; deploy **one per AZ** for HA | Can be placed in a public subnet, use SG for HA |
| **Maintenance** | Automatic software/OS patching | You patch and manage the OS |
| **Failover** | Automatic within AZ; cross-AZ requires separate GWs | Use ASG or route table failover script |
| **Cost** | Per-hour + per-GB processed | EC2 instance cost + data processing |
| **Bandwidth** | Scales to 45 Gbps per gateway | Limited by instance type |
| **Security** | Use **route tables** to control which subnets use it | Use **Security Groups** for fine-grained control |

## How it works

```
Private Subnet (10.0.2.0/24)
  Instance (10.0.2.50)
    → route 0.0.0.0/0 → nat-gw-xxxx (in public subnet)
      → IGW → Internet
        → response returns via NAT GW → instance
```

1. Instance sends outbound traffic to NAT Gateway (private subnet route table)
2. NAT GW replaces **source IP** with its own **EIP** (public IP)
3. Response comes back to NAT GW, which forwards to original private IP
4. **Stateful**: NAT GW tracks connections; only allows return traffic for initiated requests

## Deployment pattern (per AZ)

```
Public Subnet A:   NAT GW A (EIP-A) ← private subnet A routes here
Public Subnet B:   NAT GW B (EIP-B) ← private subnet B routes here
Private Subnet A:  route 0.0.0.0/0 → nat-gw-A
Private Subnet B:  route 0.0.0.0/0 → nat-gw-B
```

- **One NAT GW per AZ** — AZ-scoped; if you use one shared NAT GW, AZ failure breaks the other AZs
- **Cost saver**: use one NAT GW for non-critical workloads (dev/test), multi-AZ for prod

## NAT Instance specifics

- Must be an **AWS NAT AMI** (Amazon Linux or marketplace)
- **Disable source/dest check** on the instance (required for NAT to forward packets)
- Place in **public subnet** with a public IP
- SG must allow: inbound 80/443, outbound all
- Bandwidth scales with instance type (use `c5.xlarge` or larger for production)

## Egress-only Internet Gateway (IPv6)

- NAT Gateway/Instance **only works with IPv4**
- For IPv6 private instances that need outbound internet → use **Egress-only IGW**
- Stateful: only allows return traffic for outbound-initiated connections
- Added as a route target in the subnet route table

## NAT Gateway metrics (CloudWatch)

| Metric | Why it matters |
|---|---|
| **PacketsDropCount** | Drops due to full connection tracking — scale up |
| **ActiveConnectionCount** | Connection saturation |
| **ErrorPortAllocation** | No free ephemeral ports — scale or increase timeout |
| **BytesProcessed** | Cost driver — per-GB charges |

- Connection tracking: **default timeout 350 seconds**; adjustable 60–3600s per route

## Exam domains

- [x] **Secure (30%)** — private subnet egress without inbound exposure, SG control on NAT Instance
- [x] **Resilient (26%)** — one NAT GW per AZ for HA; AZ failure isolates from internet
- [x] **High-Performing (24%)** — NAT GW scales to 45 Gbps; instance type selection for NAT Instance
- [x] **Cost-Optimized (20%)** — NAT GW per-GB pricing vs single shared GW for dev; Gateway Endpoints (free) for S3/DynamoDB to avoid NAT costs

## Key gotchas

1. **NAT GW is AZ-scoped** — one per AZ; shared NAT GW = AZ failure breaks other AZs
2. **NAT Instance requires source/dest check disabled** — won't work without it
3. **NAT is IPv4 only** — use Egress-only IGW for IPv6
4. **Connection tracking limits** — NAT GW has soft limits; monitor `PacketsDropCount`
5. **NAT GW costs add up** — $0.045/hr + $0.045/GB; use Gateway Endpoints for S3/DynamoDB
6. **NAT Instance is single point of failure** unless paired with ASG/failover
7. **NAT GW doesn't support SG rules** — use route tables to control which subnets route through it
8. **NAT GW max bandwidth: 45 Gbps** — NAT Instance scales higher with right instance type
9. **Private subnets have no internet** — only through NAT; ensure route table has 0.0.0.0/0 → NAT GW
10. **Default timeout: 350s** — long-lived connections may drop; adjust if needed

## Related services

- **VPC** — subnets, route tables, IGW context
- **EC2** — NAT Instance is a self-managed EC2
- **Security Groups / NACLs** — firewall for NAT Instance
- **CloudWatch** — NAT GW metrics for monitoring
- **Gateway Endpoints** — free alternative for S3/DynamoDB (bypasses NAT)
- **Transit Gateway** — for cross-VPC connectivity without NAT
