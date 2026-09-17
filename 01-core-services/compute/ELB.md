# ELB — Elastic Load Balancing

## What it is

Elastic Load Balancing automatically distributes incoming application traffic across multiple targets — EC2 instances, containers, IP addresses, and Lambda functions — in one or more Availability Zones. ELB is the entry point for most web-facing architectures: clients connect to the load balancer's DNS name, and the load balancer routes each request to a healthy backend target.

ELB is the foundation of **high availability** for web applications. Without a load balancer, clients connect directly to a single instance — if that instance fails, the application is down. With ELB, traffic is distributed across multiple instances in multiple AZs, and unhealthy targets are automatically removed from rotation.

## Load balancer types

### ALB — Layer 7 (HTTP/HTTPS)
- **Content-based routing** (path `/api/*`, host, headers, query), WebSocket, HTTP/2, gRPC.
- **SSL/TLS termination**, ACM integration, **WAF** integration.
- **Lambda as targets**; round-robin or least-outstanding-requests.
- Cross-zone **enabled by default** (free).
- Use: microservices, containers, content-based routing, HTTP/HTTPS.

### NLB — Layer 4 (TCP/UDP/TLS)
- **Ultra-low latency**, millions of RPS. Forwards raw packets (no HTTP inspection).
- **Static IPs** (Elastic IP per AZ) — not DNS-based.
- **Preserves source IP** (ALB replaces it with the LB's IP).
- Long-lived TCP connections; cross-zone **disabled by default** (costs when enabled).
- Use: extreme perf, non-HTTP, static IP needed, preserve source IP, MQTT/gaming, WebSocket (no content routing).

### GWLB — Layer 3
- For scaling **third-party virtual appliances** (firewalls, IDS/IPS, DPI).
- Uses **GENEVE protocol** to encapsulate/route traffic through the appliance fleet (bump-in-the-wire).

### CLB — Legacy
- Layer 4+7, old original. **Avoid in new designs**; no path routing, WebSocket, Lambda, or WAF. Almost always the wrong answer unless legacy compatibility is stated.

## Core components

- **Listener**: checks for requests on a protocol+port (e.g. HTTP:80). Multiple listeners per LB.
- **Listener Rules (ALB)**: conditions (path/host/header/query/source IP) route to target groups. Evaluated by priority, default rule catches the rest.
- **Target Group**: logical set of targets (EC2, IPs, Lambda, ports) for one listener, monitored by health checks.
- **Health Checks**: hit a path/port, need success code within threshold(healthy default 5, unhealthy 2), interval, timeout, success codes.
- **Cross-Zone**: enabled=spread across all targets in all AZs; disabled=within each AZ. ALB on by default; NLB off + charges.

## Sticky sessions

- Round-robin by default. **Sticky** routes one client to the same target via a **cookie** (`AWSALB`).
- **Best practice**: externalize session state (ElastiCache/DynamoDB) and avoid sticky sessions.

## SSL/TLS termination

LB decrypts HTTPS, forwards plain HTTP to targets (offloads TLS from servers). Certificates in **ACM** auto-renew. Use **backend SSL** to re-encrypt LP→target traffic for compliance.

## Connection draining / deregistration delay

Stop new connections to a deregistered/unhealthy target but let existing finish (default 300s). Short requests → reduce it; long-lived (uploads/WebSocket) → increase it.

## ALB vs NLB

| | ALB | NLB |
|---|---|---|
| Content routing | Yes | No |
| WebSocket | Yes | Yes (no content routing) |
| Static IP | No | Yes |
| Preserve source IP | No | Yes |
| Extreme throughput | Moderate | Yes |
| TCP/UDP | No | Yes |
| gRPC | Yes | No |
| Lambda target | Yes | No |
| WAF | Yes | No |

## Exam domains

- [ ] Secure (30%)
- [x] **Resilient (26%)** — multi-AZ, health checks, cross-zone, connection draining
- [x] **High-Performing (24%)** — choosing ALB/NLB/GWLB, routing algorithms
- [ ] Cost-Optimized (20%)

## Key gotchas

1. **ALB stateful, NLB can be stateless** (raw TCP forwarding, faster, less features)
2. **NLB preserves source IP; ALB does not** (use `X-Forwarded-For` on ALB)
3. Cross-zone **free on ALB, costs on NLB**
4. Health checks aren't free — each check = an HTTP request
5. **CLB has no path-based routing** → use ALB
6. ALB rules have priority; default rule lowest
7. Tune deregistration delay to match connection length
8. **NLB doesn't support WAF** — WAF → ALB
9. **ALB can route to Lambda** (serverless migration)
10. **"GENEVE" / third-party appliance → GWLB**


## Related services

- **EC2** — primary LB targets
- **ASG** — manages target pool, auto-registers/deregisters
- **VPC** — LBs deployed in VPC subnets (ALB/NLB need ≥2 AZs)
- **ACM** — TLS certificates for HTTPS listeners
- **WAF** — integrates with ALB
- **Route53** — DNS to load balancer
- **CloudWatch** — LB metrics for monitoring/scaling
