# Route 53 Course  DNS & Failover

## 1. Purpose

Route 53 is AWS's **managed DNS (Domain Name System) + domain registration + health checking** service. It translates domain names (`example.com`) into IP addresses, routes users to the right endpoint worldwide, and  critically  can **fail over to a healthy secondary resource** when one goes down. "53" = DNS port 53.

## 2. How it works

- Two jobs: **public DNS hosting** (maps names→records) and **domain registration** (buy/manage domains)
- A **hosted zone** holds your records (e.g. `example.com.` zone with A, AAAA, CNAME, MX, TXT records)
- Resolvers query through the **AWS global anycast network** (low latency, 100% SLA)
- **Records** point to resources **Alias records** (Route 53 special) point at AWS resources (ELB, CloudFront, S3, API GW) directly  free, no extra health check needed, auto-tracks IP changes
- **Health checks** poll endpoints regularly from global checker locations (not at query time) after N consecutive failures the endpoint is marked unhealthy and Route 53 stops returning it
- **Routing policy** decides *which* healthy record responds to a query

```
User query example.com → DNS → Route 53 (anycast) → routing policy + health checks
  → returns the right health endpoint IP
Example: failover policy → primary healthy? → primary. else → secondary.
```

## 3. When to use

- **DNS hosting** for your domain (public or private VPC DNS)
- **Domain registration**  buy/transfer `.com` etc. and manage records in one place
- **Cross-region/global failover & DR**  the classic failover-routing policy
- **Routing by latency**  send users to the fastest AWS region
- **Weighted rollouts / A/B testing**  split traffic between versions or blue/green
- **Geographic routing**  by country/continent, or geo-proximity to your resources
- **Multi-value + health checks**  simple per-request load spreading across healthy IPs
- **Private DNS**  resolution between VPC resources via private hosted zones (works with Direct Connect, shared VPCs)

## 4. When NOT to use

- **Per-request load balancing within a region**  that's an **ALB/NLB** (DNS results are cached LB shares load per request)
- **Deploying latency-based edge routing with static global IPs**  that's **Global Accelerator** (anycast IPs, not DNS)
- Layer-7 routing, TLS termination, sticky sessions, WAF  **ALB/API Gateway**, not Route 53
- Simply buying a domain and pointing it at a static IP  any registrar + public DNS works use Route 53 for the managed years + failover
- Health-checking private/VPC resources directly  Route 53 checks public endpoints private resources need a Lambda/CloudWatch alarm workaround

## 5. Important features

- **Routing policies**:
  - **Simple**  one record, one resource (no health checks)
  - **Weighted**  send X% of traffic per record (A/B, blue/green)
  - **Failover**  active-passive: primary healthy → primary, else secondary (needs health check)
  - **Latency**  route to region with lowest latency for the user
  - **Geolocation**  route by user's country/continent (can deny/route specific regions)
  - **Geo-proximity**  route by resource location + bias to shift traffic
  - **Multi-value answer**  return up to **8 healthy random** records (client-side load spreading + health checks)
- **Alias records**  point to ELB/CloudFront/S3/API GW/other R53 records free, auto-heals, `Evaluate Target Health`
- **Health checks**  HTTP/HTTPS/TCP, string matching (first 5120 bytes), threshold ~3 failures can check **CloudWatch alarms** invert option
- **Active-active** (weighted/latency/geolocation + health checks) vs **active-passive** (failover policy)
- **TTL control**  lower TTL = faster failover, higher TTL = cheaper/more caching
- **Private hosted zones**  internal DNS in VPCs
- **AWS global anycast**  100% availability SLA on DNS durable, authoritative
- **Resolver / Route 53 Resolver**  hybrid DNS forwarding between VPC and on-prem
- Integration with ACM (certificates for your domains), CloudFront, ELB

## 6. Limitations

- **DNS-level, not request-level**  resolution happens at query time results get cached per TTL (slow failover without low TTL)
- **No health checks on Simple routing**
- **Records without health checks are always considered "healthy"**  easy to get wrong mixing policies
- **Geolocation can misroute** users on VPNs/offshore or if no match (falls through to default)
- **Spreading is random/weighted**, not algorithmically balanced like ALB (elliptical routing spread multi-value returns random healthy subset)
- **Health checks bill** per check/month and have check limits checking private resources needs workarounds (CloudWatch alarm-based checks)
- Not a load balancer  no keep-alive, sticky session, TLS termination it only returns DNS answers
- DNS answer size limits (UDP/TCP) 50 hosted zones default (soft limit)

## 7. Trade-offs

- **Route 53 vs ALB**  DNS routing + global failover vs per-request HTTP load balancing in-region. Often both: R53 at the edge → ALB in the region
- **Route 53 vs Global Accelerator**  DNS name + cached/TTL vs **static anycast IPs** (for whitelisting, UDP, or when DNS change lag matters)
- **Low vs high TTL**  quick failover (more queries/cost) vs DNS caching efficiency
- **Latency vs Geolocation vs Geo-proximity**  measured latency (dynamic) vs legal/targeted routing (compliance) vs resource-location + bias control
- **Failover vs Multi-value vs Weighted**  active-passive DR vs active-active spreading vs controlled percentages
- **Managed DNS vs self-hosted BIND**  100% SLA + anycast + health checks vs running your own DNS servers (vendor choice/edge cases)

## 8. Architecture

Reference cross-region DR / failover pattern:

```
Route 53 (example.com)  failover policy → primary/secondary (Alias to ELB)
   ├─ Primary   us-east-1   ALB → ASG (healthy) ← health check passes
   └─ Secondary us-west-2   ALB → ASG (standby)  ← used only if primary fails
Both aliases: Evaluate Target Health = Yes (no manual health checks needed for ELB)
TTL: 30–50s for fast failover.
Weighted = blue/green: weight 100/0 → shift to 50/50 → 0/100.
Latency: multi-region active-active, users hit the closest healthy region.
```

## 9. SAA-C03 Perspective

Route 53 is **pure DR/HA exam territory** (Domain 2, Resilient):

- **Scenario→Failover policy**: active-passive DR  primary region down → route to secondary. Classic exam pattern
- **Scenario→Alias + Evaluate Target Health**: failover between ELBs/CloudFront/S3 without managing your own health checks
- **Scenario→Latency**: multi-region active-active where users should hit the fastest region
- **Scenario→Weighted**: canary/blue-green deploy or shifting traffic percent
- **Scenario→Geolocation**: compliance/regional hashing split **Geo-proximity** = routing by resource location with bias
- **Scenario→Multi-value**: return up to 8 healthy IPs with per-record health checks (active-active, simple)
- Know: **Route 53 is DNS, not a load balancer**  ALB only makes failover-testing questions trickier by adding target-group health checks
- **Failover speed** tied to TTL lower TTL for aggressive DR
- **Private hosted zones** for VPC internal DNS hybrid with Route 53 Resolver
- Route 53 sits in front of regional stacks: `R53 → Global Accelerator OR CloudFront → ALB → ASG`
- Be able to combine policies (latency at top, weighted below) in complex trees

Exam trap: "spread load between two ALBs per request with health checks and quick failover without a fixed single active primary" → **Multi-value or Weighted** "active-passive primary/secondary DR in another region" → **Failover policy** "static IPs for whitelisting across regions" → Global Accelerator, not Route 53 DNS.