# ECS Course — Elastic Container Service

## 1. Purpose

ECS is AWS's **managed container orchestration service** — the AWS-native way to run Docker containers without managing Kubernetes. You define **tasks** (containers + resources + IAM + networking) and **services** (long-running task groups), and ECS schedules, runs, scales, load-balances, and restarts them. It's the default container answer on AWS when you don't need K8s portability: deep AWS integration, free control plane, simple operations.

## 2. How it works

- **Task definition** — the blueprint: image (ECR/Docker Hub), CPU/memory, ports, IAM **task role**, network mode, volumes, env/secrets
- **Cluster** — logical grouping of capacity; tasks/services run inside
- **Task** — one running instance of a task definition (short-lived or part of a service)
- **Service** — maintains desired count of tasks, integrates with **ALB/NLB/Gateway Load Balancer**, auto-heals, rolls updates
- **Compute options (capacity providers)**:
  | Capacity | Model | Notes |
  |---|---|---|
  | **Fargate** | **Serverless** — no servers, per-task billing | `awsvpc` networking (1 ENI per task), spread across AZs |
  | **Fargate Spot** | Serverless spare capacity, discounted | 2-min interruption warning; tolerant workloads only |
  | **EC2 (Auto Scaling group CP)** | You manage/scale EC2 instances | For big/gpu/stateful/steady workloads |
  | **ECS Managed Instances** | **AWS-managed EC2** (Fargate simplicity + EC2 flexibility) | AWS patches/scales; GPU/accel compute support |
  | **External** | on-prem via **ECS Anywhere** | register external instances |
- Recommended: use **capacity providers** (Fargate / ASG / Managed Instances) rather than launch types; a service can use a **strategy** mixing weighted providers
- **Scheduling** — cluster picks capacity; **Service Auto Scaling** (target tracking) scales task count on CPU/memory/request count
- **Networking** — `awsvpc` (per-task ENI + own SG) is standard on Fargate; **awsvpc/bridge/host** (and others) on EC2
- **Storage** — EBS volumes on EC2 tasks, bind mounts; **EFS** shared across Fargate/EC2 tasks; secrets via Secrets Manager/SSM
- Integrations: ALB/NLB/GWLB, CloudWatch (Container Insights), EventBridge (task state changes), IAM, Secrets Manager, Auto Scaling

```
Task definition (image, CPU, memory, IAM role, ports)
   → ECS cluster → capacity provider (Fargate / EC2 ASG / Managed Instances / on-prem)
   → Service (desired count, LB target groups, scaling policy)
   → ALB/NLB → traffic across tasks across AZs
   → CloudWatch/Container Insights + EventBridge on task state changes
```

## 3. When to use

- **Run containers on AWS without managing Kubernetes** — task/services model is simpler than K8s
- **Serverless containers** — **Fargate**: no nodes to patch/scale; per-task CPU/memory billing; spikes without cluster ops
- **Cost-sensitive interruptible workloads** — **Fargate Spot** (batch, dev/staging, stateless)
- **Steady/high-volume/gpu/stateful compute** — **EC2 capacity providers** (or **Managed Instances** for AWS-managed EC2 with full instance access)
- **Deep AWS integration required** — ALB/NLB, IAM task roles, CloudWatch, EFS, Secrets Manager out of the box
- **Single app or few microservices** without needing a K8s ecosystem
- **Batch/convergent jobs** — ECS tasks as one-off jobs
- **On-prem + cloud same orchestrator** — **ECS Anywhere** (External capacity)

## 4. When NOT to use

- **Need portability / standard K8s APIs / Helm / Istio / existing K8s skills** → **EKS**
- **Need to run the same orchestrator on-prem + AWS identically** → **EKS** (K8s everywhere)
- **K8s community ecosystem, operators, CRDs, Argo CD pipelines** → **EKS**
- **Very large microservice estate (~15-20+ services) where platform teams want K8s patterns** → **EKS**
- **Pure Lambda-able functions** — a single short-running event handler is often better as **Lambda** (though containers win for binaries >50 MB/zip limits, long timeouts, custom runtimes)
- **Windows containers / complex Windows workloads** → Fargate/ECS partial support exists (Windows Server on Fargate/EC2) but evaluate FSx/EC2 hosting
- **Kafka-like streaming with heavy tooling** → separate platform, not ECS
- **You need on-prem bare control** → EKS Hybrid or classic non-AWS K8s

## 5. Important features

- **Launch/capacity agility** — capacity providers: **Fargate + Fargate Spot + EC2 ASG + Managed Instances**; a **strategy** can weight providers (e.g., 70% on-demand / 30% spot); tasks launch ~500/min (EC2) / ~250/min (Fargate)
- **`awsvpc` network mode** — each task gets its **own ENI + security group** (Fargate requires it); per-task isolation; best security posture
- **IAM task roles** — container gets its own IAM role (no credentials in code) — a strong security story
- **Load balancing & service discovery** — ALB (L7, host/path-based), NLB (L4/TCP), GWLB; **Cloud Map** service discovery for dynamic DNS
- **Service Auto Scaling** — **target tracking** on CPU/memory/ALB request count; also **scheduled scaling**; scales the service (and EC2 CP cluster) automatically
- **Rolling updates & deployment types** — rolling, blue/green via **CodeDeploy**; **canary deployments**
- **EFS support (Fargate + EC2)** — shared persistent storage across tasks/AZs (stateful containers serverless)
- **Secrets Manager / SSM Parameter Store** integration; **ENV/Files**; container **health checks** (now exposed via Container Insights metric)
- **Container Insights + enhanced observability** — cluster→service→task→container metrics, drill-down dashboards, `UnHealthyContainerHealthStatus` metric for alarms
- **EventBridge integration** — task/service **state-change events** (start/stop/why) → alerts/automation
- **ECS Anywhere** — manage on-prem containers with the same ECS APIs
- **SOCI lazy image loading** — faster task startup (Fargate/Linux)
- **No control plane cost** — free service; you pay for compute only
- **Fargate platform versions** — Linux (Amazon Linux 2 / Bottlerocket), Windows Server 2019

