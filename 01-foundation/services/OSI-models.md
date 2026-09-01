# OSI Model - The Seven Layers Mapped to AWS Services

## Complete Layers Table

| # | Layer (EN) | Core Function | Protocol/Device Examples | Related AWS Services |
|---|---|---|---|---|
| 7 | **Application** | Direct interaction with the user/application; processes requests at the content level (URL, Headers, Cookies) | HTTP, HTTPS, FTP, SMTP, DNS, SSH | **ALB**, API Gateway, CloudFront |
| 6 | **Presentation** | Formats and translates data; handles encryption/decryption and compression | SSL/TLS, JPEG, JSON, XML | ACM (SSL/TLS certificates), SSL Termination |
| 5 | **Session** | Opens, manages, and closes sessions between two parties; maintains connection state | NetBIOS, RPC, PPTP | Sticky Sessions on ELB |
| 4 | **Transport** | Reliable (TCP) or fast unguaranteed (UDP) data transfer; segments data; manages Ports | TCP, UDP | **NLB**, Security Groups (partially operate here) |
| 3 | **Network** | Routing between networks based on IP addresses | IP, ICMP, Routers | **GWLB**, VPC Route Tables, NAT Gateway |
| 2 | **Data Link** | Transfers data within the same local network via MAC addresses | Ethernet, Switches, ARP | ENI (Elastic Network Interface) |
| 1 | **Physical** | Actual physical transmission: cables, electrical signals, fiber optics | Cables, Wi-Fi, Fiber | Physical infrastructure of AWS data centers |