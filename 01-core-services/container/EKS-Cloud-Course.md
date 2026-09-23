# EKS Course — Elastic Kubernetes Service

## 1. Purpose

EKS is AWS's **managed Kubernetes service**: a highly available, patched **control plane** across 3 AZs plus the managed **worker nodes** you choose. You keep the **standard Kubernetes API** — manifests, Helm, operators, Istio, Argo CD, kubectl — running on other clouds or on-prem too. For SAA, EKS is the answer whenever the requirement is **"run Kubernetes on AWS"** or **"portable container platform conforming to upstream K8s"**.

## 2. How it works

- AWS runs a **managed control plane** (API server, etcd, controllers) **HA across 3 AZs**, patches/upgrades it, exposes via a Cluster API endpoint
- **Compute options** pick how worker capacity runs:
  | Option | Managed by | Notes |
  |---|---|---|
  | **EKS Auto Mode** (recommended) | AWS manages nodes, networking, storage, load balancing, upgrades | **Karpenter** node lifecycle/consolidation; fully K8s-conformant; GPU/Spot/ARM support; single containers-per-node isolation like Fargate |
  | **Managed Node Groups** | AWS manages instance fleet health/patches | ASG-backed; keep control of scaling/instance types |
  | **Self-managed nodes** | You run EC2 + ASG yourself | max control, more ops |
  | **Fargate profiles** | **Serverless pods** — no nodes | each pod in isolated microVM; limited (no DaemonSets, no EBS, EFS yes, no custom CNI) |
- **Karpenter** (Auto Mode) — modern node autoscaler: provisions right-sized instances for pending pods, **consolidates/right-sizes** underutilized nodes for cost
- **Cluster add-ons / components** — VPC CNI, CoreDNS, kube-proxy, AWS Load Balancer Controller, EBS/EFS CSI drivers; **Pod Identity / IRSA (IAM roles for service accounts)** gives pods IAM roles
- **Networking** — VPC with per-account node/svc/pod CIDRs; **AWS Load Balancer Controller** (ALB/NLB), Gateway API, network policies; **Node-local DNS**, separate pod subnets/SGs (Auto Mode)
- **Storage** — **EBS** (block, per node/AZ), **EFS** (shared across AZs/pods), FSx for Lustre (HPC)
- **Upgrades** — control plane + node (managed in Auto Mode), version skew rules apply

```
kubectl / manifests / Helm → EKS managed control plane (HA, 3 AZs)
   → compute: EKS Auto Mode (Karpenter) / Managed Node Groups / self-managed / Fargate profiles
   → VPC CNI + Load Balancer Controller → services to ALB/NLB
   → Storage: EBS / EFS CSI; IAM: Pod Identity / IRSA
   → Observability: Container Insights (CNI metrics, PromQL), CloudWatch, CloudTrail
```

## 3. When to use

- **"Run Kubernetes / migrate existing K8s workload"** on AWS
- **Portability** — same K8s API as GKE/AKS/on-prem; Helm, Argo CD, Istio, Operators, CRDs all work
- **K8s ecosystem leverage** — standard tooling/skills; independent of AWS API
- **Microservice-heavy platforms** — many services (~15-20+) with service mesh/network policy/IPv6/K8s-native scaling needs
- **Fully-managed compute, K8s-native** — **EKS Auto Mode** (recommended): right-sized nodes via Karpenter, auto upgrades, handled networking/storage/LB
- **Mixed instance optimizations** — Auto Mode supports Spot, GPU, ARM/Graviton, Savings Plan integration
- **Isolated serverless pods** — **Fargate** for spiky/high-security/burst pods
- **Hybrid/multi-cloud** — same platform on-prem (EKS Anywhere/Hybrid) and in-cloud

## 4. When NOT to use

