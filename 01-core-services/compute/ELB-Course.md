# ELB Course — Elastic Load Balancing

## 1. Purpose

ELB is AWS's **managed load balancer**: it automatically distributes incoming traffic across multiple targets (EC2 instances, containers, IPs, Lambda) in one or more AZs, runs health checks, and only routes to **healthy** targets. It's the front door for making applications **highly available, scalable, and fault-tolerant**.

Four types:

| Type | OSI layer | What it does |
|---|---|---|
| **ALB** (Application) | Layer 7 | HTTP/HTTPS/gRPC — request-level routing (path, host, headers, method, query) |
| **NLB** (Network) | Layer 4 | TCP/UDP/TLS — ultra-low latency, static IPs, millions of conns/sec |
| **GWLB** (Gateway) | Layer 3 | Transparent insertion of third-party appliances (firewalls, IDS/IPS) |
| **CLB** (Classic) | L4/L7 | Legacy — AWS recommends migrating to ALB/NLB |

## 2. How it works

- Listeners receive traffic on a port and route it to **target groups** using rules
- **ALB**: terminates TLS, parses the HTTP request, routes by content (path `/api`, host, header, query) → can split traffic across microservices. Uses round-robin + least outstanding requests
- **NLB**: forwards TCP/UDP connections with a **flow hash** (5-tuple); preserves client **source IP**; one **static IP per AZ** (optionally Elastic IPs); TLS passthrough or termination
- **GWLB**: acts as a transparent gateway; all packets (all ports) go through the appliance fleet via **GENEVE on port 6081**, scale appliances along with traffic
- **Cross-zone load balancing**: ALB on by default (spreads evenly across AZs); NLB off by default
- **Health checks** run on a cadence; unhealthy targets are removed; ALB also checks targets' connection thresholds
- Only healthy targets receive traffic → the ASG and LB work together

```
Internet → ALB (L7) → Target Group → EC2/ECS/Lambda
Internet → NLB (L4, static IP) → Target Group → EC2/IP
Internet → GWLB → GENEVE(6081) → Firewall/IDS appliance fleet
```

## 3. When to use

- **ALB** — any HTTP/HTTPS/gRPC/WebSocket app, API, microservices (path-based routing), containerized apps (ECS/EKS), and **Lambda targets**; built-in Cognito/OIDC auth; WAF integration
- **NLB** — **extreme performance** (sub-ms latency, millions of req/s), non-HTTP protocols (TCP/UDP), or when you need **static/Elastic IPs** (whitelisting, DNS A records); TLS passthrough; fronting an **ALB for static IPs** ("ALB-as-target")
- **GWLB** — insert/scale virtual **firewalls, IDS/IPS, deep packet inspection** across the VPC, or across VPCs via PrivateLink GWLB endpoints
- Always needed when: multiple instances serve the same workload and you want HA/failover across AZs

## 4. When NOT to use

- Single instance / single point — one server needs no LB
- Only need DNS failover across regions — that's **Route 53** (especially with health checks)
- Serverless gateway-only apps (API Gateway direct to Lambda without ALB)
- Zone apex / cross-region global load balancing — use Route 53 or **Global Accelerator** (or NLB with static IPs)
- **CLB** — legacy only; don't build new architectures on it

## 5. Important features

- **Managed & highly available** — AWS scales the LB automatically; deploy across ≥2 AZs
- **Health checks** — configurable path/port, interval, thresholds; routes around unhealthy targets
- **Target types**: instance ID, IP, or Lambda (ALB also ECS/EKS service targets)
- **Content-based routing** (ALB): path, host, header, query string, HTTP method — up to ~100 rules
- **ALB integrations**: Cognito/OIDC auth, AWS WAF, WebSockets, HTTP/2, gRPC, sticky sessions (cookies), connection draining, X-Forwarded-For
- **NLB**: static IPs per AZ, Elastic IPs, source IP preservation, TLS termination/passthrough, zonal isolation
- **Sticky sessions (session affinity)** — ALB and NLB
- **Deletion protection, cross-zone LB toggles, connection draining/deregistration delay**
- **Pricing** — ALB by LCUs (latency, load-balanced requests, active/new connections + bytes); NLB by NLCU/h; GWLB per hour + GLCU + bytes

## 6. Limitations

- **ALB cannot serve non-HTTP protocols** (no raw TCP/UDP)
- **ALB has no static IP** — DNS name only (`.elb.amazonaws.com`); pair with NLB/Global Accelerator if you need fixed IPs
- **ALB doesn't preserve source IP at network layer** — uses `X-Forwarded-For` header (must be trusted/proxied)
- **Listener/rule limits**: 50 listeners, ~100 rules per LB
- **LB ≠ horizontally infinite** — needs proper subnet sizing (recommended /27, 8 free IPs per subnet)
- **Single-AZ LB = single failure risk** — must deploy across AZs
- **GWLB requires GENEVE-capable appliances** (6081); IPv6 encapsulation considerations
- **CLB is deprecated-grade** — limited features, must migrate

## 7. Trade-offs

- **ALB vs NLB**: rich content routing + auth + WAF vs raw speed + static IP + source IP + TCP/UDP
- **ALB vs Route 53**: request-level distribution within a region vs DNS-based failover across regions
- **NLB vs Global Accelerator**: NLB = regional static IPs; GA = global anycast static IPs with AWS edge routing
- **LB + ASG**: LB routes to *healthy* targets; ASG adds/removes instances — work as a pair for HA
- **Cross-zone on/off**: even distribution (ALB default) vs zonal traffic containment (NLB for stateful per-AZ apps)
- **Proxy (ALB) vs passthrough (NLB)**: TLS termination & features vs preserving raw connection & IP, lower latency

## 8. Architecture

Reference web-app pattern:

```
Internet
   ↘ Route 53 (DNS, optional) 
        ↘ ALB (L7, TLS term, WAF)
             → Target Group → Auto Scaling Group → EC2/ECS [AZ-A, AZ-B, AZ-C]
                                    ↘ RDS Multi-AZ
App servers must allow only the ALB's SG (port 80/443) — nothing else.

Pattern for fixed IP / hybrid:
Internet → NLB (static EIP per AZ) → ALB ("ALB-as-target") → targets
Internet → GWLB endpoint → GWLB → NVA fleet → next-hop route table for VPC inspection
```

## 9. SAA-C03 Perspective

The exam **heavily tests ALB vs NLB vs GWLB vs Route 53** choice:

- **Scenario→ALB**: HTTP/HTTPS app, microservices with path/host routing, ECS/EKS or Lambda targets, WebSockets, need auth/WAF, sticky sessions
- **Scenario→NLB**: sub-ms latency, TCP/UDP non-HTTP, static/Elastic IPs, source IP preservation, TLS passthrough, cross-region/private-link patterns
- **Scenario→GWLB**: "insert a fleet of third-party **firewalls/IDS/IPS** transparently" → GWLB with GENEVE, appliance in service provider VPC
- **Scenario→Route 53 instead**: failover across regions, weighted routing — not a per-request LB
- **Scenario→Global Accelerator**: global static anycast IPs, edge-optimized routing for international users
- Know **cross-zone LB**, **health checks driving ASG**, and **connection draining** for zero-downtime rolling deploys
- Know **SG chaining**: app SG only allows the LB SG (security domain questions)

Exam trap: "static IP for on-prem whitelisting + TCP protocol" → **NLB**, not ALB. "Route to `/api` and `/images` services" → **ALB**. "Fleet of firewalls" → **GWLB**.