# AWS Compute Optimizer  Rightsizing / Cost-Optimization Recommendations

## Purpose

AWS Compute Optimizer **uses ML on your CloudWatch utilization + resource configuration** to recommend the **optimal compute resources** for your workloads  primarily **EC2 instance types/sizes (rightsizing)**, plus **Auto Scaling groups, EBS volumes, Lambda functions, ECS services on Fargate, and commercial software licenses**. It classifies findings (over-provisioned / under-provisioned / optimized) and reports **estimated monthly savings (USD, before/after discounts) and performance risk**, refreshed daily. Free (beyond base CloudWatch monitoring) **enhanced infrastructure metrics** (up to 93 days of history) is paid.

## Main use cases

- **EC2 rightsizing**  find over-provisioned instances and match instance type/family/size to actual load
- **Idle resource cleanup**  spot/terminate idle instances, idle EBS volumes (unattached/underused)
- **Optimization of ASGs, Lambda (memory/CPU config), Fargate (vCPU/mem), and license (BYOL/SQL per-core) configs**
- **Waving cost governance**  prioritize savings via the Compute Optimizer dashboard, Cost Optimization Hub, or Cost Explorer rightsizing (a subset surfaced with billing data)
- **Automation**  recommendations via APIs/EventBridge → Step Functions/Lambda to apply (e.g., change instance type, resize Fargate)

## Key features

- **ML-based recommendations** across EC2 instances, ASGs, **EBS volumes**, **Lambda functions**, **ECS on Fargate**, **commercial software licenses (incl. SQL Server per-core)**
- **Findings-driven**  under-provisioned / over-provisioned / optimized / none up to **3 alternative instance types (Option 1-3)** per resource
- **Savings & risk surfaces**  estimated **monthly savings**, savings opportunity %, **performance risk** per option
- **Metrics**  CPU, network I/O, local disk I/O/throughput (default, 14-day lookback) **memory** via CloudWatch agent (or Datadog/Dynatrace/Instana/New Relic ingestion) up to **93 days** with enhanced metrics
- **Integration**  Console dashboard, Cost Explorer rightsizing, Cost Optimization Hub (discount-aware, after-discount savings), licensing optimizer (SQL per-core rightsizing lowers license cost)
- **Actionable**  API/CLI/console apply/export **no agents required** for core CPU/I/O recommendations

## When to use

- Proactively find **underutilized EC2/ASG instances** to downsize or terminate
- Quantify **savings potential** and hidden idle spend for a cost-optimization report
- Rightsize **EBS volumes, Lambda memory, or Fargate sizes** to the workload's real profile
- Reduce **software-license cost** (e.g., SQL Server) by matching instance size to CPU needs
- Continuous cost governance with **automated remediation** pipelines

## Important limitation

- **Recommendations only as good as the metrics you have**: default lookback is **14 days** **memory is not collected by default** (needs the CloudWatch agent / external metrics)  without it Compute Optimizer conservatively avoids downsizing that dimension. It's **historical, not forecasting**  pick a size that fits *future* peaks and it **does not size your app**  real load vs synthetic tests matter. Dashboards are refreshed **daily** and most results assume On-Demand vs discount shares (check Savings Plans/RI interplay via Cost Explorer). No recommendations exist for idle/non-reportable or <30-hr resources **you still decide**  it recommends, you apply.

## SAA relevance

- "**Right-size EC2 / find over-provisioned instances / optimize instance types**" → **AWS Compute Optimizer**
- "**ML-based utilization recommendations** for EBS/Lambda/Fargate/ASG" → Compute Optimizer
- "**Identify idle resources / savings opportunities** month-over-month" → Compute Optimizer dashboard (+ Cost Explorer rightsizing, Cost Optimization Hub for after-discount dollars)
- "Automated near-real-time scaling" → **Auto Scaling / Aurora Serverless** (not Compute Optimizer  it's a *recommender*, not a scaler)
- Exam traps: Compute Optimizer **recommends** (it does not resize, nor replace Auto Scaling) key inputs = **CloudWatch metrics (14 days)** + optional **memory via agent** free default distinguishes **under-provisioned vs over-provisioned vs optimized** pair with **Trusted Advisor / Cost Explorer / Cost Anomaly Detection** for full FinOps.