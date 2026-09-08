# ASG — Auto Scaling Groups

![ASG](/images/icons/arch/Arch_Amazon-EC2-Auto-Scaling_64.svg)

## What it is

Auto Scaling Groups automatically maintain the right number of EC2 instances to match demand. They scale out (launch instances) under load and scale in (terminate instances) when demand drops — ensuring performance without over-provisioning and cost without under-provisioning.

ASGs are the mechanism that makes EC2 **self-healing**: if an instance fails its health check, the ASG terminates it and launches a replacement automatically. This is the foundation of resilient architectures on AWS.

ASGs do not work in isolation — they are almost always paired with an **ELB** (for traffic distribution) and use a **Launch Template** (for instance configuration).

## Core components

**Launch Template** (preferred over legacy Launch Configuration):
- Defines AMI, instance type, key pair, SGs, IAM profile, user data, EBS.
- Versioned — reference a version or `$Latest`/`$Default`. Launch Configurations are immutable.

**ASG settings:**
- **min** — never below this (fails get replaced); **max** — never above; **desired** — target count.
- Example: `min=2, max=10, desired=4`.
- **Multi-AZ**: spread instances across AZs for high availability.
- **Target group**: ASG auto-registers/deregisters instances with a target group.

## Scaling policies

- **Target Tracking** (simplest): set target (e.g. CPU at 50%), ASG auto-adjusts. Built-in: CPU, ALB requests, network in/out, custom metrics.
- **Step Scaling**: add/remove by increments based on multiple CloudWatch alarms (e.g. CPU>70% add 2, >90% add 4).
- **Simple Scaling** (legacy): single action, waits for cooldown before next. Use target tracking instead.
- **Scheduled Scaling**: scale at known times (daily 8 AM spike) — good for predictable patterns.
- **Predictive Scaling**: ML-based forecast from history (needs ~14 days). Complements target tracking.

## Health checks

- **EC2 status checks**: hardware/OS-level only.
- **ELB health checks**: verifies app actually serving — more meaningful; drives replacement.
- **Exam:** if instance is "unhealthy" but app works manually, the health check path/port/threshold is misconfigured.

## Cooldown

Period after a scaling activity before another of the same type triggers (default 300s) — prevents flapping. **Per-policy**, not global.

## Termination policy

Controls which instance is terminated on scale-in. **Default**: terminates the instance closest to the next billing hour (cost-efficient) with preference for oldest launch template. Can be customized (oldest, custom via Lambda).

## Instance Refresh

Gradually replaces instances using an **updated launch template** while maintaining a minimum healthy percentage (e.g. 80%). Enables zero-downtime deployments. Pauseable/resumable.

## Lifecycle hooks

Extend launch/terminate with custom actions:
- **Launching**: pause before putting in service — run init, install, wait for confirmation.
- **Terminating**: pause before removal — drain connections, save state.
- Uses **SNS/SQS** notifications; signal `CompleteLifecycleAction` when done.

## Warm Pools

Set of pre-initialized instances in `Stopped`/`Hibernated` state, moved to `Running` in seconds on demand. Bootstrap cost paid upfront, saved on spikes. **Warm pool instances count toward max size.** For unpredictable spikes (flash sales).

## Mixed instances policy

Mix **On-Demand base** (reliability) + **Spot** (cost for capacity). Allocation: `lowest-price`, `capacity-optimized`, or `diversified`.

## Exam domains

- [ ] Secure (30%)
- [x] **Resilient (26%)** — self-healing, multi-AZ, health checks, lifecycle hooks
- [x] **High-Performing (24%)** — target tracking, predictive scaling, warm pools
- [x] **Cost-Optimized (20%)** — scheduled scaling, mixed instances, warm pools, termination policy

## Key gotchas

1. ASG replaces instances that **fail health checks**; degraded-but-serving instances are only replaced if ELB marks them unhealthy
2. Launch Templates are versioned; Launch Configurations are **immutable**
3. **Desired is bounded by min/max** — forced into range
4. **Scale-in protection**: mark instances protected from termination (stateful workloads)
5. Default termination policy favors **billing efficiency**
6. **Warm pool instances count toward max size**
7. Predictive scaling needs ~**14 days** of history
8. **Cooldown is per-policy**, not global
9. **Health check grace period** (default 300s) prevents marking new instances unhealthy before init


## Related services

- [[EC2]] — compute the ASG manages
- [[ELB]] — traffic distribution; health check signals
- [[CloudWatch]] — metrics driving scaling policies
- [[Launch-Template]] — instance blueprint
- [[SNS]] — lifecycle hook notifications
- [[EC2-Pricing-Models]] — mix On-Demand, Spot, Reserved
