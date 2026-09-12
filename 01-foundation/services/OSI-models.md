# OSI Model — The Seven Layers Mapped to AWS Services

## What it is

Conceptual framework breaking network communication into **seven layers**. The exam doesn't ask to memorize the model — it asks you to **choose the right AWS service** based on which layer the problem operates at.

## The seven layers

### L7 — Application
Interacts with the user/app — understands URLs, HTTP headers, cookies, query params.
**Protocols:** HTTP, HTTPS, FTP, SMTP, DNS, SSH, WebSocket
**AWS:** **ALB** (path/host/header routing), **API Gateway** (managed APIs), **CloudFront** (edge caching)
**Exam:** "route by URL path" → ALB · "managed API" → API Gateway · "cache at edge" → CloudFront

### L6 — Presentation
Formats, translates, **encrypts/decrypts** data.
**Protocols:** SSL/TLS, JPEG, JSON, XML, ASCII
**AWS:** **ACM** (certificates), **SSL/TLS termination on ELB**, CloudFront compression
**Exam:** "SSL/TLS certificates" → ACM · "encrypt traffic" → HTTPS listener + ACM cert

### L5 — Session
Opens/manages/closes sessions, maintains connection state.
**Protocols:** NetBIOS, RPC, PPTP
**AWS:** **Sticky sessions on ELB**, **Cognito** (user auth sessions), API Gateway usage plans/keys
**Exam:** "keep client on same backend" → sticky sessions · "auth sessions" → Cognito

### L4 — Transport
Reliable (TCP) or fast (UDP) transfer; manages ports. Does **not** understand HTTP content — only IP/port/protocol.
**Protocols:** TCP, UDP
**AWS:** **NLB** (raw TCP/UDP, ultra-low latency), **Security Groups** (IP+port+protocol filtering), TCP termination on ELB
**Exam:** "millions RPS, TCP/UDP" → NLB · "filter by port/protocol" → SG

### L3 — Network
**Routing** between networks by IP; NAT here.
**Protocols:** IP, ICMP, IPsec
**AWS:** **GWLB** (IP packet → virtual appliances), **Route Tables**, **NAT Gateway**, **Internet Gateway**, **VPC Peering / Transit Gateway**
**Exam:** "route between VPCs" → Peering/TGW · "private instances → internet" → NAT GW · "inspect with firewall appliance" → GWLB

### L2 — Data Link
Transfers data within same local network (MAC addresses); switches here.
**Protocols:** Ethernet, Wi-Fi, ARP, PPP
**AWS:** **ENI** (virtual NIC with MAC), **Subnets** (L2 broadcast domain)
**Exam:** "multiple network interfaces" → ENI · "same-subnet direct communication" → L2

### L1 — Physical
Actual physical transmission — cables, fiber, radio.
**Medium:** Cat5/6, fiber, Wi-Fi, hubs
**AWS:** **Data centers**, **Direct Connect** (dedicated physical fiber), **Global Accelerator** (AWS private backbone vs public internet)
**Exam:** "dedicated physical connection" → Direct Connect · "private AWS fiber" → Global Accelerator

## Quick reference

| Layer | Name | AWS services | When |
|---|---|---|---|
| 7 | Application | ALB, API Gateway, CloudFront | HTTP-aware routing/APIs/caching |
| 6 | Presentation | ACM, SSL termination | Encryption, certs |
| 5 | Session | Sticky sessions, Cognito | Session persistence, auth |
| 4 | Transport | NLB, Security Groups | TCP/UDP, port filtering, extreme perf |
| 3 | Network | GWLB, Route Tables, NAT, IGW, TGW | IP routing, NAT, appliance inspection |
| 2 | Data Link | ENI, Subnets | Multi-NIC, same-subnet comms |
| 1 | Physical | Data centers, Direct Connect | Physical connectivity |

## Exam scenarios

- "URL path routing" → **L7 ALB**
- "Millions TCP conns/sec with static IPs" → **L4 NLB**
- "Third-party firewall inspection" → **L3 GWLB**
- "Closest region over AWS private fiber" → **Global Accelerator**
- "Central SSL cert management" → **ACM (L6)**

## Encapsulation

Data descends layers, each adding a header (L7→L6→L5→TCP→IP→Ethernet→signals); receiver strips headers (**de-encapsulation**). Not tested directly, but explains why NLB (L4) can't do path routing — it has no access to HTTP headers.

## Exam domains

- [x] **Secure (30%)** — SG (L4), NACL (L3/L4), SSL termination (L6), WAF (L7)
- [x] **Resilient (26%)** — LB selection, cross-AZ routing
- [x] **High-Performing (24%)** — matching layer to performance needs

## Key gotchas

1. **ALB = L7, NLB = L4** — the #1 mapping. NLB faster because it inspects less
2. **Security Groups filter at L4** (IP+protocol+port); they don't inspect content — that's WAF (L7)
3. **NACLs** are lower level than SGs — stateless, ordered rules, explicit Deny
4. **GWLB = L3 (GENEVE)** — transparent IP packet encapsulation to appliances
5. **Sticky sessions = L5 concept** — connection state, not content
6. **SSL/TLS conceptually L6** — ACM / SSL termination
7. **Direct Connect = L1** physical fiber; **VPN = L3** IPsec over internet
8. **Global Accelerator** uses AWS private backbone (not public internet)

## Related services

- **ALB** — L7 HTTP routing
- **NLB** — L4 TCP/UDP
- **GWLB** — L3 virtual appliance integration
- **VPC** — implements L1-3
- **Security-Groups** — L4 firewall
- **ACM** — L6 cert management