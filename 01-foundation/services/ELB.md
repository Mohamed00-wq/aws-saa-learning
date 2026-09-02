# ELB — Elastic Load Balancing

## What it is

Elastic Load Balancing automatically distributes incoming application traffic across multiple targets — EC2 instances, containers, IP addresses, and Lambda functions — in one or more Availability Zones. ELB is the entry point for most web-facing architectures: clients connect to the load balancer's DNS name, and the load balancer routes each request to a healthy backend target.

ELB is the foundation of **high availability** for web applications. Without a load balancer, clients connect directly to a single instance — if that instance fails, the application is down. With ELB, traffic is distributed across multiple instances in multiple AZs, and unhealthy targets are automatically removed from rotation.

---

## Key concepts

### Load Balancer Types

AWS provides four types of load balancers. The exam tests your ability to choose the right one for a given scenario.

#### Application Load Balancer (ALB) — Layer 7 (HTTP/HTTPS)

The most common choice for web applications and microservices.

- Operates at **Layer 7** (Application layer) — it understands HTTP/HTTPS.
- **Content-based routing:** routes requests based on URL path, hostname, HTTP headers, or query parameters. Example: `/api/*` goes to API servers, `/images/*` goes to image servers.
- **WebSocket and HTTP/2 support** natively.
- **SSL/TLS termination** at the load balancer — the ALB decrypts HTTPS traffic and forwards plain HTTP to backend targets, offloading TLS overhead from your application servers.
- **Integration with AWS Certificate Manager (ACM)** for automated certificate provisioning and renewal.
- **WAF integration** — attach an AWS WAF web ACL directly to an ALB for protection against common web exploits.
- **Lambda functions as targets** — ALB can invoke Lambda functions directly based on URL path, enabling serverless backends behind a traditional load balancer.
- **gRPC support** — for microservice architectures using gRPC.
- **Routing algorithms:** round-robin (default) and least-outstanding-requests.

**Exam scenarios for ALB:** microservices, container-based applications (ECS/EKS), content-based routing, WebSocket applications, any HTTP/HTTPS workload.

#### Network Load Balancer (NLB) — Layer 4 (TCP/UDP/TLS)

Designed for extreme performance and low latency.

- Operates at **Layer 4** (Transport layer) — it forwards raw TCP/UDP packets without inspecting the HTTP content.
- **Ultra-low latency** — single-digit millisecond latency, capable of handling millions of requests per second.
- **Static IP addresses** — each NLB node in each AZ gets an Elastic IP address. Clients connect to a specific IP, not a DNS name that rotates (useful for DNS-based integrations, firewalls, or client applications that cache IPs).
- **Preserves source IP address** — the original client IP is preserved in the request headers (or can be passed via the Proxy Protocol). ALB replaces the client IP with the load balancer's IP.
- **Connection draining** — maintains long-lived TCP connections (important for MQTT, gaming, IoT).
- **Cross-zone load balancing** disabled by default (unlike ALB where it is enabled by default). This is because NLB is designed for maximum performance and cross-zone traffic adds latency.

**Exam scenarios for NLB:** extreme performance requirements, non-HTTP protocols (TCP, UDP, TLS), static IP requirement, preserving source IP, WebSocket (if no content-based routing needed), IoT/mqtt, gaming.

#### Gateway Load Balancer (GWLB) — Layer 3 (Network)

Purpose-built for deploying and scaling **third-party virtual appliances** (firewalls, IDS/IPS, deep packet inspection).

- Operates at **Layer 3** (Network layer) — forwards IP packets.
- Uses the **GENEVE protocol** (Generic Network Virtualization Encapsulation) to encapsulate traffic and pass it to the third-party appliance for inspection.
- The GWLB acts as a transparent bump-in-the-wire: traffic flows through the GWLB, is routed to the virtual appliance, gets inspected/modified, and is returned to the GWLB which then forwards it to its destination.
- Manages the fleet of virtual appliances (scaling, health checks, replacement).

**Exam scenarios for GWLB:** "a company needs to deploy a fleet of third-party firewalls in front of their application," "traffic must be inspected by a virtual appliance before reaching the target."

#### Classic Load Balancer (CLB) — Legacy

