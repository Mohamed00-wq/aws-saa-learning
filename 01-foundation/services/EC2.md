# EC2 — Elastic Compute Cloud

## What it is

EC2 is the foundational compute building block in AWS — it provides resizable virtual servers (instances) in the cloud. If S3 is where data lives at rest and DynamoDB is where structured data lives, EC2 is where you run code that needs full control over the operating system, networking, and software stack.

EC2 is not the answer for everything (Lambda, ECS, Fargate, and Beanstalk are alternatives for specific workloads), but it remains the most flexible compute option and the one most other services integrate with. Nearly every architecture on the exam involves EC2 in some capacity, whether directly or as the target behind an ELB.

---

## Key concepts

### Instances

An **instance** is a running virtual server. It is always launched **from an AMI** — you cannot launch an instance without one. Once launched, the instance is independent of the source AMI: changes made to the running instance do not modify the AMI, and deleting the AMI does not affect instances already running from it.

Instances are identified by an **instance ID** (e.g. `i-0abc1234def56789`), which is unique and immutable for the lifetime of that instance.

### Instance Types

An instance type defines the hardware profile: CPU count and type (Intel, AMD, Graviton/ARM), memory, network bandwidth, and local storage. Instance types are grouped into families by workload:

| Family | Purpose | Examples |
|---|---|---|
| **General Purpose (t, m)** | Balanced compute, memory, network | t3.micro, m5.large, m7g.medium |
| **Compute Optimized (c)** | CPU-intensive workloads | c5.large, c7g.xlarge |
| **Memory Optimized (r, x, z)** | In-memory databases, real-time analytics | r5.large, x1e.16xlarge |
| **Storage Optimized (i, d, h)** | High sequential I/O to local storage | i3.large, d2.8xlarge |
| **Accelerated Computing (p, g, trn, inf)** | GPU/ML workloads | p4d.24xlarge, g5.xlarge, trn1.32xlarge |

Within each family, the **size** (nano, micro, small, medium, large, xlarge, 2xlarge, ...) determines the number of vCPUs and the amount of RAM.

The **t-series** (t3, t3a, t4g) uses a **CPU credit** model: the instance earns credits when idle and spends them when bursting above baseline CPU. If credits run out, CPU performance is throttled to baseline. Unlimited mode allows bursting beyond your credit balance at an extra cost.

### Networking

Every instance gets a **primary private IP** from the subnet's CIDR range. It also gets a **primary Elastic Network Interface (ENI)** bound to that IP. You can attach additional ENIs for multi-homed or management scenarios.

**Public IP assignment:**
- A regular **public IP** is automatically assigned at launch (if the subnet has `Auto-assign Public IP` enabled) and is released when the instance is stopped or terminated. It changes each time you stop/start the instance.
- An **Elastic IP (EIP)** is a static public IP you allocate to your account. It persists across stop/start cycles and can be reassigned to a different instance in the same region. **AWS charges for idle Elastic IPs** — if an EIP is allocated but not associated with a running instance, you pay a per-hour fee. This is a key cost-optimization exam point.

### Security Groups vs NACLs

These are the two layers of network access control for an instance:

**Security Groups (SG):**
- Stateful: if inbound traffic is allowed, the response is automatically allowed outbound, regardless of outbound rules.
- Act as a firewall at the **instance level** (ENI level).
- All rules are evaluated; they are additive (there is no explicit Deny in SGs — all rules are Allow rules; traffic not matching any rule is implicitly denied).
- Can reference other security groups as a source/destination (e.g. "allow port 443 from sg-abc").

**Network ACLs (NACLs):**
- Stateless: inbound and outbound rules are evaluated independently. If you allow inbound port 80, you must also explicitly allow outbound ephemeral ports for the response.
- Act at the **subnet level**.
- Rules are numbered and evaluated in order; the first matching rule wins.
- Support explicit **Deny** rules, which SGs do not.
- Stateless nature makes them harder to manage but useful as a subnet-level safety net.

**On the exam:** Security Groups are almost always the answer for instance-level access control. NACLs appear when the question specifically mentions subnet-level stateless filtering or explicit deny rules at the network boundary.

### Elastic Network Interfaces (ENIs)

An ENI is a virtual network card attached to an instance within a VPC. Key properties:
- Has a primary private IP, optional secondary private IPs, a MAC address, one or more security groups, and optionally a public IP or Elastic IP.
- The **primary ENI** (eth0) cannot be detached from a running instance. It is created automatically at launch and is deleted when the instance is terminated (unless it was pre-existing).
- **Secondary ENIs** can be detached and reattached to another instance in the same AZ — useful for failover scenarios, management interfaces, or dual-homed network configurations.
- ENIs are AZ-scoped: you cannot attach an ENI to an instance in a different AZ.

### Placement Groups

A placement group controls how instances are placed on the underlying hardware:

