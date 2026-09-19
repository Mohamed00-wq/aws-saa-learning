# EC2 Auto Scaling Course — ASG

## 1. Purpose

EC2 Auto Scaling keeps the **right number of instances** running for your workload — automatically adding (scale out) and removing (scale in) instances as demand changes. Combined with a **load balancer**, it's how you build **elastic, highly available, cost-efficient** EC2 fleets (auto-heals failed instances, spans Availability Zones).

**Core concept = Auto Scaling Group (ASG)**: holds your desired capacity, min/max, launch template, and scaling policies.

## 2. How it works

- An ASG uses a **Launch Template** (AMI, instance type, SG, key, IAM role) to launch instances
- Maintains **Desired / Min / Max** capacity; `desired` is what the group runs at
- **Scaling triggers**:
  - **Manual** — you set desired capacity
  - **Dynamic** — policies react to CloudWatch metrics
  - **Scheduled** — time-based (e.g. 8am weekdays, sales events)
- **Health checks** — EC2 status checks + (optionally) target group / ELB health; unhealthy → replace (terminates + launches)
- **Multi-AZ**: ASG balances instances across the AZs you specify for HA
- Scale-out instance flow: `Pending → (hooks) → InService`; scale-in: `Terminating → Terminated` (load balancer deregisters first, draining connections)

```
Metric (e.g. avg CPU > 50%) → CloudWatch alarm → ASG scales out → new instance (Launch Template) 
→ healthy → attached to ALB target group → receives traffic
```

## 3. When to use

- Any **EC2 fleet** that should survive instance failure (auto-replacement)
- **Variable/unpredictable load** — web apps, APIs, batch, spiky traffic
- Need **HA across AZs** — spread instances so one AZ doesn't take you down
- **Cost control** — scale in on idle → no paying for unused servers; mix in Spot instances
- Apps with **long boot times** — combine with **Warm Pools** (pre-initialized instances)
- Often pairs with: **ALB** (routes to group), **CloudWatch** (metrics), **Spot** (capacity)

## 4. When NOT to use

- **Single instance** that just needs replacing on failure — consider EC2 + recovery (but ASG is still the standard answer for one+ instances)
- **Non-variable, always-full** workloads — a fixed fleet may be simpler (still use ASG minimum)
- **Serverless** (Lambda) or container platforms manage their own scaling (ECS/Fargate, EKS Karpenter)
- Unmanaged, long-running batch with known capacity — Batch/ECS handles it
- When configure-burst spins up instances faster than they become useful — tuning (warm-up, cooldown) is needed instead

## 5. Important features

- **Scaling policy types**:
  - **Target tracking** — like a thermostat: keep metric at target (CPU %, ALB request count/target, SQS backlog). Most common, AWS creates/manages the alarms
  - **Step scaling** — add/remove a specific number of instances based on alarm breach size
  - **Simple scaling** — obsolete-ish, single adjustment + cooldown
- **Scheduled scaling** — predictable load at set times
- **Suspended scaling / cooldowns & instance warm-up** — avoid over-scaling; scrub metric bumps
- **Mixed instances policy** — combine On-Demand + Spot (e.g. 70/30), several instance types for capacity
- **Lifecycle hooks** — pause during launch/termination for custom steps (config, drain, backup) via EventBridge/SNS/Lambda
- **Warm Pools** — pre-init instances ready in seconds; supports **instance reuse** on scale-in
- **Standby state** — for maintenance without terminating
- **Termination policies** — which instances to kill first (oldest, AZ balance)
- **ELB integration** — auto-register/deregister targets, connection draining
- **Instance Refresh** — rolling updates to new Launch Template/AMI with zero downtime (immutable-style deploys)

## 6. Limitations

- **Not instant** — launch takes minutes (AMI + user data + warm-up); cold-scale lags behind spikes
- **Scaling policies are reactive** — they respond to load *after* it changes
- **Metrics granularity** — default CloudWatch EC2 metrics at 5-min; 1-min needs detailed monitoring ($)
- **Capacity squeeze** — hitting account limits / Spot capacity can block scale-out
- **No protection against application-level bugs** — it scales what you asked it to run
- **Cooldowns/warm-up misconfig** → flapping or overscaling
- **Stateful apps** — instances are disposable; don't store state on them (use EFS, EBS+snapshot logic, or external stores; note instance store data lost on scale-in)

## 7. Trade-offs

- **Reactive vs scheduled** — dynamic handles the unpredictable but lags; scheduled handles known peaks ahead of time. Often used together
- **Scale out (more instances) vs scale up (bigger instance)** — ASG scales out = HA + elasticity; vertical scaling = single point of failure
- **On-Demand vs Spot (mixed policy)** — reliability vs up-to-90% cost savings (Spot reclaimable)
- **Warm Pools vs no warm pool** — start fast ($$$) vs slower cold start (cheaper)
- **Speed vs cost** — aggressive cooldowns/warmup → fast response but wasteful; conservative → cheap but slower
- **Auto-scaling vs fixed minimum** — ASG flexibility vs simpler static sizing

## 8. Architecture

Reference elastic web-tier pattern:

```
CloudWatch (EC2 avg CPU) ─ targets ~50% ─ ASG(min 2 / desired 2 / max 10, AZ-A+B)
   Launch Template:
     Golden AMI + User Data
     Mixed instances: 70% on-demand + 30% spot
     IAM instance role, Security Group (only ALB allowed)
ALB → Target Group health checks → ASG registers healthy instances
Lifecycle hook → Lambda configures instances before InService
Warm pool (2) → fast scale for long-boot app
Instance Refresh on new AMI → rolling replace without downtime
```

## 9. SAA-C03 Perspective

Think **ASG + ELB as one unit** — the exam treats them together:

- **Target tracking** on CPU/ALB-request/SQS-backlog-per-instance is the go-to; "least operational overhead" answers pick it
- **SNS/SQS-backed scaling** — scale workers on **queue backlog per instance** (classic question!) 
- **Min/Desired/Max** and **AZ distribution** for HA answers
- **Spot instances in mixed policy** — cost-optimization domain favorite
- **Warm-up time / cooldown / warm pool** — appears in tricky scaling-behavior questions (prevent overscaling)
- **Lifecycle hooks** — run custom setup/drain via Lambda/SSM
- **Instance Refresh** — immutable/golden-AMI rolling deployments
- Health-check-driven self-healing: ASG replaces unhealthy instances automatically
- **Scaling ≠ ELB** — ASG adds capacity; ELB distributes traffic to it

Exam trap: "slow / spiky long-boot app" → **Warm Pool** or **scheduled scaling**, not plain target tracking. "Cost optimization on capacity" → **mixed On-Demand + Spot**. "Golden AMI rollout" → **Instance Refresh**.