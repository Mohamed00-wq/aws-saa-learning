# Performance

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Performance is about delivering fast, responsive, and efficient workloads. The goal is to minimize latency (time per request), maximize throughput (requests per second), and use resources efficiently so users perceive the application as snappy. A performance-focused architecture considers every layer from the network edge to the database, because the slowest component (the bottleneck) sets the pace for the whole system.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - Requests are served as close to the user as possible (CDN/edge), static and repeated data is cached, expensive database work is minimized, and hot resources are scaled and tuned.
  - Response time is the sum of each layer's time, network, compute, storage, and data. Reducing any layer reduces the total.
- Why does it work?
  - Caching moves data nearer to the user or avoids recomputation entirely. Content distribution puts static assets at edge locations. Choosing the right instance, buffer, and query plan converts wasted cycles into faster responses.
- What is the main idea behind it?
  - Find and remove bottlenecks. Attack the **latency** contributors in order of impact: network distance and cold starts, then compute efficiency, then database and storage access. Measure everything (CloudWatch, X-Ray) because you can only optimize what you can see.

## 3. Trade-offs

What do you gain?
- Better user experience and engagement, lower bounce rates, higher conversions
- Higher throughput from the same resources (better utilization, lower cost per request)
- Competitive advantage when speed is part of the product

What do you sacrifice?
- **Cost**  caching layers (CloudFront, ElastiCache), faster instance types, and more provisioned IOPS cost money
- **Complexity and consistency**  caches introduce staleness, and distributing state adds invalidation logic
- **Engineering effort**  profiling and tuning continuous loads, benchmarks, and observability
- Micrometer-level optimizations (over-tuning queries, excessive caching) can complicate maintenance for marginal gains

## 4. AWS Services That Work With This Concept

- **Amazon CloudFront**  CDN, serves static content and API responses from the edge, hugely cuts latency
- **Amazon ElastiCache (Redis/Memcached)**  in-memory cache to offload reads from the database
- **Amazon RDS Performance Insights / enhanced monitoring**  find slow queries and database bottlenecks
- **Amazon DynamoDB DAX**  in-memory cache for DynamoDB, single-digit millisecond reads
- **Amazon Aurora**  fast, dataclass-optimized storage, auto-scaling read replicas
- **AWS Global Accelerator**  lowers latency by routing over the AWS backbone and keeping client IPs fixed
- **AWS Lambda (provisioned concurrency)**  eliminates cold start latency for serverless functions
- **Amazon EBS (gp3, io2) / Elastic Network Adapter (ENA)**  higher I/O and network throughput
- **AWS X-Ray + CloudWatch**  distributed tracing and metrics to find the actual bottleneck
- **Amazon S3 Transfer Acceleration**  faster uploads over long distances

## 5. When to Use

Use this concept when:
- User experience, conversion, or real-time interactions depend on low latency
- You serve a global audience and want edge-delivered static assets and responses
- The workload has a clear bottleneck (database reads, cold starts, WAN latency) that caching or routing can address
- Exam questions mention "low latency for users," "cache to reduce database load," "CDN," "performance optimization," or "users around the world"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- Users are co-located in one data center and latency is already negligible
- Data changes so often that caching introduces more invalidation complexity than it saves (heavy write workloads with mostly fresh data)
- The budget cannot justify premium tiers, provisioned concurrency is a real, recurring cost
- Adding faster hardware masks a design problem (chatty API calls, N+1 queries) that should be fixed in code instead
- The performance requirement is actually a scalability need, growing capacity rather than speeding up a single request