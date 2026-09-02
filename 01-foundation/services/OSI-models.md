# OSI Model — The Seven Layers Mapped to AWS Services

## What it is

The OSI (Open Systems Interconnection) model is a conceptual framework that standardizes how network communication works by breaking it into seven abstraction layers. Each layer has a distinct role, and understanding the model is essential for the exam because AWS load balancers, firewalls, and networking services map directly to specific layers.

The exam does not ask you to memorize the OSI model in isolation — it asks you to **choose the right AWS service** based on which layer the problem operates at.

---

## The Seven Layers

### Layer 7 — Application

**Core function:** Direct interaction with the user or application. Processes requests at the content level — understands URLs, HTTP headers, cookies, query parameters, and application-specific protocols.

**Protocols:** HTTP, HTTPS, FTP, SMTP, DNS, SSH, WebSocket

**Key concept:** this is where the application "speaks" — the browser, the API client, the mobile app. Data at this layer is fully formatted and meaningful to the application.

**AWS services:**
- **Application Load Balancer (ALB)** — routes HTTP/HTTPS traffic based on path, host, headers, or query strings. Understands the application's content to make routing decisions.
- **API Gateway** — fully managed Layer 7 service for creating, publishing, and securing APIs. Handles REST, HTTP, and WebSocket APIs.
- **CloudFront** — CDN that caches and delivers content at the edge. Operates at Layer 7 for HTTP/HTTPS content.

**Exam scenarios:** "route traffic based on URL path" → ALB. "create a managed API" → API Gateway. "cache static content at the edge" → CloudFront.

### Layer 6 — Presentation

**Core function:** Formats, translates, and encrypts/decrypts data. Ensures that data from the application layer of one system can be read by the application layer of another — handling differences in data representation (character encoding, image formats, compression).

**Protocols:** SSL/TLS, JPEG, PNG, JSON, XML, ASCII, EBCDIC

**Key concept:** this layer is responsible for encryption/decryption in the OSI model. When you visit an HTTPS website, the TLS handshake and decryption happen conceptually at this layer.

**AWS services:**
- **AWS Certificate Manager (ACM)** — provisions and manages SSL/TLS certificates used for encryption at this layer.
- **SSL/TLS termination on ELB** — the load balancer terminates the encrypted connection (decrypts) and forwards plain HTTP to targets, performing the Presentation layer function.
- **Data compression and format conversion** — CloudFront can compress responses (gzip, Brotli) before delivery.

**Exam scenarios:** "SSL/TLS certificates" → ACM. "encrypt traffic between client and load balancer" → HTTPS listener with ACM certificate.

### Layer 5 — Session

**Core function:** Opens, manages, and closes sessions (connections) between two parties. Maintains connection state, handles session tokens, and determines whether a connection is new, resumed, or terminated.

**Protocols:** NetBIOS, RPC, PPTP

**Key concept:** this layer tracks whether a user is "logged in" or a session is active. It is responsible for session persistence and reconnection.

**AWS services:**
- **Sticky sessions on ELB** — the load balancer uses a session cookie to maintain the client's session on the same target, a Layer 5 function.
- **API Gateway usage plans and API keys** — manage client sessions and rate limiting.
- **Cognito** — manages user sessions (authentication state, token refresh, session tokens).

**Exam scenarios:** "users must stay connected to the same backend for the duration of their session" → sticky sessions on ELB. "manage user authentication sessions" → Cognito.

### Layer 4 — Transport

**Core function:** Reliable (TCP) or fast unguaranteed (UDP) data transfer. Segments data, manages ports, and ensures end-to-end delivery. Does not understand HTTP content — it only sees TCP/UDP packets.

**Protocols:** TCP, UDP

**Key concept:** this layer is about moving data reliably (TCP) or quickly (UDP) between two endpoints. It does not look at the content of the data — only source IP, destination IP, source port, and destination port.

