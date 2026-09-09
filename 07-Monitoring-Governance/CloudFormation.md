# CloudFormation — Infrastructure as Code

## What it is

CloudFormation is AWS's native Infrastructure as Code (IaC) service. You define your AWS resources in a YAML or JSON template, and CloudFormation provisions and manages them as a **stack**. Change the template, update the stack — CloudFormation figures out what to create, modify, or delete. No clicking in the console, no scripting one-off CLI commands.

For the SAA exam, CloudFormation shows up in any scenario about repeatable deployments, environment consistency, drift detection, cross-account infrastructure, or governance. It's also the engine behind CDK, SAM, and many AWS service features, so understanding it is foundational even if you never write a template by hand.

## Templates & stacks

- **Template**: YAML or JSON file describing desired state — resources, parameters, outputs, conditions.
- **Stack**: a live instance of a template. One stack = one deployment of the infrastructure. Deleting a stack deletes all resources it created (unless protected).
- **Stack set**: deploy the same template across **multiple accounts and regions** from a single administrative account.

## Template anatomy

| Section | Purpose |
|---|---|
| `AWSTemplateFormatVersion` | Template version (`2010-09-09` — the only one) |
| `Description` | Human-readable summary |
| `Parameters` | Input values at deploy time (instance type, CIDR, etc.) |
| `Mappings` | Static lookup tables (region → AMI ID) |
| `Conditions` | Create/skip resources based on parameter values |
| `Resources` | **Required** — the only mandatory section |
| `Outputs` | Export values (ARNs, IDs) for cross-stack references |
| `Metadata` | Helper info (e.g. CloudFormation Init for EC2 bootstrapping) |

### Intrinsic functions

| Function | Use |
|---|---|
| `Ref` | Reference a resource (returns physical ID) or parameter |
| `Fn::GetAtt` | Access a resource attribute (e.g. `Arn`, `DNSName`) |
| `Fn::Join` | Concatenate strings |
| `Fn::Sub` | String substitution with `${Variable}` syntax |
| `Fn::If` / `Fn::Select` / `Fn::Split` | Conditionals and list operations |
| `Fn::ImportValue` | Import a value exported by another stack |
| `Fn::Cidr` | Generate a CIDR block from a range |

## Parameters

- Accept input at deploy time — types: `String`, `Number`, `List`, `AWS::EC2::KeyPair::KeyName`, `AWS::EC2::SecurityGroup::Id`, etc.
- **Default** values are optional; `NoEcho` hides sensitive values in console/API.
- **Constraints**: `AllowedValues`, `MinLength`, `MaxValue`, `ConstraintDescription` for validation.
- Parameters are the primary mechanism for **reusability** — one template, different values per environment.

## Mappings & conditions

- **Mappings**: static key-value lookups (e.g. `RegionMap: us-east-1 → ami-abc`). Use `Fn::FindInMap`.
- **Conditions**: `Fn::Equals`, `Fn::If`, `Fn::Not`, `Fn::And`, `Fn::Or` — control whether a resource is created.
  - Example: create a NAT Gateway only if `Environment = prod`.

## Change sets

- Preview what will happen **before** executing an update.
- Shows: resources to create, modify, delete, and any replacement (delete + recreate).
- Always review change sets for production stacks — CloudFormation can **replace** resources that don't support in-place updates (e.g. changing an EC2 instance type).

## Drift detection

- Compare the **actual** state of resources against the expected state in the stack.
- Detects manual changes made outside CloudFormation (console, CLI).
- Drift is **detection only** — you must decide to revert or update the template.
- Doesn't detect drift on every property — only supported resource types and properties.

## Stack policies

- JSON document that controls **which resources can be deleted or replaced** during an update.
- Protect critical resources (RDS, DynamoDB tables) from accidental replacement.
- Cannot be removed once applied (only updated).

## Termination protection

- Prevents accidental stack deletion — must explicitly disable before deleting.
- Enable on production stacks as a safeguard.

## DeletionPolicy & UpdateReplacePolicy

