# AWS Global Accelerator Course — Anycast IPs on the AWS Global Network

## 1. Purpose

AWS Global Accelerator is a **networking service** that improves **availability and performance** for global applications by sending user traffic over the **AWS global network** instead of the public internet. You get **2 static, anycast public IPv4 addresses** (never change, so users and firewalls can hard-code them), and traffic from anywhere in the world is routed via the nearest AWS **edge location** to your application endpoint in the best-performing Region. It operates at **L3/L4** (TCP/UDP) — it is **not a cache and not an HTTP router**. For SAA it's the answer to **"static anycast IPs / route over AWS backbone / global failover for TCP-UDP / non-HTTP traffic"**.

## 2. How it works

- **2 static global anycast IPs** — announced from every AWS edge location; clients at any point on Earth connect to the nearest anycast address
- **AWS edge network → global backbone → your endpoint** — traffic travels inside AWS (reduces internet hops, packet loss, and jitter; ~60%+ latency improvement vs public internet)
- **Endpoint groups** — group endpoints (ALB | NLB | EC2 | Elastic IP | Global Accelerator-compatible GWLB) in one or more Regions
- **Health checks per endpoint** — Global Accelerator performs continuous health checks and **fails over to a healthy endpoint/Region** in seconds (or shifts via **traffic dials** 0–100% per endpoint group)
- **Static IP whitelisting** — one pair of IPs works even if you scale/change endpoints behind it
- **Client IP preservation** — supports preserving source IP to ALB/NLB (network interface-based)
- Works side by side with **CloudFront** (CloudFront for HTTP(S) edge delivery/caching, GA when you also need acceleration for dynamic and non-HTTP traffic)

```
Users worldwide → nearest AWS edge (anycast IP) → AWS global backbone
  → endpoint group (Region A: ALB/NLB/EC2/EIP) with traffic dial + health checks
  → on failure, traffic shifts to Region B endpoint group in seconds
```

## 3. When to use

- **Global user base** needing **TCP/UDP acceleration and reliability** (gaming, VoIP, IoT/MQTT, financial trading, file transfer)
- **Multi-Region failover / disaster recovery** — route around a Region outage quickly
- **Static IP whitelisting** — partners/customers allowlist a small set of IPs; you scale regions behind them
- **Reduce latency/jitter over long distances** — non-HTTP protocols where CloudFront's HTTP cache can't help
- **Global apps that can't tolerate connection drops** (long-lived sessions, WebSockets, streaming)

## 4. When NOT to use

- **HTTP(S) content caching/CDN** — that's **CloudFront** (GA does no caching)
- **Caching static content at edge** — CloudFront (GA is L4, no per-path routing)
- **Single-region, low-traffic internal apps** — no global need, no benefit
- **L7 routing** by path/host/cookies — use ALB (GA just forwards flows)
- **DNS-based routing with user-locality logic** → Route 53 (GA is anycast, not DNS records)

## 5. Important features

- **2 static anycast IPv4 addresses** per accelerator (consistent IPs across Regions; optional IPv6)
- **TCP/UDP (L3-L4)** acceleration over AWS backbone; TLS termination NOT offered (pass-through)
- **Endpoint groups + traffic dials (0–100%)** and endpoint weights — gradual rollouts/failover
- **Health checks** and fast cross-Region failover (seconds)
- **Improves performance** (less internet variability) and **reliability** (fewer routing flaps)
- **Client IP preservation**; works with ALB, NLB, EC2/EIP, GWLB
- **Wires with CloudFront**, Route 53, AWS Global Network; 99.99% SLA
- **Cheap and simple** — hourly IP + per-GB data transfer

## 6. Limitations

- **Not a cache / not L7** — can't cache content, route by path, or do HTTP features (use CloudFront/ALB for that)
- **Two fixed IPs only** (per accelerator) — you design around them
- Pricing based on **hourly per-IP + bandwidth** — small single-region apps may not justify it
- Accelerator adds a hop through AWS edge — for purely regional traffic the gain is small

## 7. Trade-offs

- **Global Accelerator vs CloudFront** — L4 static-IP acceleration for TCP/UDP & dynamic traffic vs L7 HTTP(S) CDN with caching/ACL at edge. **Best practice: use both** — CloudFront for static content, GA for dynamic/API/WebSocket/non-HTTP
- **Global Accelerator vs Route 53 latency/geo routing** — GA: static anycast IPs, failover in-seconds, enter AWS network early (performance + consistency) vs R53: DNS-based, TTL-dependent failover, no traffic "acceleration"
- **vs Direct Connect/VPN** — GA optimizes public traffic; DX/VPN give private dedicated links (different problem)

## 8. Architecture

```
Global games/VoIP stack:
  Static anycast IPs (edge) → AWS backbone
    → EU endpoint group: NLB/ALB (weight 100%)
    → US endpoint group: NLB/ALB (traffic dial 0% until failover)
  Health check on endpoints → outage in EU auto-fails-over to US
Pair with CloudFront for static assets; keep static IPs for client whitelisting
```

## 9. SAA-C03 Perspective

- **"Static anycast IPs" / "always-fixed IP addresses for whitelisting"** → **Global Accelerator**
- **"Route over AWS global network to improve latency/jitter for TCP/UDP (gaming, IoT, VoIP)"** → **Global Accelerator**
- **"Fail over an entire Region in seconds using health checks"** → **Global Accelerator endpoint groups + traffic dials**
- **"Global HTTP content caching / CDN"** → NOT GA — **CloudFront**
- **"Accelerate WebSockets/dynamic traffic AND cache static"** → **GA + CloudFront together**

Exam traps: "Global Accelerator caches content" → **no, L4, no cache (that's CloudFront)**; "GA replaces Route 53" → **no, they complement**; "GA handles HTTP path routing" → **no**; "one IP you choose" → **two fixed anycast IPs are assigned by AWS**.