**AWS services:**
- **Network Load Balancer (NLB)** — forwards raw TCP/UDP packets without inspecting HTTP content. Ultra-low latency, millions of RPS.
- **Security Groups** — operate partially at Layer 4. They filter traffic based on IP address, protocol (TCP/UDP), and port number — which is Transport layer information.
- **Elastic Load Balancing (all types)** — the transport layer is where the actual TCP connection termination happens.

**Exam scenarios:** "ultra-low latency, millions of RPS, TCP/UDP traffic" → NLB. "firewall rules based on port and protocol" → Security Groups.

### Layer 3 — Network

**Core function:** Routing between networks based on IP addresses. Determines the best path for data to travel from source to destination across network boundaries.

**Protocols:** IP, ICMP, IPsec

**Key concept:** this is where routing decisions happen. Routers operate at this layer — they examine IP headers and forward packets toward their destination. NAT (Network Address Translation) also happens here.

**AWS services:**
- **Gateway Load Balancer (GWLB)** — forwards IP packets to virtual appliances for inspection.
- **VPC Route Tables** — route traffic between subnets, to the internet, or to on-premises networks via VPN/Direct Connect based on IP destination.
- **NAT Gateway** — performs Network Address Translation, allowing private instances to reach the internet. A Layer 3 function.
- **Internet Gateway** — enables VPC-to-internet connectivity. Operates at Layer 3.
- **VPC Peering and Transit Gateway** — route traffic between VPCs based on IP CIDR ranges.

**Exam scenarios:** "route traffic between VPCs" → VPC Peering or Transit Gateway. "allow private instances to access the internet" → NAT Gateway. "inspect IP packets with a firewall appliance" → GWLB.

### Layer 2 — Data Link

**Core function:** Transfers data within the same local network using MAC addresses. Handles framing, error detection, and physical addressing.

**Protocols:** Ethernet, Wi-Fi (802.11), ARP, PPP

**Key concept:** this layer operates within a single network segment (e.g. one subnet or VLAN). Switches operate at this layer — they forward frames based on MAC addresses.

**AWS services:**
- **Elastic Network Interface (ENI)** — a virtual network card with its own MAC address, attached to an EC2 instance within a VPC. The primary ENI is created at launch; additional ENIs can be attached for multi-homed configurations.
- **VPC Subnets** — represent a Layer 2 broadcast domain. All instances in a subnet can communicate at Layer 2 without routing.
- **AWS Virtual Private Cloud** — at the lowest level, VPCs are implemented using virtual switches (VSwitches) that operate at Layer 2.

**Exam scenarios:** "instance needs multiple network interfaces" → ENI. "two instances in the same subnet can communicate directly" → Layer 2.

### Layer 1 — Physical

**Core function:** The actual physical transmission of data — cables, electrical signals, fiber optics, radio waves, network hardware.

**Protocols/medium:** Cat5/Cat6 cables, fiber optics, Wi-Fi, Bluetooth, hubs

**Key concept:** this is the hardware layer — the physical infrastructure that makes all higher-layer communication possible.

**AWS services:**
- **AWS data centers** — the physical buildings, power, cooling, and networking hardware.
- **AWS Direct Connect** — a dedicated physical fiber connection from your on-premises data center to AWS.
- **Amazon Global Accelerator** — uses the AWS global network (private fiber backbone) instead of the public internet to route traffic to the nearest edge location. While Global Accelerator operates at a higher layer conceptually, its value proposition is about bypassing public internet congestion on the physical layer.

**Exam scenarios:** "dedicated physical connection to AWS" → Direct Connect. "use AWS private fiber backbone instead of public internet" → Global Accelerator.

---

## OSI Layer ↔ AWS Service Mapping (Quick Reference)

