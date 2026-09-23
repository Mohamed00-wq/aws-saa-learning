# ECR Course — Elastic Container Registry

## 1. Purpose

ECR is AWS's **fully managed container image registry** — the secure home for Docker/OCI images and Helm charts used by **ECS, EKS, Lambda (container images), and Fargate**. It gives you **IAM-controlled private or public repositories**, **vulnerability scanning**, **lifecycle policies** for cleanup, **cross-region/account replication**, and **pull-through caching** of upstream registries — no self-hosted registry to run.

## 2. How it works

- **Repositories** live in a **private registry** (per account/Region; also a **public registry** for OSS images)
- **Push/pull** uses standard Docker/containerd via `docker push/pull` (or `crane`/`cosign`), authenticated with **`GetAuthorizationToken`** (12-hour token) or task/role-based access
- **Trusted identity** — IAM policies on repos/registry; ECS/EKS/Lambda pull with IAM (no stored passwords)
- **Image layers** stored in the repo (blob-mounted across repos in a registry); tags + digests of image manifests
- **Scanning** — **basic scanning** (AWS-native, free, on-push or on-pull) and **enhanced scanning** (continuous, **Amazon Inspector**, OS + language packages + Lambda functions)
- **Lifecycle policies** — JSON rules (priority-ordered) that **expire or archive** unused images (by count, days-since-pushed, days-since-pulled, age)
- **Replication** — copy images cross-Region and/or cross-account automatically (registry settings)
- **Pull-through cache rules** — cache images from **Docker Hub, ECR Public, Kubernetes registry, Quay, GitHub, GitLab, Azure, Chainguard** into your private registry (namespace-prefixed)

```
CI/CD (CodeBuild/BuildKit, signed) → ECR repo (private, IAM policy, KMS encryption)
   ├─ scanning (basic/Inspector enhanced) → findings/alerts
   ├─ lifecycle policy → expire/archive stale images ≤24h
   ├─ replication → other Regions/accounts
   └─ pull-through cache ← upstream public registries
   → pulled by ECS/EKS/Lambda(Fargate) via IAM (no creds in code)
```

## 3. When to use

- **Store & distribute container images** for **ECS, EKS, Fargate, Lambda (image packages)**
- **Secure/private images** — IAM-based access, KMS/SSE encryption, no public exposure
- **Automated cleanup** — lifecycle policies prune stale tags/images (prevent repo bloat/cost)
- **Compliance/security** — vulnerability scanning (Inspector), image signing (Notary/AWS Signer), immutable tags
- **Cross-Region/account DR or publishing** — replication to other regions/accounts
- **Avoid Docker Hub rate limits / faster pulls in prod** — **pull-through cache** + replication into your VPC-adjacent registry
- **Vitreous supply chain** — mirror public images (K8s, Quay, Helm) into your registry for control/audit

## 4. When NOT to use

- **No containers** — use S3 for artifacts/binaries, CodeArtifact for language packages
- **Public OSS images only, no control needed** — just pull from Docker Hub/ECR Public directly (pay rate-limit risk/latency vs cache setup)
- **Simple CI artifact storage** (not images) → S3/CodeArtifact
- **Helm chart-heavy workflow**: ECR supports **OCI Helm charts repos** in OCI format; if you prefer classic Helm repos/toolchains → other Helm repositories or use ECR OCI mode
- **You need to store large non-image blobs** → S3 (cheaper for generic objects)
- **Truly ephemeral local dev** → local Docker daemon registry is fine
- **Full-featured third-party registry ecosystem** (e.g., Harbor, JFrog with many artifact types) → consider those if you want multi-artifact, UI, signing, policies beyond images

## 5. Important features

- **Private + public registries** — per-account private; AWS-managed **public** registry for open-source
- **IAM + registry/repo policies** — fine-grained pull-only (`AmazonEC2ContainerRegistryPullOnly`), push, admin; no passwords for AWS compute
- **Encryption** — **SSE-KMS** or AES-256; in-transit TLS
- **Image scanning** — **basic** (AWS-native, free — Clair-based scanning **deprecated Feb 2026**, native basic replaces it) and **enhanced** (**Inspector**: continuous OS + programming-language + **Lambda function** scanning, findings via Inspector/EventBridge/Security Hub)
- **Lifecycle policies** — rule **priority order** (1 = highest), `tagStatus`, `tagPrefixList`/`tagPatternList` (wildcards), `imageCountMoreThan`, `sinceImagePushed` (days), **`sinceImagePulled`** (last-pull time), actions **`expire` or `archive`**; runs **within 24h**; **preview** before applying; pull-time update exclusions to protect CI images
- **Replication** — **Cross-Region + cross-account** (registry replication config; per-repo filtering; service-linked role)
- **Pull-through cache** — automate sync from upstream registries (auth secrets in Secrets Manager for private upstreams); repository creation templates standardize names/settings
- **Immutable tags & Manifest list support** — prevent overwrites; multi-arch images
- **Image signing** — ECR + AWS Signer (Notary) to sign/verify images in supply chains
- **Blob mounting / layer sharing** — share common layers across repos in the registry (save storage/transfer)
- **Registry settings dashboard** — scanning, replication, pull-through cache, repository templates centrally
- **ECR TaskDefinition mount / sizing** — Fargate StartTaskService has no extra setup; images pulled via IAM
- **EventBridge/CloudWatch** integration for scan/lifecycle/events; **CloudTrail** audit of registry API calls
- **Lambda container images** — Lambda supports images from ECR up to **10 GB**

