# Hybrid Connectivity Course — VPN, Direct Connect, Transit Gateway

## 1. Purpose

Hybrid connectivity links **on-premises networks to AWS** securely and with the right bandwidth/latency characteristics. AWS gives you three main paths: the public internet (VPN), a managed private line (**Direct Connect**), and the hub service that scales these connections out to many VPCs (**Transit Gateway**). Choosing correctly is a core SAA-C03 "network connection options" skill — the choice is driven by security, bandwidth, latency, and budget.

## 2. How it works

- **Site-to-Site VPN (S2S VPN)** — encrypted IPSec tunnels over the public internet between your on-prem **customer gateway (CGW)** and an AWS-side endpoint: a **virtual private gateway (VGW — attached to one VPC)** or a **Transit Gateway (TGW — many VPCs)**. Each VPN has **two tunnels** for redundancy; standard tunnels up to **1.25 Gbps** each, "Large Bandwidth Tunnels" up to **5 Gbps** on TGW/Cloud WAN
- **Direct Connect (DX)** — a **physical cross-connect** from your data center (via an AWS partner or colocation facility) to an AWS Direct Connect location, terminating on a **virtual interface**
  - **Private VIF** → connects to a VGW/TGW → private VPC IPs
  - **Public VIF** → public AWS endpoints (S3, DynamoDB, SQS) without internet
  - Speed tiers 1–100 Gbps (dedicated) or hosted (fractional, via partners); DX does **not encrypt** by default → pair with a VPN tunnel over it ("VPN over DX") for encryption
- **Transit Gateway (TGW)** — a **regional hub-and-spoke router**: one attachment point for many VPCs + on-prem (VPN/DX), **transitive routing** between all spokes, ECMP across multiple links, and centralized egress/inspection VPC patterns
- **VPC Peering** — direct 1:1 connection, **not transitive**, overlapping CIDRs forbidden; fine for a few VPCs, wrong tool at scale
- **AWS Cloud WAN** — the global, managed multi-region/multi-account version of TGW (beyond core exam scope, know it exists for wide-area spoke-to-spoke across regions)
- **Client VPN** — remote users connecting in (vs site-to-site) — separate concept, occasionally appears

```
On-prem ──(public internet)── S2S VPN (IPSec) ── VGW (1 VPC) or TGW (many VPCs)
On-prem ──(private fiber)── Direct Connect ── Private VIF ── VGW/TGW ── VPCs
                                       └ Public VIF ── S3/DDB public endpoints
On-prem ── DX + VPN-on-top ── encrypted + private + stable
```

## 3. When to use

- **Site-to-Site VPN**: quick to provision, cheap, encrypted, over existing internet — most hybrid setups start here; good for backup/secondary path
- **Direct Connect**: stable, predictable, high-bandwidth (1–100 Gbps), low-latency, when you move lots of data or need SLA-grade networking or avoid internet egress costs; also for **public cloud services over private path** (Public VIF)
- **Transit Gateway**: many VPCs and on-prem connected with transitive routing, centralized security/egress VPC, ECMP over multiple DX/VPN links
- Combine **DX (bandwidth) + VPN over it (encryption)** for the full-best-practice hybrid data path

## 4. When NOT to use

- A single VPC and simple needs → **peering** is cheaper and simpler than a TGW
- No on-prem, all-in-AWS → none of these are needed
- Small, infrequent data movement and internet is fine → VPN is overkill vs plain public endpoints; and internet + HTTPS may suffice
- **Latency/stability isn't critical** and budget is tight → pick VPN over Direct Connect (DX has long setup, port fees, and is pricier)
- Encryption isn't required and public path unacceptable — hmm: if traffic can't traverse the internet at all (compliance), that's DX (and usually VPN-over-DX); if public internet is acceptable, VPN suffices
- Large multi-region global mesh without on-prem → Cloud WAN / transit patterns rather than cascading DX

## 5. Important features

- **S2S VPN**: two encrypted tunnels per connection, IPSec/IKEv2, static routes or BGP (dynamic), route propagation to VPC route tables; VGW must attach to a single VPC (no IPv6 on VGW — use TGW/Cloud WAN)
- **Direct Connect**: dedicated (1–100 Gbps) or hosted (slower, partner-managed); Private VIF (VPC/VGW/TGW) vs Public VIF (public AWS endpoints); Standard Direct Connect location (AWS colo) vs AWS Outposts; DX Gateway can attach multiple VGWs/VPCs (in multiple accounts/regions) — but only one AWS region per DX connection historically, DX Gateway solves multi-region
- **Transit Gateway**: transitive hub, per-attachment billing, ECMP (equal-cost multi-path) across up to 10 tunnels/connections, regional, route tables per TGW scope, central inspection/egress VPC patterns, inter-region peering of TGWs
- **VPN CloudHub** — hub-and-spoke VPN between multiple on-prem sites over AWS (simple transit)
- **Accelerated site-to-site VPN** — uses AWS Global Accelerator endpoints to improve latency/stability over internet for long-distance paths (Global Accelerator integrates here)
- **Migration interplay**: DataSync, Storage Gateway, App Migration Service assume a working hybrid path; DX is commonly the backbone for large migrations