| Attribute | Behavior |
|---|---|
| `DeletionPolicy: Retain` | Resource survives stack deletion (e.g. RDS database, S3 bucket) |
| `DeletionPolicy: Snapshot` | Create a snapshot before deletion (EBS, RDS) |
| `DeletionPolicy: Delete` | Default — resource is deleted |
| `UpdateReplacePolicy: Retain` | On replacement, keep the old resource (don't delete it) |

## Nested stacks

- A stack can create **another stack** as a resource (`AWS::CloudFormation::Stack`).
- Child stack gets its own physical resource group.
- Use for: modular templates, shared infrastructure (VPC), team separation.
- Child stack deletion is controlled by parent stack.

## Cross-stack references

- Stack A **exports** a value (`Outputs → Export → Name`).
- Stack B imports it via `Fn::ImportValue`.
- CloudFormation tracks the dependency — Stack A cannot be deleted while Stack B imports from it.
- Use for: sharing VPC ID, subnet IDs, ALB ARNs across stacks.

## Stack sets

- Deploy the same template to **multiple accounts and/or regions** from a central admin account.
- **Execution role**: `AWSCloudFormationStackSetExecutionRole` in target accounts.
- **Admin role**: `AWSCloudFormationStackSetAdminRole` in the management account.
- **Concurrent accounts**: how many accounts to update simultaneously.
- **Operation preferences**: failure tolerance, max concurrent accounts, region order.
- Use for: enterprise-wide guardrails, logging stacks, security baselines.

## CloudFormation Init (`cfn-init`)

- Metadata helper that runs on **first boot** (via `AWS::CloudFormation::Init`).
- Installs packages, creates files, starts services — replaces complex user data scripts.
- **cfn-signal**: reports success/failure back to CloudFormation (prevents creation timeout).
- **cfn-hup**: daemon that watches for metadata changes and re-runs `cfn-init` on updates.
- Use with `CreationSignal` condition to wait for bootstrapping before marking resource CREATE_COMPLETE.

## Custom resources

- Extend CloudFormation with **anything not natively supported** — API calls, third-party APIs, data lookups.
- Backed by **Lambda function** or **SNS topic**.
- CloudFormation sends a `Create`, `Update`, or `Delete` event → your function handles it → returns success/failure.
- Use for: fetching latest AMI, rotating credentials, provisioning non-AWS resources.

## Service roles

- CloudFormation assumes an **IAM service role** to make API calls on your behalf.
- Separate from your personal/user credentials.
- Grant least-privilege permissions for the resources the template creates.
- Required for stack sets (admin role in management account, execution role in target accounts).

## SAM (Serverless Application Model)

- Extension of CloudFormation — simplified syntax for Lambda, API Gateway, DynamoDB, etc.
- `sam build` → `sam deploy` packages and uploads to S3, creates/updates a CloudFormation stack.
- Under the hood: SAM transforms into a standard CloudFormation template.
- Supports **local testing** (`sam local invoke`, `sam local start-api`).

## Pricing

CloudFormation itself is **free** — you pay only for the underlying AWS resources it creates and manages. No charge for templates, stacks, change sets, or drift detection.

## Exam domains

- [x] **Secure (30%)** — service roles, NoEcho parameters, stack policies protecting critical resources
- [x] **Resilient (26%)** — DeletionPolicy (Retain/Snapshot), nested stacks, stack sets for multi-account/region
- [x] **High-Performing (24%)** — cfn-init bootstrapping, custom resources for dynamic lookups
- [x] **Cost-Optimized (20%)** — parameters for environment reusability, conditions to skip dev-only resources

## Key gotchas

1. **Stack deletion deletes all resources** — use `DeletionPolicy: Retain` on critical resources (RDS, S3)
2. **CloudFormation can replace resources** on update — always review change sets first
3. **Drift detection is manual** — CloudFormation doesn't auto-detect drift; run it explicitly
4. **Stack sets need cross-account IAM roles** — won't work without `AWSCloudFormationStackSetExecutionRole`
5. **`Ref` returns physical ID**, `Fn::GetAtt` returns specific attributes — don't mix them up
6. **Nested stacks require templates in S3** — can't nest from local files
7. **`cfn-signal` must be called** or creation will time out waiting for success
8. **Custom resource responses must be signed** and come from a pre-signed S3 URL
9. **Outputs export names must be unique** across the region for cross-stack references
10. **Template max size**: 51,200 bytes (upload to S3 for larger)

## Related services

- [[CDK]] — write constructs in code; synthesizes to CloudFormation templates
- [[AWS-Batch]] — create compute environments via CloudFormation
- [[IAM]] — define roles, policies, users in templates
- [[VPC]] — define subnets, route tables, security groups as code
- [[Auto-scaling]] — launch templates, scaling policies via templates
- [[SSM]] — send commands to instances for bootstrapping alongside cfn-init
- [[S3]] — store templates, nested stack templates, deployment packages
- [[Lambda]] — custom resource backing functions
