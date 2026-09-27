# Scalability

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Scalability is the ability of a system to **handle growing amounts of work** by adding resources. When users, requests, or data grow, the system must keep performing without redesign. The goal is to support growth from a handful of users to millions while keeping response times stable. Scalability is about the capacity to grow, elasticity is about automatically matching capacity to current demand. A system can be scalable without being elastic (grows slowly and manually) and elastic without scaling back.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - Two ways to grow: **vertical scaling (scale up)** adds more power to one machine (bigger instance type, more RAM/CPU). **Horizontal scaling (scale out)** adds more machines (more instances) behind a load balancer.
  - Each new unit carries part of the load, so total throughput grows with the number of units.
- Why does it work?
  - Horizontal scaling works because the load balancer distributes requests across all healthy units, so no single machine becomes the bottleneck. The key requirement is that any unit can serve any request, meaning the app must be **stateless** or state must live in a shared store (database, cache, session store).
- What is the main idea behind it?
  - Design the architecture so capacity is a **multiplier** (add instances) rather than a **ceiling** (single instance limit). Shrink each unit's dependency on itself: no local state, partitioned or shared data, and services that reference each other through stable interfaces.

## 3. Trade-offs

What do you gain?
- Serve more users, more requests, and more data without rewriting the application
- Horizontal scaling gives virtually unlimited headroom (just add units)
- Blue-green, rolling deployments and fault isolation are easier with many small units

What do you sacrifice?
- **Operational complexity**  load balancers, distributed state, and eventual consistency become necessary
- **Vertical scaling has a hard ceiling** (biggest instance type) and is usually more expensive per unit of capacity
- **Distribution overhead**  more machines means more failure modes, networking, and monitoring
- Shrinking your code's assumptions about "one server" takes engineering effort (sessions, uploads, locks, local caches)

## 4. AWS Services That Work With This Concept

- **Elastic Load Balancing (ALB/NLB)**  spreads traffic across many targets for horizontal scaling
- **Amazon EC2 Auto Scaling**  adjusts the number of instances to sustain load
- **Amazon EC2 (larger instance types)**  for vertical scaling of compute, and vertical options exist for RDS/ElastiCache
- **Amazon Aurora**  read replicas scale reads, Aurora Serverless scales, storage grows automatically to 128 TB
- **Amazon DynamoDB**  scales horizontally with partitions and supports on-demand capacity
- **Amazon ElastiCache**  scales read-heavy workloads and shared sessions horizontally
- **Amazon SQS**  buffers requests so consumer fleets scale independently of producer load
- **AWS Lambda**  scales to thousands of concurrent executions with no infrastructure to manage
- **Amazon Redshift**  scales compute and storage independently (RA3)

## 5. When to Use

Use this concept when:
- Growth is expected (more users, regions, or data) and you must keep response times predictable
- You have a stateless or state-externalizable workload (web tier, APIs, workers) that can spread across instances
- Read-heavy workloads can exploit read replicas or caching to scale beyond one database node
- Exam questions mention "elastically scale out/up," "handle growing traffic," "stateless app behind a load balancer," or "read replicas to scale reads"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- Peak and average load are tiny, one instance is plenty and adding orchestration is over-engineering
- The application holds state on local disks or in memory with no easy way to externalize sessions and it was not built for horizontal scaling
- Your database has a single-writer bottleneck and you cannot partition (shard) the data, scaling compute alone will not help
- A simpler path exists such as vertical scaling during a transition period or moving a hot function to a serverless service
- Complexity budget is low for the team that will have to operate the distributed system