# Route Tables — Where VPC Traffic Goes

## What it is

A route table is the **routing decision-maker** of a VPC — a set of rules (routes) telling each subnet where to send traffic for a given destination. Every subnet **must** be associated with exactly **one** route table. The **main route table** is the default for subnets with no explicit association.

## Route evaluation

- Routes are evaluated by **destination CIDR** — most specific match wins (longest prefix). There's no first-match ordering like NACL.
- Each route: **Destination** (CIDR or prefix list) → **Target** (IGW, NAT, endpoint, peering, TGW, etc.).
- Traffic toward the VPC's own CIDR always has a localized route (`local`) — created automatically, cannot be deleted.

```
Destination               Target
10.0.0.0/16               local          ← always present
0.0.0.0/0                 igw-12345678   ← internet
172.16.0.0/16             pcx-abcdef     ← peered VPC
pl-91d7dee1               vpce-xxxx      ← Gateway Endpoint (S3)
```

## Route table types & association

| | Main route table | Custom route table |
|---|---|---|
| Created | Automatically with VPC | By you |
| Default for | Subnets w/o explicit association | Only subnets you attach |
| Editable | Yes (but it's the fallback for every unassociated subnet) | Yes |
| When IGW/custom added | VPC keeps Main for unassigned | — |

- **Association is 1:1** — a subnet can only have one route table, but one route table can serve **many** subnets (all with same AZ scoping duplication).
- Route table changes take effect **immediately** for all associated subnets — a fast way to cut off or open traffic.

## Common routes & targets

| Target | When you need it |
|---|---|
| **IGW** (`igw-...`) | Public internet access (public subnet) |
| **NAT** (`nat-...`) | Private subnet → outbound internet |
| **VPC Peering** (`pcx-...`) | Reach a peered VPC's CIDR |
| **Transit Gateway** (`tgw-...`) | Hub for many VPCs / on-prem |
| **Gateway Endpoint** (`vpce-...`) | Private route to S3 / DynamoDB (prefix list destination) |
| **Local** | Always — VPC's own CIDR |
| **Network Interface / Instance** | Route a specific host via an instance (e.g. NAT instance, appliance) |

## Exam domains

- [x] **Secure (30%)** — keeping private subnet route from IGW, correct target choices
- [x] **Resilient (26%)** — multi-AZ redundancy, same routing across AZs via one table
- [x] **High-Performing (24%)** — most-specific-match routing, prefix lists for endpoints
- [x] **Cost-Optimized (20%)** — no extra cost; Gateway Endpoint routes avoid NAT charges

## Key gotchas

1. **Longest prefix wins** — `/0` is lowest priority, specific CIDRs override it
2. A subnet has exactly **one** route table; multiple subnets share one easily
3. **Main route table** silently applies to any subnet you forget to associate
4. There's always a **local route** for the VPC CIDR — cannot be removed or out-prioritized
5. Route to a NAT GW/IGW that's in a **different AZ** still works (NAT GW redundant per AZ recommended for HA)
6. **Gateway Endpoints** (S3/DynamoDB) use **prefix lists** as destination — most-specific matching picks them over NAT
7. Route table updates apply **instantly** — no instance reboot needed
8. Adding/removing a route can briefly cause traffic blackholes if misconfigured
9. Peering needs routes **in both VPCs** (both route tables) pointing at each other — and no overlapping CIDRs
10. Default route `0.0.0.0/0 → IGW` makes a subnet public only if instances also get **public IPs + SG allow**

## Related services

- **VPC** — the network everything routes within
- **Subnets** — the association that links route tables to AZs
- **IGW / NAT GW / Gateways endpoint** — the route targets
- **Transit Gateway / VPC Peering** — routes to other networks
- **S3 / DynamoDB** — reachable via Gateway Endpoint prefix-list routes