- Operates at both Layer 4 and Layer 7.
- The original AWS load balancer — **avoid in new designs**.
- Does not support content-based routing, WebSocket, Lambda targets, or WAF integration.
- Still exists for backward compatibility with applications that were built for it.

**Exam note:** if CLB appears as an option, it is almost always wrong unless the question specifically mentions legacy compatibility.

### Core Components

**Listener:**
- A listener checks for incoming client requests on a specific protocol and port (e.g. HTTP on port 80, HTTPS on port 443).
- A load balancer can have multiple listeners on different ports.
- Each listener defines rules that determine how to route requests to target groups.

**Listener Rules (ALB):**
- Rules evaluate requests based on conditions (path, host, header, query string, source IP).
- Each rule routes matching requests to a specific target group.
- Rules are evaluated in priority order (1 = highest priority).
- A default rule catches anything that doesn't match specific rules.

**Target Group:**
- A logical grouping of targets (EC2 instances, IPs, Lambda functions, or container ports).
- Each target group is associated with a single load balancer listener.
- Targets within a group are monitored by health checks.
- Different target groups can have different health check configurations.

**Health Checks:**
- The load balancer periodically sends a request to each target on a configurable path and port.
- A target is marked **healthy** if it responds with a successful HTTP status code (e.g. 200) within the healthy threshold count.
- A target is marked **unhealthy** if it fails the check for the unhealthy threshold count.
- Unhealthy targets are removed from rotation — the load balancer stops sending traffic to them.
- Health check parameters: protocol, port, path, healthy threshold, unhealthy threshold, interval, timeout, success codes.

**Cross-Zone Load Balancing:**
- When enabled, the load balancer distributes traffic evenly across **all registered targets in all enabled AZs**, regardless of which AZ the request arrived at.
- When disabled, traffic is distributed evenly across targets **within each AZ** — if one AZ has more targets than another, it gets proportionally more traffic.

**Exam note:** ALB has cross-zone enabled by default (free). NLB has it disabled by default and it incurs data processing charges when enabled.

### Sticky Sessions (Session Affinity)

- By default, the load balancer uses a **round-robin** algorithm to distribute requests evenly.
- **Sticky sessions** (enabled via a **stickiness policy** on the target group) route all requests from the same client to the same target for the duration of the session.
- Implemented using a **load balancer cookie** — the ALB/NLB sets a cookie (`AWSALB`, `AWSALBAuth`, etc.) on the first request, and subsequent requests with that cookie are routed to the same target.
- Use case: applications that store session state locally (not in a shared store like DynamoDB or ElastiCache). If the application stores session state externally, sticky sessions are not needed.
- **Downside:** sticky sessions reduce the effectiveness of load balancing because traffic is pinned rather than distributed evenly.

**Exam note:** the best practice is to externalize session state (use ElastiCache, DynamoDB, or S3) and avoid sticky sessions altogether.

### SSL/TLS Termination

- The load balancer terminates the SSL/TLS connection: it decrypts the client's HTTPS request and forwards plain HTTP to the backend targets.
- This offloads the computationally expensive TLS handshake from your application servers.
- You upload SSL/TLS certificates to **AWS Certificate Manager (ACM)** and reference them in the listener configuration.
- ACM handles certificate provisioning, renewal, and deployment automatically.
- **SSL/TLS termination at the load balancer** means traffic between the load balancer and backend targets is unencrypted by default. For compliance, you can enable **backend SSL** (the load balancer re-encrypts traffic to targets using a new HTTPS connection).

### Connection Draining / Deregistration Delay

When a target is deregistered from a target group (or becomes unhealthy), the load balancer stops sending **new** connections to it but allows **existing** connections to complete within a configurable timeout (default 300 seconds).

This prevents abruptly dropping in-flight requests when an instance is being removed.

---

## Architecture deep dive

### How traffic flows through an ALB

1. Client resolves the ALB's DNS name (e.g. `my-alb-123456.us-east-1.elb.amazonaws.com`) via Route 53 or another DNS service.
2. DNS resolves to one of the ALB's IP addresses (in the AZ closest to the client, or via anycast).
3. The client establishes a TCP connection to the ALB and performs the TLS handshake (for HTTPS).
4. The ALB decrypts the request and evaluates listener rules against the HTTP headers (path, host, etc.).
5. The ALB selects a healthy target in the target group based on the routing algorithm.
6. The ALB forwards the request to the target, replaces the client IP with its own (unless Proxy Protocol is enabled), and adds `X-Forwarded-For` and `X-Forwarded-Proto` headers.
7. The target processes the request and responds to the ALB.
8. The ALB forwards the response back to the client.