- **Don't need K8s, want simplest container ops on AWS** → **ECS** (task definitions, native integrations, no K8s API)
- **Pure serverless functions** → Lambda
- **Single-image zero-ops apps** → App Runner / Beanstalk
- **Only a few services, no K8s team or ecosystem** → ECS/Fargate is lighter to operate
- **Fargate-only with full K8s features** — Fargate profiles lack DaemonSets/custom CNI/EBS; use Auto Mode or node groups if you need them
- **Very cost-sensitive small estate** — EKS control plane has a flat hourly cost (plus EC2); small AWS-only apps are often cheaper on ECS
- **No RAM/CPU per-pod burst vs fine-grained control need** — ensure you actually benefit from K8s
- **Legacy monolith, no containerization intent** → EC2/BEanstalk

## 5. Important features

- **Managed, multi-AZ control plane** — API server/etcd HA, patched by AWS
- **EKS Auto Mode** — **AWS takes over nodes + networking + storage + load balancing + upgrades**; **Karpenter** auto-provisions/right-sizes nodes and **consolidates** underutilized ones (faster scale-out, 30%+ more capacity, cost savings); **fully Kubernetes-conformant** (supports Istio, all upstream primitives); all EC2 purchase options incl. **Spot and GPU**, Trainium/Inferentia; per-node isolation equivalent to Fargate; **recommended over Fargate for most new clusters**
- **Managed Node Groups** — ASG-backed, AWS handles updates/health; you control scaling, instance types, launch templates
- **Fargate on EKS** — serverless pods (no nodes), **Fargate profiles** select namespaces; per-pod vCPU/GB billing; supports EFS (not EBS), NLB (IP mode), SGs per pod
- **Karpenter** — modern node autoscaler in Auto Mode; also runnable standalone (self-managed/managed node groups): node lifecycle, drift detection, consolidation
- **IRSA + EKS Pod Identity** — IAM roles for pods (least privilege); Pod Identity simplifies role assignment per pod
- **AWS Load Balancer Controller** — ALB ingress, NLB services, Gateway API; fine-grained routing
- **VPC CNI / network policies** — native pod networking, prefix delegation, IPv6/EKS clusters now supported; **node-local DNS**
- **Storage** — EBS, EFS, FSx, Local SSD via CSI/StorageClass; **topology-aware volume scheduling** (EBS binds to AZ)
- **Upgrades/patching** — EKS version management, add-ons, FIPS nodes, security group/host controls
- **EKS Anywhere / EKS Hybrid** — self-managed K8s on-prem with EKS toolsets
- **Observability** — **Container Insights with enhanced observability**, **OTel Container Insights + PromQL in Query Studio**, CloudWatch/CloudTrail control-plane logs, cluster insights
- **Cost** — control plane **flat per-hour** per cluster + nodes; **Savings Plans/Spot/RI** apply via Auto Mode/node groups

## 6. Limitations

- **Flat hourly control-plane fee** per cluster (adds up across many small clusters)
- **K8s operational knowledge required** — concepts, manifests, updates, version skew; not zero-ops
- **Managed Node Groups still need EKS care** — upgrades/AMI patching scheduling; **Auto Mode simplifies but locks to AWS-managed behavior**
- **Fargate limitations** — no DaemonSets, **no EBS**, no custom CNI/AMI, no public-subnet pods, no node SSH, only IP target LB; DaemonSet-dependent workloads (most agents, e.g., logging/metrics daemons) need real nodes or sidecars
- **Version upgrades have skew limits** — node version ≤ control plane; upgrade sequences matter
- **EBS is AZ-bound** — cross-AZ pod failover needs EFS/FSx or app-level replication
- **Control plane is AWS-managed** — you can't reconfigure etcd/API internals (fine, but limits deep tweaks)
- **Cluster creation takes minutes**; default account limits on clusters/nodes/IPs (raisable)
- **CNI/IP exhaustion** — large clusters need prefix delegation/IPv6 planning; pod subnets planning (separate pod subnets in Auto Mode)
- **Share of fault domains** — control plane is multi-AZ but still regional (DR = multi-region WDS/backup)
- **Observability/pricing complexity** — Container Insights + Prometheus metrics have their own cost models; third-party stacks add cost/setup
- **EKS Anywhere = you operate the K8s yourself** — no AWS-managed control plane on-prem