## 6. Limitations

- **Per-Region/account organization** — registries are regional; replication config is per-Region
- **Registry-wide settings** are regional — configure per Region (scanning defaults, pull-through cache rules)
- **Lifecycle policy is per-Repository, per-Region** — replicate poles need policies in each Region/account (no global rule)
- **Pull-through cache auth** — secrets in Secrets Manager required for private upstream registries
- **Enhanced/inspector scanning costs** — continuous scanning has per-image/GB pricing; basic scanning is free but intermittent
- **Image/pull throughput** — registry endpoints have account-level API limits (raisable); cold pulls of big images need time (SOCI/lazy loading helps consumers)
- **No generic binary artifact support** — containers/OCI/images (and OCI Helm charts) only
- **GetAuthorizationToken scope** — tokens are 12 hours; long-lived robot accounts (CI) must renew / use EKS/ECS IAM instead
- **Windows images** — supported but scan/lifecycle tooling nuances
- **Public ECR** — pull-only for your published images via role; no fine-grained private ACLs there
- **Digest vs tag** — spoofing risk if you rely on mutable tags; use immutable tags/digests/signing

## 7. Trade-offs

- **ECR vs Docker Hub / self-hosted (Nexus/Harbor/GitLab Registry)** — managed, IAM-native, regional, scan/lifecycle built-in vs public default or full-featured multi-artifact open-source
- **Basic vs enhanced (Inspector) scanning** — free on-push vs continuous + language-level + Lambda support (cost)
- **Lifecycle expire vs archive** — permanently delete vs keep in an **archive tier** for compliance/rollback (archive costs more storage but retains images)
- **Replication cross-Region vs pull-through cache** — durable copies for DR/distribution vs automatic upstream syncing for availability/rate-limits
- **M&Ablob mounting vs duplicated images** — shared layers save cost on many repos vs simpler per-repo storage
- **Private vs public registry** — your workloads vs your OSS distribution
- **ECR vs S3 for releases** — image layers/lifecycle/scanning ecosystem vs generic object storage (cheaper for non-image artifacts)
- **cosign/Notary signing ceremony** — strong supply chain vs added pipeline complexity
- **Immutability on vs off** — tag safety/supply-chain vs simple dev iteration with mutable `latest`

## 8. Architecture

Reference supply-chain patterns:

```

CI/CD:
  CodeCommit/GitHub → CodeBuild build → scan (basic on-push) → push to ECR (immutable tags)
     → lifecycle policy keep last N tags + expire >90 days (preview first)
     → ECS/EKS deploy (rolling/blue-green) pulling with task/node IAM

DR/distribution:
  ECR (us-east-1) —replication→ ECR (eu-west-1)  (and prod account) → EKS/ECS consumers local pulls

Supply chain hardening:
  ECR (pull-through cache for Docker Hub / K8s / Quay) → ECS/EKS images
  → enhanced Inspector scanning continuous → findings → EventBridge → remediation
  → signed images (AWS Signer/Notary) verified at deploy
```

## 9. SAA-C03 Perspective

ECR appears in **container, security, and cost** scenarios (Domains 1, 3, 4):

- **"Store/secure container images used by ECS/EKS/Fargate/Lambda"** → **ECR** (private repo, IAM)
- **"Clean up stale images / control registry growth"** → **ECR lifecycle policies** (expire/archive, preview)
- **"Scan for vulnerabilities"** → **basic** or **enhanced (Inspector — continuous, language + Lambda)**
- **"Replicate images to another Region/account"** → **registry replication**
- **"Avoid Docker Hub rate limits; cache public images locally"** → **pull-through cache rules**
- **"Protect against tag tampering/supply chain"** → **immutable tags + image signing**
- **Auth model** — ECS/EKS/Lambda pull with **IAM** via `GetAuthorizationToken`; never embed creds
- **Lambda supports container images from ECR (up to 10 GB)**
- **Clair-based scanning deprecated (Feb 2026)** — native basic scanning now
- **Lifecycle policies run within ~24h; rules evaluated by priority**

Exam traps: "ECR is only for EKS" → **also ECS, Fargate, Lambda (images)**; "lifecycle policy cleans cross-region replicas" → **it acts on its own Region only**; "basic scanning is continuous" → **enhanced/Inspector is continuous; basic is on-push**; "ECR = Docker Hub" → **regional private registry with IAM**; "scanning is free forever" → **basic free, enhanced costs**; "registry replication is global" → **configured per Region**.