### DNS and the ALB

- The ALB's DNS name resolves to **multiple IP addresses** (one per AZ). This is how AWS provides AZ-level resilience — if one AZ is unhealthy, DNS stops resolving to that AZ's IP.
- **You cannot hardcode ALB IP addresses** — they change over time as AWS adds/removes nodes. Always use the DNS name.
- ALBs are dual-stack by default: the DNS name resolves to both IPv4 and IPv6 addresses.

### ALB vs NLB decision framework

| Requirement | ALB | NLB |
|---|---|---|
| HTTP/HTTPS content-based routing | Yes | No |
| WebSocket | Yes | Yes (but no content routing) |
| Static IP address | No (DNS only) | Yes (Elastic IP per AZ) |
| Preserve source IP | No (replaced) | Yes |
| Extreme throughput (millions of rps) | Moderate | Yes |
| Non-HTTP protocols (TCP, UDP) | No | Yes |
| gRPC | Yes | No |
| Lambda as target | Yes | No |
| WAF integration | Yes | No |
| Cost | Generally cheaper | Generally more expensive |

### Health check deep dive

Health checks are not just "is the instance reachable?" — they verify the **application** is functioning:

- **Protocol:** HTTP, HTTPS, or TCP.
- **Port:** the port the health check hits (can be different from the traffic port).
- **Path:** the URL path to check (e.g. `/health`, `/status`). The health check endpoint should return 200 OK quickly and ideally check downstream dependencies (database, cache).
- **Healthy threshold:** how many consecutive successful checks before marking healthy (default 5).
- **Unhealthy threshold:** how many consecutive failures before marking unhealthy (default 2).
- **Interval:** how often to check (default 30 seconds).
- **Timeout:** how long to wait for a response (default 5 seconds).

**Exam trap:** if the health check timeout is 5 seconds and the interval is 30 seconds, it takes at most `unhealthy threshold × interval = 2 × 30 = 60 seconds` to mark an instance unhealthy. If the application needs time to warm up, increase the healthy threshold or the grace period (in ASG).

---

## Exam domain(s)

- [ ] Design Secure Architectures (30%) — SSL termination, SG rules on ALB, WAF integration
- [x] **Design Resilient Architectures (26%)** — multi-AZ distribution, health checks, cross-zone load balancing, connection draining
- [x] **Design High-Performing Architectures (24%)** — choosing ALB vs NLB vs GWLB based on use case, routing algorithms
- [ ] Design Cost-Optimized Architectures (20%)

---

## Advanced gotchas & edge cases

1. **ALB is stateful; NLB can be stateless.** ALB maintains session state (e.g. for sticky sessions). NLB forwards raw TCP packets — no state. This means NLB is faster but less feature-rich.

2. **NLB preserves source IP; ALB does not.** If the backend application needs the original client IP, NLB passes it through natively. ALB replaces the client IP with the load balancer's IP (use `X-Forwarded-For` header or Proxy Protocol to recover it).

3. **Cross-zone is free on ALB but costs on NLB.** Enable cross-zone on NLB only when necessary.

4. **Health checks are not free.** Each health check is an HTTP request to your target. If you have 100 targets and a 10-second interval, that's 10 health checks per second — significant traffic for a high-target-count deployment.

5. **Classic Load Balancer does not support path-based routing.** If you need `/api/*` → one target group and `/web/*` → another, you must use ALB.

6. **ALB listener rules have a priority.** The default rule has the lowest priority. Specific rules with conditions are evaluated first. If no rule matches, the default rule applies.

7. **Deregistration delay (connection draining) should be tuned.** Default is 300 seconds. If your connections are short-lived (REST APIs), you can reduce it. If they are long-lived (file uploads, WebSockets), increase it.

8. **NLB does not support WAF.** If the question mentions "web application firewall" in front of a load balancer, the answer is ALB, not NLB.

9. **ALB can route to Lambda functions.** This enables serverless backends behind a traditional load balancer — useful for gradually migrating from EC2 to Lambda.