| Layer | Name | AWS Services | When to choose |
|---|---|---|---|
| 7 | Application | ALB, API Gateway, CloudFront | Need HTTP-aware routing, API management, or content caching |
| 6 | Presentation | ACM, SSL/TLS termination on ELB | Need encryption/decryption, certificate management |
| 5 | Session | Sticky sessions, Cognito | Need session persistence or user authentication sessions |
| 4 | Transport | NLB, Security Groups | Need TCP/UDP forwarding, port-based firewall, extreme performance |
| 3 | Network | GWLB, Route Tables, NAT Gateway, IGW, Transit Gateway | Need IP-level routing, NAT, or virtual appliance inspection |
| 2 | Data Link | ENI, Subnets, VSwitches | Need multiple network interfaces or same-subnet communication |
| 1 | Physical | Data centers, Direct Connect | Need physical connectivity to AWS |

---

## Architecture deep dive

### How layers work together in a typical web request

When a user visits `https://myapp.example.com`:

1. **Layer 1 (Physical):** data travels over physical infrastructure — fiber optics, Ethernet cables, Wi-Fi.
2. **Layer 2 (Data Link):** the request is framed in Ethernet and sent to the local gateway.
3. **Layer 3 (Network):** the IP packet is routed across networks (your LAN → ISP → internet → AWS edge).
4. **Layer 4 (Transport):** a TCP connection is established between the client and the load balancer. TLS handshake happens here (or at Layer 6 conceptually).
5. **Layer 5 (Session):** a session is established. The load balancer sets a session cookie for sticky sessions if configured.
6. **Layer 6 (Presentation):** the encrypted HTTPS data is decrypted by the ALB using the ACM certificate.
7. **Layer 7 (Application):** the ALB inspects the HTTP request, evaluates listener rules (path, host), and routes to the appropriate target group.

The response follows the reverse path back through all seven layers.

### Why this matters for the exam

The exam presents scenarios like:
- "A company needs to route traffic based on URL path" → you need a **Layer 7** service → ALB.
- "An application requires millions of TCP connections per second with static IPs" → you need a **Layer 4** service → NLB.
- "Traffic must be inspected by third-party firewalls" → you need a **Layer 3** service → GWLB.
- "Users must be routed to the closest AWS region over AWS private fiber" → **Global Accelerator**.
- "SSL/TLS certificates must be managed centrally" → **ACM** (Layer 6).

The key skill is matching the **problem's layer** to the **right AWS service**.

### Encapsulation

As data moves down the layers (from application to physical), each layer adds its own header (and sometimes trailer) to the data. This is called **encapsulation**:

- Layer 7 data → adds Layer 6 header → adds Layer 5 header → adds Layer 4 header (TCP/UDP) → adds Layer 3 header (IP) → adds Layer 2 header (Ethernet) → transmitted as Layer 1 signals.

At the receiving end, each layer strips its header and passes the payload up — **de-encapsulation**.

This is not tested directly on the exam, but understanding it explains why NLB (Layer 4) cannot do path-based routing (Layer 7) — it simply does not have access to the HTTP headers.

---

## Exam domain(s)

- [x] **Design Secure Architectures (30%)** — Security Groups (L4), NACLs (L3/L4), SSL termination (L6), WAF (L7)
- [x] **Design Resilient Architectures (26%)** — load balancer selection, routing across AZs
- [x] **Design High-Performing Architectures (24%)** — choosing the right layer for the right performance characteristics

---

## Advanced gotchas & edge cases

1. **ALB = Layer 7, NLB = Layer 4.** This is the #1 mapping to remember. ALB understands HTTP; NLB does not. NLB is faster because it does less inspection.