- **Cluster placement group** — all instances are packed into a single rack within a single AZ. Provides the lowest inter-instance latency and highest network throughput (up to 10 Gbps per instance, or 100 Gbps with enhanced networking). Used for HPC, tightly coupled workloads. **Risk:** if the rack fails, all instances in the group fail simultaneously.
- **Spread placement group** — each instance is placed on a distinct underlying rack. Guarantees that no two instances share the same physical hardware. Maximum of 7 instances per AZ in a spread group. Used for critical instances that must not fail together (e.g. a primary database, a controller node).
- **Partition placement group** — divides the rack floor into logical partitions. Instances within a partition share hardware, but partitions do not. Used for large distributed systems (HDFS, Kafka, Cassandra) where you want to distribute replicas across failure domains.

### Hibernation

When you stop an instance normally, the contents of RAM are lost. **Hibernation** saves the RAM contents to the root EBS volume and powers off the instance. When you start it again, the RAM is restored from disk, and the instance resumes exactly where it left off — all processes, open files, and network connections are intact.

Requirements:
- The root volume must be EBS-backed and encrypted.
- Sufficient space on the root volume to store the RAM snapshot.
- Not supported for all instance types or instance families.

Hibernation is useful for long-running processes that are expensive to restart from scratch, or for saving the in-memory state of an application that doesn't have a checkpoint mechanism.

### User Data

User data is a script or cloud-init configuration that runs **once** at first boot. It is not a configuration management tool — for ongoing configuration, use Systems Manager, Ansible, or a similar tool.

User data is stored unencrypted by default (visible to anyone with access to instance metadata). To protect secrets, use Secrets Manager or SSM Parameter Store and fetch them at boot time via user data.

User data is limited to 16 KB.

---

## Architecture deep dive

### Instance lifecycle

```
pending → running → stopping → stopped → pending → running
                         ↓
                    terminating → terminated
```

- **pending:** AWS is preparing the instance (allocating resources, booting the AMI).
- **running:** the instance is live and billable.
- **stopping:** the instance is being shut down gracefully. EBS-backed instances go through this state.
- **stopped:** the instance is not running. You are not billed for compute (but you are billed for EBS volumes). Data on EBS persists. Data on instance store is lost.
- **terminating:** the instance is being permanently destroyed. Root EBS volume is deleted by default (unless `DeleteOnTermination` is set to false).

**Critical exam detail:** the default behavior for the root volume on termination is **DeleteOnTermination = true**. Non-root EBS volumes have **DeleteOnTermination = false** by default. This is a common exam question.

### IMDS (Instance Metadata Service)

The metadata service at `169.254.169.254` provides instance configuration data, including:
- Instance type, AMI ID, availability zone
- Security groups, VPC ID, subnet ID
- IAM role credentials (via `/latest/meta-data/iam/security-credentials/<role-name>`)
- User data

**IMDSv2** is the recommended version. It requires:
1. A `PUT` request to `/latest/api/token` with the `X-aws-ec2-metadata-token-ttl-seconds` header to get a session token.
2. Subsequent metadata requests include this token in the `X-aws-ec2-metadata-token` header.

IMDSv2 prevents **SSRF-based credential theft**: an attacker who can make HTTP requests from the instance (e.g. via a web app vulnerability) cannot easily retrieve the session token because the `PUT` request requires a specific header that HTTP redirects and server-side request forgery typically do not preserve.

### AMI ↔ EC2 relationship

- **AMI is the template, EC2 instance is the running copy.** An AMI packages OS + software + configuration; an EC2 instance is the live, billable server created from that package.
- **You must select an AMI to launch an instance** — it is a required parameter.
- **The relationship is bidirectional:** launch an instance from an AMI, and create a new AMI from a configured instance.
- **Independence after launch:** modifying the instance never modifies the AMI; deleting the AMI never affects instances already running from it.
- **Region scope:** an AMI is region-locked; instances launched from it are confined to that same region.

### Instance metadata categories

| Category | Contents |
|---|---|
| Identity | AMI ID, instance type, account ID, IAM role |
| Network | Local IPv4, public IPv4, MAC, VPC ID, subnet ID, security groups |
| Placement | Availability zone, region |
| Lifecycle | Instance state, hibernation status |
| User data | Custom script or cloud-init config |
| Block devices | EBS volume IDs, device mappings |

---

## Exam domain(s)

- [x] **Design Secure Architectures (30%)** — security groups, NACLs, IMDSv2, IAM roles via instance profiles
- [x] **Design Resilient Architectures (26%)** — placement groups (spread for HA), lifecycle management, hibernation
- [x] **Design High-Performing Architectures (24%)** — instance type selection, placement groups (cluster for HPC), ENI placement
- [x] **Design Cost-Optimized Architectures (20%)** — pricing models, Elastic IP charges for idle EIPs, t-series credit model

---

## Advanced gotchas & edge cases

1. **Security Groups are stateful; NACLs are stateless.** The exam tests this distinction heavily. If you allow inbound HTTP in a SG, outbound HTTP is automatically allowed — no outbound rule needed. For NACLs, you must explicitly allow both directions.

2. **SGs are additive; NACLs have deny rules.** A security group can only add Allow rules — there is no explicit Deny. A NACL can explicitly deny traffic.

3. **Deleting an AMI does not delete its snapshots.** You must delete snapshots manually to avoid storage charges.

