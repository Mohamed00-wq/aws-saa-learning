# Elasticity

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

Elasticity is the ability to **automatically add and remove capacity in response to changing demand**. Traffic spikes in, capacity grows to absorb them. Traffic drops, capacity shrinks to avoid waste. The goal is to **pay only for what you need at any moment** while never being caught short during a surge. Elasticity is at the heart of the cloud value proposition, you treat compute and storage as flexible rather than fixed resources.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - A monitor (usually CloudWatch metrics) watches utilization such as CPU, memory, queue depth, or request count.
  - When a metric crosses a threshold toward saturation, more units are added (scale out or up).
  - When utilization drops, units are removed (scale in or down).
- Why does it work?
  - Because cloud resources are ephemeral and API-driven, capacity can be provisioned in minutes. Auto Scaling Groups and serverless services do this repeatedly and automatically.
- What is the main idea behind it?
  - Match exactly the capacity that demand requires at each point in time. Schedule predictable fluctuations (business hours, sales events) and react to unpredictable ones (viral traffic) with metric-based scaling. Elasticity is the combination of **scalability** (the ability to grow) plus the ability to **shrink back**, because growing is only useful if you can also shrink and stop paying for idle capacity.

## 3. Trade-offs

What do you gain?
- Handle traffic spikes without manual provisioning or over-provisioning
- Reduce cost by removing unused capacity when demand falls
- Automate scaling decisions that would be slow and error-prone by hand

What do you sacrifice?
- **Cold start / ramp up time**  a sudden surge can outpace new capacity being ready (minutes for EC2, fractions of a second for Lambda)
- **Monitoring complexity**  you need good metrics, thresholds, and cooldowns tuned so scaling doesn't "thrash" (scale out then immediately scale in)
- **Statefulness is hard**  stateless apps scale beautifully, session-ful apps need external session stores (ElastiCache, DynamoDB)
- Scaling down aggressively can hurt if it goes too far and users return abruptly

## 4. AWS Services That Work With This Concept

- **Amazon EC2 Auto Scaling**  dynamic scaling (target tracking, step) and scheduled scaling for instance fleets
- **AWS Auto Scaling (Application Auto Scaling)**  scaling for ECS, DynamoDB, Aurora, and Spot Fleets
- **Amazon Aurora Serverless / DynamoDB (on-demand)**  database capacity scales with workload automatically
- **AWS Lambda**  inherently elastic, scales instantly to thousands of concurrent executions
- **Elastic Load Balancing**  adds/removes targets from traffic as they scale
- **Amazon SQS**  decouples bursts so producers and consumers can scale independently
- **CloudWatch + EventBridge**  the metric and threshold engine that drives scaling actions

## 5. When to Use

Use this concept when:
- Traffic is variable or spiky, with clear peaks and troughs (web traffic, seasonal promo, batch jobs)
- You want to avoid paying for peak capacity during off-peak hours
- The workload is **stateless** (web tier, workers, API backends) so instances can come and go freely
- Schedules are predictable (your target = scale automatically at 8am or before a flash sale)
- Exam questions mention "scale based on demand," "CloudWatch metric triggers scaling," "handle a spike," or "don't waste money on idle capacity"

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The workload is **stateful** with data on local instances, scaling becomes dangerous without re-design (use external state stores first)
- Demand is flat and predictable at a constant level, fixed capacity may be cheaper and simpler than orchestration
- You cannot tolerate the few minutes it takes to provision new instances for a sudden spike, consider keeping a small buffer or using Lambda/serverless
- Scaling logic would create instability ("thrashing," overscaling cost blowup), better to pin capacity until observability improves
- Regulatory or licensing models charge per instance regardless, so shrinking does not actually save money