## 6. Limitations

- **No Kubernetes APIs/ecosystem** — proprietary model; can't run Helm/Istio/CRDs as-is
- **Fargate constraints** — no SSH to host, no custom AMI/CNI, **no EBS volume attach**, **no DaemonSet-equiv**, `awsvpc` only, no public-subnet pods (EKS Fargate); large/steady cost premium vs fully-packed EC2
- **Fargate Spot interruptible** — 2-min warning; not for HA-critical workloads; may be unavailable during heavy demand (no auto fallback to on-demand)
- **EC2 scheduling** — need to manage instance fleet, capacity, patching (or use Managed Instances)
- **Capacity strategy limits** — a provider strategy can't mix Fargate and EC2 providers in one strategy; weight/base semantics; immutable per-service strategies
- **Task limits** — CPU/memory per task caps; ENI limits per account; image pull maturity; lifecycle of containers tied to task
- **Fargate per-task billing overhead** at scale — running many small tasks costs more than bin-packing on EC2
- **No native GitOps/declarative API** — need CDK/CloudFormation/Terraform or CodePipeline for IaC (fine, but not K8s-style apply)
- **Windows support narrower** on Fargate (platform/version constrained)
- **No per-pod network policies** beyond SGs/awsvpc — K8s network policies are an EKS thing
- **Job orchestration** — use ECS scheduled tasks or integrate Step Functions; not a DAG engine

## 7. Trade-offs

- **ECS (Fargate) vs EC2 capacity** — zero ops, per-task cost, isolation vs control, bin-packing savings at scale, GPU/stateful/steady workloads; **Managed Instances** splits the difference
- **Fargate vs Fargate Spot** — reliability vs ~50-70% discount (batch/dev/staging)
- **ECS vs EKS** — simplest ops + AWS-native integrations vs **K8s portability/ecosystem**; 82% of container users run K8s (2025), so platform teams often pick EKS even on AWS
- **ECS vs Lambda** — long-running, custom runtime, large footprints, persistent apps vs event-driven short functions (trivial to scale/keepalive)
- **ECS vs App Runner** — full control (tasks/services/networking) vs zero-config deploy-a-container (single image)
- **ECS vs Beanstalk (Docker)** — production-grade orchestration vs simple single-container deploys
- **Managed Instances vs Fargate vs ASG** — AWS-managed EC2 flexibility vs total serverless vs self-managed fleet
- **awsvpc vs bridge/host** — per-task ENI/SG (best isolation) vs instance-sharing networking (more classic Docker)
- **Container Insights vs external observability** — native per-observation pricing vs Prometheus/Grafana ecosystem

## 8. Architecture

Reference patterns:

```
Serverless web tier:
  Route53 → ALB → ECS service (Fargate, awsvpc, task role, target count)
              ├─ scales via target-tracking on request count/CPU
              └─ tasks across AZs; EFS mount for shared uploads; secrets from Secrets Manager

Cost-optimized bursty:
  ECS service capacity strategy: Fargate weight 1 + Fargate Spot weight 2
  (spiky batch/deploy workloads ride spot; core stays on-demand)

Stateful/GPU:
  ECS on EC2 capacity provider (ASG, gpu instances) → tasks on EBS-backed instances + EFS for shared state

CI/CD + on-prem:
  CodePipeline → ECR → ECS rolling/blue-green via CodeDeploy; ECS Anywhere runs same tasks on-prem
```

## 9. SAA-C03 Perspective

ECS matters for **resilience, performance, and cost** (Domains 2, 3, 4):

- **"Run containers serverless, no node management"** → **ECS on Fargate**
- **"Cut costs for interruptible batch/stateless containers"** → **Fargate Spot** (2-min warning)
- **"GPU / large steady / stateful containers"** → **ECS on EC2 (capacity provider, maybe Managed Instances)**
- **"Need Kubernetes portability / standard K8s APIs"** → **EKS**, not ECS
- **Task definition = image + CPU/memory + IAM role + ports**; **task role** for least-privilege
- **`awsvpc` mode** → per-task ENI + SG (security questions)
- **ALB/NLB target type = `ip` for Fargate**; canary = CodeDeploy blue/green
- **EFS = shared storage across Fargate tasks**; **EBS not supported on Fargate**
- **Service Auto Scaling** on CPU/memory/request count (target tracking)
- **Capacity providers > launch types** (recommended model)
- **Container Insights** for monitoring; **EventBridge** for task state automation

Exam traps: "Fargate supports EBS attach" → **no, use EFS**; "Fargate lets you SSH into nodes" → **no nodes visible**; "ECS = Kubernetes" → **no, different orchestrator**; "ECS service scales instances" → **it scales tasks** (cluster provider scales instances if configured); "Fargate runs DaemonSets" → **that's EKS, and even EKS Fargate doesn't support DaemonSets**.