# ASG — Auto Scaling Groups

## What it is

Auto Scaling Groups automatically maintain the right number of EC2 instances to match demand. They scale out (launch instances) under load and scale in (terminate instances) when demand drops — ensuring performance without over-provisioning and cost without under-provisioning.

ASGs are the mechanism that makes EC2 **self-healing**: if an instance fails its health check, the ASG terminates it and launches a replacement automatically. This is the foundation of resilient architectures on AWS.

ASGs do not work in isolation — they are almost always paired with an **ELB** (for traffic distribution) and use a **Launch Template** (for instance configuration).

---

## Key concepts

### Core components

**Launch Template** (preferred over the legacy Launch Configuration):
- Defines what each new instance looks like: AMI, instance type, key pair, security groups, IAM instance profile, user data script, EBS volume configuration.
- Versioned — you can create multiple versions of a template and reference a specific version or `$Latest` or `$Default`.
- Using Launch Templates is required for new features like mixed instances policies, capacity reservations, and placement strategies.

**ASG settings:**
- **Minimum size** — the ASG will never have fewer than this many instances. If an instance fails, a replacement is launched immediately.
- **Maximum size** — the ASG will never exceed this limit.
- **Desired capacity** — the target number of instances. The ASG maintains this by launching/terminating instances as needed.

Example: `min=2, max=10, desired=4` — the ASG starts with 4 instances, can scale up to 10, and will never drop below 2.

**VPC and subnets:**
- The ASG is configured to launch instances across specific subnets (and thus AZs).
- Spreading instances across multiple AZs provides built-in **high availability** — if one AZ goes down, instances in other AZs continue serving traffic.

**Target Group attachment:**
- The ASG automatically registers new instances with an attached ELB target group and deregisters terminated instances.
- This is how traffic starts flowing to new instances and stops flowing to terminated ones — no manual steps needed.

### Scaling Policies

Scaling policies tell the ASG **when** and **how many** instances to add or remove.

**Target Tracking:**
- The simplest and most common policy type.
- You set a target metric (e.g. "keep average CPU utilization at 50%"), and the ASG automatically adjusts the instance count to maintain that target.
- The ASG handles the math — you don't need to define thresholds or step increments.
- Supports built-in metrics: CPU utilization, ALB request count per target, average network in/out, custom CloudWatch metrics.

**Step Scaling:**
- Scale by defined increments based on CloudWatch alarm thresholds.
- Example: if CPU > 70%, add 2 instances. If CPU > 90%, add 4 more. If CPU < 30%, remove 1 instance.
- More granular control than target tracking but requires defining multiple CloudWatch alarms.
- Better for workloads with known scaling behaviors where you want precise control over the response.

**Simple Scaling** (legacy):
- A single scaling action triggered by a CloudWatch alarm.
- After the scaling action completes, a **cooldown period** must expire before another action can trigger.
- Largely superseded by target tracking and step scaling. AWS recommends using target tracking instead.

**Scheduled Scaling:**
- Scale at specific known times (e.g. "add 4 instances every weekday at 8 AM, remove them at 6 PM").
- Use for predictable, time-based patterns (business-hours traffic, batch job windows).
- Supports cron expressions for complex schedules.

**Predictive Scaling:**
- Uses machine learning to analyze historical traffic patterns and forecast future demand.
- Automatically creates scheduled scaling actions based on the forecast.
- Best for workloads with recurring traffic patterns (daily, weekly, seasonal).
- Works alongside other scaling policies — predictive scaling handles the forecast, while target tracking handles real-time adjustments.

### Health Checks

ASG monitors instance health and replaces unhealthy instances automatically:

**EC2 status checks:**
- Basic system status checks (underlying hardware) and instance status checks (OS-level issues).
- If an instance fails a status check for a sustained period, the ASG can replace it.

**ELB health checks:**
- When an ELB target group is attached, ASG uses the ELB's health check result to determine instance health.
- If the ELB marks an instance as unhealthy (e.g. the application on port 80 is not responding), the ASG terminates it and launches a replacement.
- ELB health checks are more meaningful than EC2 status checks because they verify the application is actually serving traffic, not just that the OS is running.

**Exam note:** if a question asks "the application on the instance is responding but the instance is marked unhealthy," the health check path or port may be misconfigured, not that the instance is actually failing.

### Cooldown Period

After a scaling activity completes, the ASG waits for a cooldown period before executing another scaling activity of the same type. This prevents rapid, repeated scaling actions (flapping) that could cause instability.

Default cooldown is 300 seconds (5 minutes). You can customize this per policy. A **group cooldown** applies to the ASG, while a **policy cooldown** applies to a specific scaling policy.

### Termination Policy

Controls **which** instance is terminated first when the ASG needs to scale in (reduce capacity):

