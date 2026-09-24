# AWS Elastic Beanstalk Course — PaaS: Deploy Apps Without Managing Servers

## 1. Purpose

Elastic Beanstalk is AWS's **Platform-as-a-Service (PaaS)**: you upload your application code and EB automatically deploys and manages the underlying infrastructure — capacity provisioning, load balancing, Auto Scaling, health monitoring, and patching. It "reduces management complexity without restricting choice or control" — you still own the environment and can access the underlying resources. For SAA it's the answer to **"deploy a web app quickly without infra ops / PaaS"**.

## 2. How it works

- **Upload an application version** (zip/war) → EB builds the environment: EC2 instances, ALB, Auto Scaling group, security groups, health checks
- **Environment types** — **load-balanced/scalable** (ALB + ASG, multi-AZ) vs **single-instance** (one EC2 with Elastic IP, no LB — cheap for dev) vs **worker environment** (tier that drains an **SQS queue**, scales instances per queue depth)
- **Supported platforms** — Java, .NET, Node.js, PHP, Ruby, Python, Go, and **Docker** (and multicontainer Docker → ECS)
- **Deployment policies** — **All-at-once** (fastest, downtime), **Rolling** (in batches), **Rolling with additional batch** (full capacity), **Immutable** (new ASG, zero downtime, slowest/safest), **Traffic splitting** (canary % of traffic to new version)
- **Blue/green** — deploy new version to a separate environment then **swap environment URLs (CNAME swap)** to switch instantly
- **Configuration** — `.ebextensions/*.config` files, platform updates (managed), **EB CLI**, DynamoDB-based saved configs
- Monitoring — **basic health** (instances) vs **enhanced health** (per-request rollup); integrates **CloudWatch** and **X-Ray**

```
EB console / CLI → upload version
  → EB provisions EC2 + ALB + ASG + health checks (web tier)
  → or Worker tier: instances pull from SQS queue
Deploy options: all-at-once | rolling | rolling+extra batch | immutable | traffic-split | blue/green (URL swap)
```

## 3. When to use

- **Developers who want to deploy web apps fast** without managing EC2/ALB/ASG themselves
- **Standard web applications** on supported runtimes (Java/.NET/PHP/Ruby/Python/Node/Go/Docker)
- **MVPs / internal tools / demos** — lowest friction to production
- **Teams that need managed infra now, more control later** (EB exposes the EC2/ASG/ALB behind it)
- Simple **background worker** apps pulling from SQS (worker environment)

## 4. When NOT to use

- **Full infrastructure control** (custom VPC topology, install own agents, tweak every layer) → EC2 + CloudFormation/CDK
- **Container-native orchestration** (microservices, service discovery, rolling with many services) → **ECS/EKS**
- **Event-driven serverless** → **Lambda + API Gateway** (automatic scale, pay-per-use)
- **Complex custom architectures / stateful fleets** — EB is opinionated, few architectural patterns
- **You only want Infrastructure as Code with no extra abstraction** → CloudFormation/CDK

## 5. Important features

- **Platforms incl. Docker** — and multicontainer (ECS-backed) environments
- **Deployment policies** — All-at-once, Rolling, Rolling with additional batch, **Immutable**, **Traffic splitting** (canary)
- **Blue/green via CNAME/URL swap** — near-zero-downtime releases and rollback
- **Single-instance vs load-balanced (multi-AZ)** environment types
- **Worker environment (SQS)** — decouples background processing; scales with queue length
- **Managed platform updates** with Immutable deployment
- **Enhanced health reporting**, CloudWatch metrics/alarms, X-Ray tracing
- **RDS integration**, environment variables/configuration files, EB CLI

## 6. Limitations

- **Limited to supported platform runtimes** — can't run arbitrary infrastructure
- Less control than raw EC2; environment configuration follows EB's model
- **Config changes are often applied as rolling/immutable and some require instance replacement**
- In-place version updates cause brief unavailability unless you use immutable/blue-green
- **Database-inside-environment is discouraged** — decouple RDS for blue/green (retain lifecycle option)
- Out-of-band manual changes to resources confuse EB's model

## 7. Trade-offs

- **Beanstalk vs EC2** — zero-infra-ops PaaS (fast, opinionated) vs full control and responsibility
- **Beanstalk vs ECS/EKS** — single-platform web apps vs container orchestration/microservices
- **Beanstalk vs Lambda/API GW** — managed servers + scaling vs fully serverless pay-per-use
- **All-at-once vs Immutable** — fastest/cheapest with downtime vs zero downtime at double capacity cost
- **Rolling vs Rolling-with-extra-batch** — capacity dip during batch vs extra capacity to keep full
- **Traffic splitting vs Blue/green** — canary percentage on one env vs full separate environments (also for platform upgrades)

## 8. Architecture

```
Web tier: Route 53 → ALB → ASG of EC2 (health-enhanced) ← EB deploys platform + app
Worker tier: producers → SQS → workers (ASG scales with queue depth)
DB: RDS kept OUTSIDE the environment (decoupled) for blue/green to work
Release: immutable/traffic-split on same env, or separate env + URL swap (blue/green)
```

## 9. SAA-C03 Perspective

- **"Deploy a web app without managing servers, PaaS"** → **Elastic Beanstalk**
- **"Zero-downtime deployment, slowest/safest"** → **Immutable**; "canary" → **Traffic splitting**
- **"Switch traffic to a new version instantly with separate environments"** → **Blue/green + CNAME swap**
- **"Application processes jobs from a queue, scales automatically"** → **worker environment (SQS)**
- **"Fast deployment, downtime acceptable"** → **All-at-once**; "Cheap dev environment" → **single-instance**
- **"Container microservices / orchestration"** → not EB — **ECS/EKS**

Exam traps: "Beanstalk manages containers with no config" → **EB uses ECS for multicontainer, but is really for classic web apps**; "Beanstalk is serverless" → **no, it provisions EC2**; "all-at-once is safest" → **no, immutable/traffic-split are**; "put RDS inside the environment" → **decouple for production/blue-green**.