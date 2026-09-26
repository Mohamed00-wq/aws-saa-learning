# AWS Service Catalog  Approved Service Catalogs

## Purpose

AWS Service Catalog is a **governance service for standardized self-service provisioning**. Admins package approved, compliant infrastructure (backed by **CloudFormation templates**) into **Products**, group them into **Portfolios**, and share those portfolios with users or accounts. End users launch pre-approved resources **without needing direct IAM permissions**. For the SAA exam it's the answer for **controlling what users can create, enforcing tags, and centralizing approved blueprints**.

## Main use cases

- **Curated catalogs of approved services**  users pick from a menu instead of getting broad AWS access
- **Self-service provisioning** of standard environments (dev VPCs, RDS, reusable EC2 baselines) without creating IAM admin access
- **Tag enforcement** on everything launched via **TagOptions**
- **Cross-account / organization-wide blueprints** via portfolio sharing (IAM users, specific accounts, or **all Org accounts**)
- **Restricting user inputs** with **Template constraints** (e.g. force `t3.small`, block large instance types)

## Key features

- **Core components**  **Product** (deployable item = CF template), **Portfolio** (collection of products), **Constraint** (policy on a product), **TagOption** (predefined tag applied on launch)
- **Constraints**  **Launch** (assigns an IAM role so users don't need direct permissions  the key governance feature), **Notification** (SNS on launch/update), **Tag update** (control post-launch tag changes), **Template** (limit parameter values users can supply)
- **TagOptions**  library of allowed tag key-values users pick from the list, can't invent tags
- **Sharing modes**  IAM (users/roles/groups), Account (specific AWS accounts), **Organizations** (all accounts, most scalable) shared portfolios show as **imported** and stay in sync
- **Governance for any IaC tool**  native CloudFormation products, Terraform wrapped in a CF template or Lambda-backed custom resource
- **Pricing**: free (pay for the provisioned resources only)

## When to use

- Need to **give many users the power to launch standard resources safely**
- Need **approval-free but governed** provisioning (users don't get console IAM rights, and **can't see the underlying template**)
- **Large orgs (multi-account)** standardizing reusable infrastructure with **Organizations sharing**
- Enforcing **cost-allocation tags and allowed instance types** at launch time

## Important limitation

- **No ongoing management**  after launch, **CloudFormation owns the resources**, and **TagOptions are not enforced post-launch** (use **AWS Config** for continuous compliance). **End users can't read the template**, only descriptions and parameters, which limits debugging. **Region-specific** (share per Region). Works best with CloudFormation-native products Terraform needs wrapping.

## SAA relevance

- "**Standardized, governed self-service provisioning** of AWS resources" -> **Service Catalog**
- "**Users launch pre-approved resources without IAM permissions**" -> **Launch constraint** (IAM role)
- "**Enforce tags and allowed instance types on user launches**" -> **TagOptions + Template constraints**
- "**Roll out approved blueprints across the organization**" -> Portfolios + **Organizations sharing**
- "**Post-launch compliance**" -> pair with **AWS Config**
- Exam trap: Service Catalog is a **provisioning gate, not a runtime manager**  it delegates to **CloudFormation**, and it **cannot enforce tags after the fact** (Config does that).