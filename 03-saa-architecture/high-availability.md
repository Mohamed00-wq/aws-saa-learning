# High Availability

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

High availability (HA) minimizes the time your application is down or unreachable. The goal is to keep services available for as close to 100% of the time as possible, measured by availability targets known as the "nines" (99.9%, 99.99%). HA is about designing in **redundancy, failover, and quick recovery** so that a single component failure does not take the whole system down. In the cloud this translates into placing resources across multiple Availability Zones, distributing traffic with load balancers, and automatically replacing failed instances.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - You run more than one copy of every critical component (instances, databases, load balancers) in separate isolated facilities, the Availability Zones.
  - A load balancer (ALB/ELB) or DNS routing (Route 53) spreads traffic among the healthy copies.
  - Health checks constantly test each component. When one fails, traffic is redirected away from it and new capacity is provisioned automatically.
- Why does it work?
  - Redundancy means no single point of failure exists. Even if one AZ, instance, or database node dies, the others continue serving traffic.
  - Automated detection and re-routing (health checks + failover) is fast and removes dependency on humans.
- What is the main idea behind it?
  - Assume components will fail. Design so that failure of any one component is invisible to users. Availability = uptime / total time, and every layer must be redundant for the overall system to be highly available.

## 3. Trade-offs

What do you gain?
- Meaningful uptime even during hardware or AZ failures
- Protected from single point of failure at compute, network, and data layers
- Automatic failover instead of manual recovery, which usually takes hours

What do you sacrifice?
- **Cost**  you pay for duplicate resources (standby instances, multi-AZ databases, extra storage) that sit idle or partially idle
- **Complexity**  more components to design, monitor, and debug (load balancers, health checks, replication)
- **Performance overhead**  replication and synchronization between copies can add latency
- HA does not guarantee zero data loss or zero downtime, a failure still triggers a brief failover window

## 4. AWS Services That Work With This Concept

- **Amazon EC2 Auto Scaling**  keeps the desired number of instances running across AZs and replaces unhealthy ones
- **Elastic Load Balancing (ALB/NLB)**  distributes traffic across healthy targets in multiple AZs
- **Amazon Route 53**  health checks with failover, geolocation, and latency-based routing
- **Amazon RDS Multi-AZ / Aurora**  replicas in a separate AZ with automatic failover to a standby
- **Amazon S3 / DynamoDB**  inherently HA across AZs (S3 is 11 nines of durability)
- **Amazon ECS/EKS**  run containers across AZs with service auto scaling
- **AWS Global Accelerator**  routes traffic to healthy endpoints with fast failover

## 5. When to Use

Use this concept when:
- The application is customer-facing and downtime directly costs money, reputation, or SLA penalties
- You have an availability requirement such as 99.9% or higher (multi-AZ deployment in at least two AZs)
- You want to survive single-AZ or single-instance failures without manual intervention
- You run stateful services (databases) where failover matters, for example RDS Multi-AZ
- Exam questions mention "highly available," "no single point of failure," "fault tolerant at the AZ level," or "survive an AZ outage"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The workload is experimental, batch, dev/test, or has no uptime requirement because HA adds cost without value
- A single region or a few servers is genuinely acceptable for the business need
- Database writes must be strongly consistent and synchronous replication cost is not justified
- You only need the ability to recover after a disaster in another region, which is **disaster recovery**, not HA (HA handles component failures, DR handles whole-site loss)
- The added failover complexity would increase operational risk beyond the benefit of staying up