4. **Elastic IP idle charges.** An allocated but unassociated EIP incurs a per-hour charge. This is a common cost-optimization exam trap.

5. **Root volume deletion by default.** When an instance is terminated, its root EBS volume is deleted unless `DeleteOnTermination` is set to false. Non-root volumes are preserved by default.

6. **Instance store data is lost on stop/terminate.** Instance store is physically attached, ephemeral storage. If the instance stops (even due to a hardware failure), the data is gone.

7. **t-series unlimited mode can cost more than expected.** If your workload sustains high CPU for long periods, t-series instances in unlimited mode can exceed the cost of a larger fixed-size instance.

8. **Hibernation requires an encrypted root volume.** If the root volume is not encrypted, hibernation is not available.

9. **Security groups can reference other security groups, not NACLs.** This is a powerful way to manage access (e.g. "allow port 3306 from sg-app" where sg-app is attached to application servers).

10. **IMDSv1 is vulnerable to SSRF.** If a question mentions credential theft via SSRF, the answer is almost always "enable IMDSv2."

---

## Exam-style questions

**Q1.** A Solutions Architect needs to launch 20 identical EC2 instances with the same OS, software, and configuration. What should they do?
- A) Manually launch 20 blank instances and configure each one
- B) Create a custom AMI from a fully configured instance and launch all 20 from it
- C) Launch all 20 from different AMIs
- D) Launch all 20, then configure each via SSH

<details><summary>Answer</summary>
**B** — a custom AMI ensures all instances are identical and launch with software pre-installed, eliminating configuration drift and reducing launch time. Manual approaches (A, D) are slow and error-prone. Different AMIs (C) break consistency.
</details>

**Q2.** An EC2 instance was launched from a public AMI. The engineer installs additional software and makes configuration changes on the running instance. What happens to the original AMI?
- A) The AMI is automatically updated
- B) The AMI remains unchanged; the instance is now independent
- C) The instance loses its connection to the AMI and stops
- D) AWS creates a new AMI version automatically

<details><summary>Answer</summary>
**B** — once launched, the instance is a fully independent copy. Changes to the running instance have no effect on the source AMI.
</details>

**Q3.** An application requires the absolute lowest network latency between EC2 instances for a tightly coupled HPC workload. What placement strategy should the Architect use?
- A) Spread placement group
- B) Partition placement group
- C) Cluster placement group
- D) No placement group — use multiple AZs

<details><summary>Answer</summary>
**C** — cluster placement group packs instances into the same physical rack, providing the lowest latency and highest network throughput. Spread groups (A) are for HA, not performance. Partition groups (B) are for distributed systems like HDFS.
</details>

**Q4.** A developer has an application that takes 45 minutes to initialize. They want the instance to resume quickly after a stop/start cycle without re-running initialization. What should they use?
- A) Instance store for temporary data
- B) Hibernation
- C) A placement group
- D) A larger instance type

<details><summary>Answer</summary>
**B** — hibernation saves the RAM contents to the encrypted root EBS volume and restores them on start, so the application resumes from exactly where it left off without re-initialization.
</details>

**Q5.** An EC2 instance in a public subnet needs to communicate with an on-premises database over a VPN. The instance has a private IP of 10.0.1.50. The engineer assigns an Elastic IP of 52.1.2.3. Which IP address will the instance use to reach the on-premises database?
- A) 52.1.2.3
- B) 10.0.1.50
- C) The instance cannot communicate with on-premises resources
- D) The Elastic IP is only for inbound traffic

<details><summary>Answer</summary>
**B** — Elastic IPs are for public-facing traffic (inbound from the internet). Communication to on-premises resources over VPN uses the instance's private IP. The VPN tunnel routes between the VPC CIDR and the on-premises network.
</details>

**Q6.** A Solutions Architect is designing a web application with instances in a public subnet. The security group allows inbound HTTP (80) and HTTPS (443) from 0.0.0.0/0. The NACL allows inbound HTTP/HTTPS and outbound ephemeral ports (1024–65535). A user reports they cannot reach the web application. What is the MOST likely cause?
- A) The NACL is blocking inbound traffic
- B) The NACL is blocking outbound traffic on ports 80 and 443
- C) The security group is blocking outbound traffic
- D) The NACL is blocking outbound HTTP/HTTPS

<details><summary>Answer</summary>
**B** — NACLs are stateless. The inbound HTTP request is allowed, but the response (which comes back on the ephemeral port range on the client side) is fine. However, if the NACL also blocks outbound ports 80/443, the instance cannot send responses back to the client. In practice, most default NACLs allow all traffic, but the exam tests the stateless nature of NACLs — you must allow the response traffic explicitly.
</details>

---

## Related services

- [[AMI]] — the template from which EC2 instances are launched
- [[EBS]] — persistent block storage attached to EC2 instances
- [[Auto-scaling]] — automatically launches/terminates instances based on demand
- [[ELB]] — distributes traffic across EC2 instances
- [[VPC]] — defines the networking context (subnets, routing) for instances
- [[IAM]] — roles assigned to EC2 via instance profiles
- [[S3]] — commonly used for storing AMIs, user data scripts, and application assets
