# AWS CloudFormation  Infrastructure as Code (IaC)

## Purpose

AWS CloudFormation is AWS's **native Infrastructure as Code** service: define resources in **YAML/JSON templates**, provision them as **stacks**, and update or delete everything together. It's the foundation that **AWS CDK and AWS SAM** synthesize into. For the SAA exam it's the default answer for **repeatable deployments, change management (change sets), drift detection, and cross-account/region infrastructure (stack sets)**.

## Main use cases

- **Repeatable, versioned infrastructure** provisioning (S3, EC2, IAM, DNS, load balancers, databases) from templates
- **Application deploys** with bootstrapping via **cfn-init/cfn-signal/cfn-hup** (install packages, configure services, signal readiness)
- **Multi-account and multi-region rollouts** using **stack sets**
- **Modular infrastructure** via **nested stacks** and **cross-stack references** (export/import)
- **Update management** with **change sets** (preview creates/modifies/deletes/replacements) and **drift detection**
- **Governance** with stack policies and termination protection custom/3rd-party resources via **custom resources** (Lambda/SNS)

## Key features

- **Templates & stacks**  Template (YAML/JSON desired state) -> **Stack** (live instance, one Region)  delete stack = delete resources it created (unless protected)
- **Stack sets**  deploy the same template across **multiple accounts/regions** from one admin account (needs Admin + Execution cross-account roles)
- **Template anatomy**  Parameters (inputs), Mappings (lookups e.g. Region->AMI), Conditions, Resources (**required**), Outputs (export values)
- **Intrinsic functions**  `Ref`, `Fn::GetAtt`, `Fn::Sub`, `Fn::Join`, `Fn::If`, `Fn::ImportValue`, `Fn::Cidr`
- **Lifecycle controls**  Change sets (preview), **drift detection** (report actual vs template), **stack policies** (guard resources during updates), **termination protection**
- **DeletionPolicy / UpdateReplacePolicy**  `Retain` (keep resource), `Snapshot` (backup before delete, e.g. RDS/EBS)
- **cfn-init / cfn-signal / cfn-hup**  first-boot setup, success/failure signaling (prevents creation timeout), metadata-driven updates
- **Custom resources** extend native support (Lambda/SNS) **SAM** simplifies serverless apps and transforms to CloudFormation

## When to use

- **Any IaC need**: template-based, auditable, repeatable AWS provisioning
- Change-control workflows where you must **preview and approve infrastructure changes**
- Deploying **identical baselines** (logging, security, guardrails) across whole organizations (stack sets)
- **Combined with CDK** (which synthesizes to CloudFormation) and **SAM** for serverless
- Ensuring **drift is caught** and resources are protected from accidental deletion

## Important limitation

- **Not a runtime/system config manager** on its own (use **cfn-init** or tools like Ansible/SSM for that). **Replaces resources** during updates (always review change sets) and **stack deletion deletes everything** unless you set `DeletionPolicy`. Template size limit (~51 KB), 500-resource stack limit (use nested stacks). **Drift detection is manual, not automatic**. Cross-stack exports cannot be deleted while imports exist.

## SAA relevance

- "**Repeatable / auditable / IaC deployment** of AWS resources" -> **CloudFormation** (or CDK atop it)
- "**Preview infra changes / approval-gated updates**" -> **Change sets** "**compare actual config vs template**" -> **Drift detection**
- "**Deploy same resources to many accounts/regions**" -> **Stack sets** "**modular/reusable templates**" -> **Nested stacks**
- "**First-boot config of EC2**" -> **cfn-init/cfn-signal** "**protect an RDS DB from stack deletion**" -> **DeletionPolicy: Retain/Snapshot**
- "**Serverless app in CF**" -> **AWS SAM** "**use CDK**" -> still CloudFormation under the hood
- Exam trick: CloudFormation is **free** (you pay only for created resources), **YAML/JSON only** (CDK is the code option), and updates use **change sets** while **EC2 user-data** is not OS-level config management alone.