10. **GWLB uses GENEVE protocol.** If a question mentions "traffic must be inspected by a third-party appliance" or "GENEVE," the answer is Gateway Load Balancer.

---

## Exam-style questions

**Q1.** A company needs to route traffic based on URL path: `/api/*` to one set of servers and `/images/*` to another. Which load balancer type should they use?
- A) Network Load Balancer
- B) Application Load Balancer
- C) Gateway Load Balancer
- D) Classic Load Balancer

<details><summary>Answer</summary>
**B** — ALB supports Layer 7 content-based routing using path conditions in listener rules. NLB operates at Layer 4 and cannot inspect HTTP paths. GWLB is for virtual appliances. CLB does not support path-based routing.
</details>

**Q2.** An application requires extremely low latency, needs to handle millions of requests per second, and requires a static IP address. Which load balancer fits best?
- A) Application Load Balancer
- B) Network Load Balancer
- C) Gateway Load Balancer
- D) Classic Load Balancer

<details><summary>Answer</summary>
**B** — NLB is designed for Layer 4 performance with single-digit millisecond latency, millions of RPS, and Elastic IP support per AZ. ALB does not offer static IPs. GWLB is for virtual appliances.
</details>

**Q3.** True or False: Cross-Zone Load Balancing on an NLB distributes traffic evenly across ALL targets in ALL AZs, and it is enabled by default.
- A) True
- B) False — it distributes within each AZ only
- C) False — it distributes evenly across AZs, but is disabled by default
- D) False — cross-zone is not supported on NLB

<details><summary>Answer</summary>
**C** — Cross-zone on NLB does distribute evenly across all targets, but it is **disabled by default** (unlike ALB where it is enabled by default). Enabling it incurs data processing charges on NLB.
</details>

**Q4.** A Solutions Architect is designing a web application where session state is stored in DynamoDB. Users report being logged out intermittently. What is the MOST likely cause?
- A) The ALB health check is failing
- B) Sticky sessions are not enabled and requests are going to different targets
- C) The target group deregistration delay is too short
- D) The ASG is terminating instances too frequently

<details><summary>Answer</summary>
**A** — if session state is in DynamoDB (externalized), sticky sessions are not needed — any target can serve any session. The intermittent logouts are most likely caused by health checks incorrectly marking healthy targets as unhealthy, causing the load balancer to stop sending traffic to them. The health check path, port, or success codes may be misconfigured.
</details>

**Q5.** A company needs to deploy a fleet of third-party virtual firewalls in front of their application. Traffic must be inspected by the firewalls before reaching the application servers. Which load balancer should they use?
- A) Application Load Balancer
- B) Network Load Balancer
- C) Gateway Load Balancer
- D) Classic Load Balancer

<details><summary>Answer</summary>
**C** — GWLB is purpose-built for transparent integration with third-party virtual appliances. It uses the GENEVE protocol to encapsulate traffic and route it through the firewall fleet. ALB and NLB do not support this pattern.
</details>

**Q6.** An engineer configures an ALB with two target groups. Target Group A has healthy instances and is the default target. Target Group B has no registered targets. A client sends a request that matches a rule routing to Target Group B. What happens?
- A) The request is routed to Target Group A as a fallback
- B) The ALB returns a 502 Bad Gateway error
- C) The ALB returns a 503 Service Unavailable error
- D) The ALB queues the request until a target becomes available

<details><summary>Answer</summary>
**C** — when a request is routed to a target group with no healthy targets, the ALB returns a 503 error. There is no automatic fallback to another target group (A). The ALB does not queue requests (D). 502 indicates a bad response from the target, while 503 indicates no available targets.
</details>

---

## Related services

- [[EC2]] — the primary targets behind load balancers
- [[Auto-scaling]] — ASG manages the target pool; instances auto-register/deregister with ELB
- [[VPC]] — load balancers are deployed within VPC subnets (ALB/NLB require at least two AZs)
- [[ACM]] — provides SSL/TLS certificates for HTTPS listeners
- [[WAF]] — integrates with ALB for web application firewall protection
- [[Route53]] — DNS routes client traffic to the load balancer's DNS name
- [[CloudWatch]] — ELB metrics (request count, latency, 5xx errors) for monitoring and scaling