- **Default behavior:** terminate the instance that is closest to the next billing hour, with a preference for instances using the oldest launch template.
- **Custom termination policies** can prioritize:
  - Oldest instance first.
  - Closest to next billing hour first.
  - Instance with the oldest launch configuration/template.
  - Custom termination policies using Lambda functions.

**Exam note:** the default termination policy is designed to minimize wasted costs — terminating the instance closest to the next billing hour means you've already paid for most of that hour.

---

## Architecture deep dive

### Instance Refresh

When you update a launch template (e.g. new AMI, updated software), existing instances are not automatically replaced. **Instance Refresh** handles this:
1. You trigger an instance refresh, optionally specifying a minimum healthy percentage (e.g. 80%).
2. The ASG gradually terminates old instances and launches new ones from the updated launch template.
3. It maintains at least the minimum healthy percentage throughout the process.
4. New instances are registered with the target group and begin receiving traffic once healthy.

This enables zero-downtime deployments for EC2-based architectures.

### Lifecycle Hooks

Lifecycle hooks extend the instance launch and termination process with custom actions:
- **Instance launching:** after the instance is launched but before it is put into service, a lifecycle hook pauses the process. You can run custom initialization scripts, install software, or wait for a confirmation before the instance starts receiving traffic.
- **Instance terminating:** before the instance is terminated, a lifecycle hook pauses. You can drain connections, save state, or perform cleanup before the instance is removed.

Lifecycle hooks use **SNS notifications** or **SQS queues** to trigger custom workflows, and you manually signal `CompleteLifecycleAction` when done.

### Warm Pools

A warm pool maintains a set of pre-initialized instances in a `Stopped` or `Hibernated` state, ready to be quickly moved to `Running` when demand increases:
- Instances in a warm pool have already gone through the launch and initialization process (AMI boot, software install, etc.).
- When the ASG needs to scale out, it moves instances from the warm pool to the running ASG — this is much faster than launching new instances from scratch.
- Warm pool instances can be in `Stopped` state (EBS volumes persist, RAM is lost) or `Hibernated` state (RAM saved to EBS, resume in seconds).
- You pay for the instance hours when running and EBS storage for stopped/hibernated instances, but not for instance hours while stopped.

**Exam use case:** workloads with sudden, unpredictable traffic spikes where launch time matters (e.g. e-commerce flash sales, batch processing windows).

### Mixed Instances Policy

An ASG can mix On-Demand and Spot instances within the same group using a mixed instances policy:
- **On-Demand base:** a fixed number of On-Demand instances that form the baseline capacity.
- **Spot allocation:** additional capacity filled by Spot instances from diverse instance types and AZs.
- **Allocation strategy:** `lowest-price` (cheapest pool), `capacity-optimized` (most available pool, lowest interruption rate), or `diversified` (spread across pools).

This pattern provides the cost benefits of Spot while maintaining On-Demand instances for baseline reliability.

### Scaling Policies in Depth

**How target tracking works internally:**
1. You set a target value (e.g. CPU = 50%).
2. The ASG continuously monitors the metric (aggregated across all instances).
3. If the metric exceeds the target, the ASG launches instances to bring it down.
4. If the metric falls below the target, the ASG terminates excess instances.
5. The ASG calculates the exact number of instances needed to return the metric to the target — it is not a simple "add 1, wait, add another" approach.

**Custom metrics:** you can use any CloudWatch metric (e.g. queue depth, request latency, custom application metrics) as the target tracking metric.

---

## Exam domain(s)

- [ ] Design Secure Architectures (30%)
- [x] **Design Resilient Architectures (26%)** — self-healing, multi-AZ distribution, health checks, lifecycle hooks
- [x] **Design High-Performing Architectures (24%)** — target tracking, predictive scaling, warm pools for fast response
- [x] **Design Cost-Optimized Architectures (20%)** — scheduled scaling, mixed instances (Spot + On-Demand), warm pools, termination policies

---

## Advanced gotchas & edge cases

1. **ASG only replaces instances that fail health checks.** If an instance is running but degraded (slow, returning errors), it is only replaced if the ELB health check marks it unhealthy. EC2 status checks only detect hardware/OS failures.

2. **Launch Templates are versioned; Launch Configurations are not.** Launch Configurations are immutable and cannot be modified after creation. Launch Templates support multiple versions, making them more practical for production use.

3. **Desired capacity is bounded by min and max.** If you manually set desired to 15 but max is 10, the ASG caps at 10. If you set desired to 1 but min is 2, the ASG launches up to 2.

4. **Scale-in protection:** individual instances can be marked as "protected from termination." The ASG will not terminate protected instances during scale-in, even if they are the oldest. Use this for instances running stateful workloads.

5. **Default termination policy favors billing efficiency.** The ASG terminates the instance closest to the next billing hour first, which minimizes wasted spend.

6. **Instance refresh can be paused and resumed.** If a deployment goes wrong, you can pause the refresh, fix the issue, and resume.