## 6. Limitations

- **VPN**: rides the public internet → variable latency/throughput; bandwidth ceiling (~1.25–5 Gbps/tunnel); **VGW has no IPv6** (needs TGW/Cloud WAN)
- **Direct Connect**: **no encryption by default** (encrypt with VPN over it); physical provisioning takes weeks, needs partner/colo facility and port fees; single connection can be a single point of failure (need redundant connections to multiple locations for HA)
- **DX + VGW**: one region per connection; multi-region needs DX Gateway or TGW
- **Peering isn't transitive** and CIDRs must not overlap — pitfall at scale
- **TGW**: per-AZ/hourly billing + data processing fees; regional (inter-region needs peering attachments)
- **Public VIF** still reaches AWS over AWS's network but through public endpoints (S3/DDB) — not private VPC IPs

## 7. Trade-offs

- **VPN vs Direct Connect** — cheap/fast-to-set/encrypted/internet-dependent vs expensive/slow-to-set/stable-high-bandwidth/unencrypted-by-default. Exam pattern: "quick, encrypted, low setup cost" → VPN; "consistent bandwidth + latency, large data, SLA" → DX
- **DX alone vs DX + VPN** — unencrypted/cheaper vs encrypted (VPN over it). Best practice: DX for path + VPN for encryption
- **VGW vs Transit Gateway** — single VPC, low cost vs many VPCs, transitive, ECMP, central egress
- **Peering vs TGW** — many peerings: O(n²), no transitive hubs vs a few TGW attachments, transitive, central control — TGW wins past a handful of VPCs
- **Single DX vs redundant DX (multi-site/location)** — cost vs HA (a single cable/location is a SPOF)
- **AWS owned lines vs partner (hosted DX)** — dedicated (control, $, lead time) vs hosted (flexible rates, faster, limited control/contract)
- **Public internet vs Public VIF** — internet path with NAT/IGW egress costs vs private path to S3/DDB (but still public endpoints)

## 8. Architecture

Reference hybrid patterns:

```
Pattern A — simple hybrid (one VPC):
  On-prem CGW ←IPSec→ VGW → VPC subnets (route propagation: on-prem CIDR → VGW)

Pattern B — scale hybrid (many VPCs + on-prem):
  On-prem ── DX (ECMP, 2+ connections) & VPN (backup) ── Transit Gateway
              ├─ VPC-A (app)  └ attachments
              ├─ VPC-B (data)
              └─ Inspection VPC (centralized firewalls) for all egress
  Best practice: VPN over DX for encryption; VPN over internet as DR backup

Pattern C — remote sites:
  Replicated Site(s) → AWS (VPN CloudHub / Cloud WAN) → central egress/DR
```

## 9. SAA-C03 Perspective

"Network connection options" is an official exam skill (Domains 1 & 4). Expected proficiency:

- **Scenario→VPN**: "encrypted connection over the internet", "fast deploy", "backup/secondary path", "low cost" 
- **Scenario→Direct Connect**: "requires stable consistent latency/bandwidth", "large data transfer", "dedicated/private line", "bypass internet", **"SLA-grade"**; and pair with VPN for encryption
- **Scenario→Transit Gateway**: "connect multiple VPCs + on-prem centrally", "transitive", "hub-and-spoke", "ECMP across multiple connections", when VPC peering can't scale
- **Scenario→VPC Peering**: one or two VPCs only, simple direct links
- **Scenario→VGW vs TGW** for the S2S VPN endpoint; VGW = single VPC
- **Direct Connect**: Private VIF for VPCs, Public VIF for S3/DDB endpoints, DX Gateway for multi-region/multi-account
- **Encryption**: VPN tunnels encrypt; DX doesn't → "encrypt in transit over DX" → VPN over the DX connection
- **Redundancy/HA**: two Site-to-Site tunnels, redundant DX connections, multi-AZ, ECMP on TGW — always pick redundancy options
- **Migration/data transfer** questions often assume hybrid connectivity (DataSync/Storage Gateway over DX)

Exam trap: "fastest to set up + encrypted + on a budget" → **Site-to-Site VPN** (not Direct Connect — it takes weeks). "Consistent high bandwidth, low latency, large transfers, but must be encrypted" → **Direct Connect + VPN over it**. "Many VPCs + multiple on-prem sites, hub" → **Transit Gateway**, never cascaded peering.