# Fault Tolerance

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Fault tolerance keeps the system **fully operational even when a component fails**, with **zero impact to users**. Unlike high availability, which allows a small window of downtime during failover, a fault-tolerant system can mask failures completely. The goal is to design components in such a way that a failure of one part is absorbed by the others, with no degradation of service and often no user-visible interruption at all.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - Every critical function is replicated and actively serving (active-active). When one instance or node dies, requests continue flowing to the remaining healthy copies.
  - For stateful services, data is replicated continuously (synchronous replication) so the surviving node has the same data.
  - Traffic is distributed by load balancers and unhealthy nodes are removed from rotation without any request being dropped.
- Why does it work?
  - Because there is no "single dependency" to fail. The failure is isolated (fault containment) and never reaches the user.
  - The failed node is quietly replaced and restored, and traffic distribution adjusts automatically.
- What is the main idea behind it?
  - The system must be built so that it can experience a failure and continue functioning as if nothing happened. This requires generous redundancy at every layer and automatic detection and rerouting that happens faster than users notice.

## 3. Trade-offs

What do you gain?
- Near-zero or zero downtime even during individual component failures
- No manual intervention is needed to keep the service running
- Strong durability and continuity for stateful workloads (no data loss on node failure)

What do you sacrifice?
- **Significant cost**  most resources are redundant, active, and running at all times
- **Complexity**  replication, consistency, and failover logic are hard to build and test
- **Latency**  synchronous replication across AZs adds write latency
- Fully redundant designs are rarely economical for every application, so FT is usually reserved for mission-critical tiers while other tiers use HA or optimized recovery

## 4. AWS Services That Work With This Concept

- **Amazon Aurora**  storage replicated across AZs with automatic failover and up to 15 read replicas
- **Amazon DynamoDB**  data replicated across three AZs, fully managed, no single point of failure
- **Amazon S3**  designed for 11 nines of durability with redundant copies across multiple AZs
- **Elastic Load Balancing (ALB/NLB)**  removes failed targets and keeps sessions served by healthy ones
- **Amazon EC2 Auto Scaling**  replaces failed instances with healthy ones automatically
- **Amazon ElastiCache (Redis cluster mode)**  replicates across multiple nodes with automatic failover
- **AWS Global Accelerator**  fails over between healthy endpoints at the edge

## 5. When to Use

Use this concept when:
- The service is business-critical and brand or compliance demands no user-visible downtime
- The cost of a moment of downtime is higher than the cost of running fully redundant resources
- Data loss on failure is unacceptable and synchronous, continuous replication is affordable
- Exam questions emphasize "no downtime," "no impact to users," "fully fault tolerant," or "survive a failure with zero disruption"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The workload is not critical, because double or triple redundancy is expensive
- You can tolerate seconds or minutes of failover time, in which case high availability is enough and much cheaper
- Cost optimization is the top priority (fault tolerance is the most expensive availability strategy)
- Data can tolerate small amounts of loss (asynchronous replication with eventual consistency is acceptable)
- The requirement is really about recovering a whole region, which is **disaster recovery** rather than fault tolerance