## 7. Trade-offs

- **EKS vs ECS** — portability + K8s ecosystem + standard APIs vs simplest ops + AWS-native glue + no control-plane fee
- **EKS Auto Mode vs Managed Node Groups** — AWS fully automates nodes/networking/LB/upgrades (Karpenter, recommended) vs you keep scaling/instance/launch-template control
- **Auto Mode vs Fargate** — real nodes (GPU, Spot, DaemonSets, all EC2 purchase options, full conformance, better bin-packing) vs serverless isolated pods (no node management but feature-limited); **Auto Mode now recommended over Fargate**
- **Managed nodes vs self-managed nodes** — AWS-healed/patchable fleet vs full custom AMI/labels/taint control
- **Karpenter vs Cluster Autoscaler** — right-sizing consolidation vs simple pod-feasibility scaling; newer/leaner
- **EBS vs EFS for stateful pods** — fast AZ-local block vs shared storage across AZs/pods
- **IRSA vs Pod Identity** — OIDC-federated service accounts vs simpler per-pod role assignment
- **Per-cluster control-plane fee vs many clusters** — consolidate environments or pay per cluster
- **CNI (native) vs Calico/Weave & policy engines** — simplicity vs rich policy/space
- **EKS vs Beanstalk/ECS multi-container for small teams** — platform buy-in vs operational simplicity

## 8. Architecture

Reference patterns:

```
Modern default (Auto Mode):
  EKS Auto Mode cluster → Node Pool via Karpenter (right-sized, Spot mix, consolidate)
     → ALB from ingress (AWS LB Controller) → services; EFS for shared state; Pod Identity for IAM
     → Container Insights + PromQL dashboards; auto-upgrades on

Fargate-spiking web app:
  EKS + Fargate profiles (namespace: prod-web = serverless) + node groups for stateful workers
  (no DaemonSets → sidecar for agents or keep nodes for them)

Portable multi-cloud:
  Same K8s manifests/Helm on EKS (cloud) + on-prem EKS-Hybrid/Anywhere + GKE/AKS
  → Argo CD pulls from Git; Istio mesh; DR via EKS backups + cross-region restore

High-availability stateful:
  EKS multi-AZ → stateful pods on EBS (topology-aware StorageClass) + EFS/failover for shared data
```

## 9. SAA-C03 Perspective

EKS shows up in **compute choice, resilience, and cost** (Domains 2, 3, 4):

- **"Run Kubernetes on AWS / migrate existing K8s / portable container platform"** → **EKS**
- **"Standard Kubernetes APIs, Helm, operators, Istio"** → **EKS** (ECS lacks these)
- **"Managed control plane across AZs"** → **EKS**
- **Recommended default** → **EKS Auto Mode** (Karpenter, AWS-managed nodes/networking/storage/LB/upgrades)
- **No node management, serverless pods** → **Fargate profiles** — but remember limitations (no DaemonSets/EBS/custom CNI)
- **GPU/Spot/ARM + consolidation savings** → **Auto Mode / Karpenter**
- **Pods get IAM roles** → **IRSA / Pod Identity**
- **ALB ingress/NLB services** → **AWS Load Balancer Controller**
- **Shared multi-AZ storage for pods** → **EFS** (EBS is per-AZ)
- **Control-plane flat hourly fee**; savings via Spot/RI/consolidation
- **Upgrades/version-skew** discipline; Container Insights for monitoring

Exam traps: "EKS lets you manage the control plane yourself" → **AWS manages it (HA across 3 AZs)**; "Fargate supports DaemonSets and EBS on EKS" → **no**; "EKS = ECS renamed" → **different services (K8s vs AWS-native)**; "Fargate is the recommended default for EKS" → **EKS Auto Mode is now recommended**; "control plane is free" → **EKS has a flat per-hour control-plane charge (ECS doesn't)**.