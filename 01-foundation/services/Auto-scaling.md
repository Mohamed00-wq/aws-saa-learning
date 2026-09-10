# ASG — Auto Scaling Groups

## What it is

Auto Scaling Groups maintain the right number of EC2 instances to match demand. Scale out under load, scale in when demand drops. ASGs make EC2 **self-healing**: failed instances are replaced automatically. Always paired with an **ELB** and **Launch Template**.

## Core components

**Launch Template**: AMI, instance type, key pair, SGs, IAM profile, user data, EBS. Versioned (`$Latest`/`$Default`).

**ASG settings**: min (never below), max (never above), desired (target). Multi-AZ for HA. Auto-registers instances with a **target group**.

## Scaling policies

| Policy | How it works |
|---|---|
| **Target Tracking** | Set target (e.g. CPU 50%), ASG auto-adjusts. Simplest. |
| **Step Scaling** | Add/remove by increments based on CloudWatch alarms |
| **Scheduled Scaling** | Scale at known times (daily spikes) |
| **Predictive Scaling** | ML forecast from ~14 days of history |

## Health checks & termination

- **EC2 checks**: hardware/OS only. **ELB checks**: verifies app serving — drives replacement.
- **Termination policy**: default terminates instance closest to next billing hour (cost-efficient).
- **Cooldown** (default 300s): prevents flapping after scaling. Per-policy, not global.

## Instance Refresh & Lifecycle hooks

- **Instance Refresh**: replaces instances using updated launch template while maintaining min healthy %. Zero-downtime deployments.
- **Lifecycle hooks**: pause before put-in-service or removal — drain connections, run init. Uses SNS/SQS.

## Warm Pools & Mixed Instances

- **Warm Pools**: pre-initialized stopped instances, resume in seconds. Count toward max size.
- **Mixed instances**: On-Demand base + Spot capacity. Allocation: `lowest-price`, `capacity-optimized`, or `diversified`.

## Exam domains

- [ ] Secure (30%)
- [x] **Resilient (26%)** — self-healing, multi-AZ, health checks, lifecycle hooks
- [x] **High-Performing (24%)** — target tracking, predictive scaling, warm pools
- [x] **Cost-Optimized (20%)** — scheduled scaling, mixed instances, warm pools

## Key gotchas

1. ASG replaces instances that **fail health checks**; degraded-but-serving only replaced if ELB marks unhealthy
2. Launch Templates versioned; Launch Configurations **immutable**
3. **Desired bounded by min/max**
4. **Scale-in protection**: mark instances protected from termination
5. **Warm pool instances count toward max size**
6. Predictive scaling needs ~**14 days** history
7. **Health check grace period** (300s) prevents marking new instances unhealthy before init

## Related services

- [[EC2]] — compute the ASG manages
- [[ELB]] — traffic distribution; health check signals
- [[CloudWatch]] — metrics driving scaling policies
- [[SNS]] — lifecycle hook notifications