2. **Security Groups operate at Layer 4.** They filter by IP address, protocol, and port — which are all Transport layer attributes. They do not inspect HTTP content (that's WAF/Layer 7).

3. **NACLs operate at a lower level than Security Groups.** NACLs are stateless and evaluate rules in order (by number). They can explicitly deny traffic — Security Groups cannot.

4. **GWLB uses Layer 3 (GENEVE).** It encapsulates IP packets and sends them to a virtual appliance. The appliance inspects the packet and returns it. This is transparent to the client.

5. **Sticky sessions are a Layer 5 concept.** They maintain session affinity by tracking connection state, not by inspecting content (Layer 7) or forwarding packets (Layer 4).

6. **SSL/TLS is conceptually Layer 6.** On the exam, if a question asks about "encryption at the presentation layer," the answer involves ACM or SSL termination on ELB.

7. **Direct Connect is Layer 1.** It is a physical fiber connection — not a VPN (which is Layer 3). Direct Connect provides dedicated, private physical connectivity to AWS.

8. **Global Accelerator uses AWS's private backbone (Layer 1/3).** It routes traffic over AWS's fiber network instead of the public internet, reducing latency and improving reliability.

---

## Exam-style questions

**Q1.** A Solutions Architect needs to route traffic based on URL path (`/api/*` to backend A, `/web/*` to backend B). Which layer must the load balancer operate at?
- A) Layer 4
- B) Layer 7
- C) Layer 3
- D) Layer 2

<details><summary>Answer</summary>
**B** — path-based routing requires understanding HTTP content (URLs), which is Layer 7 functionality. Layer 4 (NLB) only sees TCP/UDP packets and cannot inspect HTTP paths. The answer is Layer 7 → ALB.
</details>

**Q2.** A company deploys a third-party firewall appliance in their VPC and needs all inbound and outbound traffic to pass through it for inspection. Which load balancer type operates at the appropriate OSI layer for this use case?
- A) Application Load Balancer
- B) Network Load Balancer
- C) Gateway Load Balancer
- D) Classic Load Balancer

<details><summary>Answer</summary>
**C** — GWLB operates at Layer 3 (Network) and uses the GENEVE protocol to encapsulate IP packets and route them through virtual appliances transparently. ALB (L7) and NLB (L4) cannot integrate with third-party virtual appliances in this manner.
</details>

**Q3.** An engineer wants to filter traffic at the instance level using rules that allow or deny traffic based on source IP and port number. Which OSI layer do these rules operate at?
- A) Layer 2
- B) Layer 3
- C) Layer 4
- D) Layer 7

<details><summary>Answer</summary>
**C** — Security Groups filter based on IP address, protocol (TCP/UDP), and port number — all Layer 4 (Transport) attributes. While IP addresses are Layer 3, the combination of IP + port + protocol is a Layer 4 filter.
</details>

**Q4.** A user visits a website over HTTPS. At which OSI layer is the TLS handshake performed?
- A) Layer 4 — Transport
- B) Layer 5 — Session
- C) Layer 6 — Presentation
- D) Layer 7 — Application

<details><summary>Answer</summary>
**C** — TLS/SSL encryption and decryption is a Presentation layer (Layer 6) function. It formats and encrypts data for secure transmission. While TLS operates over a Layer 4 TCP connection, the encryption itself is conceptually a Layer 6 responsibility.
</details>

**Q5.** Which AWS service provides a dedicated physical fiber connection from an on-premises data center to AWS, operating at Layer 1 of the OSI model?
- A) Site-to-Site VPN
- B) VPC Peering
- C) AWS Direct Connect
- D) AWS Transit Gateway

<details><summary>Answer</summary>
**C** — Direct Connect is a physical (Layer 1) fiber connection. Site-to-Site VPN (A) is a Layer 3 IPsec tunnel over the public internet. VPC Peering and Transit Gateway operate at Layer 3 (IP routing).
</details>

---

## Related services

- [[ALB]] — Application Load Balancer, Layer 7 HTTP routing
- [[NLB]] — Network Load Balancer, Layer 4 TCP/UDP
- [[GWLB]] — Gateway Load Balancer, Layer 3 virtual appliance integration
- [[VPC]] — implements Layers 1-3 for AWS networking
- [[Security-Groups]] — Layer 4 firewall rules
- [[ACM]] — Layer 6 SSL/TLS certificate management