7. **Warm pool instances count toward max size.** If your ASG max is 10 and you have 5 instances in a warm pool, the ASG can have up to 5 running instances (10 max minus 5 warm). The warm pool size is separate from the ASG max.

8. **Predictive scaling requires historical data.** It needs at least 14 days of historical data to generate accurate forecasts. New workloads may not benefit immediately.

9. **ASG cooldown is per-policy, not global.** Different scaling policies can have different cooldown periods.

10. **ELB health check grace period.** When an instance is launched, the ASG waits for the ELB health check grace period (default 300 seconds) before evaluating ELB health checks. This prevents newly launched instances from being marked unhealthy before the application finishes initializing.

---

## Exam-style questions

**Q1.** An ASG has min=2, max=10, desired=4, and one instance fails its ELB health check. What happens?
- A) The ASG does nothing — 4 instances are still running
- B) The ASG terminates the unhealthy instance and launches a replacement to maintain desired capacity of 4
- C) The ASG scales up to 5 instances
- D) The ASG waits for manual intervention

<details><summary>Answer</summary>
**B** — ASGs maintain desired capacity by replacing unhealthy instances automatically. The unhealthy instance is terminated and a new one is launched to bring the count back to 4. This is the self-healing property of ASGs.
</details>

**Q2.** Which scaling policy type is best for handling a predictable daily traffic spike every morning at 9 AM?
- A) Target tracking
- B) Step scaling
- C) Scheduled scaling
- D) Simple scaling

<details><summary>Answer</summary>
**C** — scheduled scaling lets you define scaling actions at known times, ensuring capacity is ready before the spike starts. Target tracking and step scaling react to metrics after the spike has already begun, which may not provide capacity fast enough.
</details>

**Q3.** A Solutions Architect wants to reduce launch time for instances in an ASG during sudden demand spikes. The instances take 5 minutes to boot and initialize. What should they do?
- A) Use a larger instance type
- B) Enable predictive scaling
- C) Use a warm pool
- D) Increase the cooldown period

<details><summary>Answer</summary>
**C** — warm pools maintain pre-initialized instances that can be moved to Running in seconds, bypassing the 5-minute boot and initialization process. Predictive scaling (B) helps with known patterns but still launches new instances. Larger instances (A) don't reduce initialization time.
</details>

**Q4.** An ASG is configured with min=3, max=6, desired=3. The company updates the launch template with a new AMI. Existing instances continue running the old AMI. What should they do to replace all instances with the new AMI?
- A) Manually terminate all instances
- B) Trigger an Instance Refresh
- C) Update the ASG desired capacity to 0, then back to 3
- D) Delete and recreate the ASG

<details><summary>Answer</summary>
**B** — Instance Refresh gradually replaces existing instances with new ones from the updated launch template while maintaining minimum healthy capacity. It provides zero-downtime deployments. Manual termination (A) is risky. Setting desired to 0 (C) violates the min and causes downtime. Deleting the ASG (D) is destructive.
</details>

**Q5.** A Solutions Architect is designing an ASG for a fault-tolerant web application. They want to minimize cost while maintaining availability. Which configuration is MOST appropriate?
- A) All On-Demand instances
- B) All Spot instances
- C) On-Demand base with Spot instances for additional capacity
- D) All Reserved Instances

<details><summary>Answer</summary>
**C** — mixed instances policy with On-Demand base + Spot for burst provides the best balance. On-Demand ensures baseline reliability. Spot reduces cost for additional capacity. All Spot (B) risks simultaneous interruption. All On-Demand (A) wastes cost savings. RIs (D) are a billing model, not a scaling strategy.
</details>

**Q6.** An engineer notices that instances launched by an ASG are immediately being marked as unhealthy by the ELB and terminated. The application works fine when tested manually. What is the MOST likely cause?
- A) The ASG min size is too low
- B) The ELB health check path or port is misconfigured in the target group
- C) The ASG is launching instances in the wrong AZ
- D) The launch template has an incorrect AMI

<details><summary>Answer</summary>
**B** — if instances pass manual testing but fail ELB health checks, the health check configuration (path, port, success codes, or thresholds) is likely wrong. The health check may be pointing to a wrong port or path, or the threshold may be too aggressive (marking instances unhealthy before the application finishes initializing). The ELB health check grace period can also be increased.
</details>

---

## Related services

- [[EC2]] — the compute service that ASG manages
- [[ELB]] — distributes traffic across ASG instances; provides health check signals
- [[CloudWatch]] — provides metrics (CPU, custom) that drive scaling policies
- [[Launch-Template]] — defines the blueprint for instances launched by the ASG
- [[SNS]] — lifecycle hooks publish notifications to SNS topics
- [[EC2-Pricing-Models]] — ASGs can mix On-Demand, Spot, and Reserved pricing models
