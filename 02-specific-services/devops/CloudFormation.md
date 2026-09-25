# CloudFormation  Infrastructure as Code

## What it is

CloudFormation is AWS's native IaC service. Define resources in YAML/JSON templates, provision them as **stacks**. Change the template, update the stack  CloudFormation figures out what to create, modify, or delete. For the SAA exam, it appears in any scenario about repeatable deployments, drift detection, cross-account infrastructure, or governance.

## Templates & stacks

- **Template**: YAML/JSON describing desired state  resources, parameters, outputs, conditions.
- **Stack**: a live instance of a template. Deleting a stack deletes all resources it created (unless protected).
- **Stack set**: deploy the same template across **multiple accounts and regions** from a single admin account.

## Template anatomy

| Section | Purpose |
|---|---|
| `Parameters` | Input values at deploy time (instance type, CIDR) |
| `Mappings` | Static lookup tables (region → AMI ID) |
| `Conditions` | Create/skip resources based on parameter values |
| `Resources` | **Required**  the only mandatory section |
| `Outputs` | Export values (ARNs, IDs) for cross-stack references |

### Key intrinsic functions

`Ref` (resource physical ID / parameter), `Fn::GetAtt` (resource attribute), `Fn::Sub` (string substitution), `Fn::Join`, `Fn::If`, `Fn::ImportValue` (import from another stack), `Fn::Cidr`.

## Change sets & drift detection

- **Change sets**: preview what will happen before executing an update  shows creates, modifies, deletes, replacements.
- **Drift detection**: compare actual resource state against template. Detection only  you decide to revert or update.

## Stack policies & termination protection

- **Stack policy**: JSON document controlling which resources can be deleted/replaced during updates. Protects RDS, DynamoDB.
- **Termination protection**: prevents accidental stack deletion  must explicitly disable before deleting.

## DeletionPolicy & UpdateReplacePolicy

| Attribute | Behavior |
|---|---|
| `DeletionPolicy: Retain` | Resource survives stack deletion |
| `DeletionPolicy: Snapshot` | Create snapshot before deletion (EBS, RDS) |
| `UpdateReplacePolicy: Retain` | On replacement, keep the old resource |

## Nested stacks & cross-stack references

- **Nested stacks**: a stack creating another stack (`AWS::CloudFormation::Stack`). Use for modular templates.
- **Cross-stack references**: Stack A exports → Stack B imports via `Fn::ImportValue`. Stack A can't be deleted while imports exist.

## Stack sets

- Deploy to **multiple accounts and/or regions** from a central admin account.
- Requires **Admin role** (management account) and **Execution role** (target accounts).
- Use for: enterprise-wide guardrails, logging stacks, security baselines.

## CloudFormation Init (`cfn-init`)

- Metadata helper that runs on **first boot**  installs packages, creates files, starts services.
- **cfn-signal**: reports success/failure back to CloudFormation (prevents creation timeout).
- **cfn-hup**: daemon watching for metadata changes, re-runs `cfn-init` on updates.

## Custom resources & SAM

- **Custom resources**: extend CloudFormation with anything not natively supported  backed by Lambda or SNS.
- **SAM**: simplified syntax for Lambda/API Gateway/DynamoDB. Under the hood, transforms to CloudFormation.

## Pricing

CloudFormation itself is **free**  you pay only for the underlying AWS resources it creates.

## Exam domains

- [x] **Secure (30%)**  service roles, NoEcho parameters, stack policies
- [x] **Resilient (26%)**  DeletionPolicy (Retain/Snapshot), nested stacks, stack sets
- [x] **High-Performing (24%)**  cfn-init bootstrapping, custom resources
- [x] **Cost-Optimized (20%)**  parameters for reuse, conditions to skip dev-only resources

## Key gotchas

1. **Stack deletion deletes all resources**  use `DeletionPolicy: Retain`
2. **CloudFormation can replace resources**  review change sets first
3. **Drift detection is manual**  not auto-detected
4. **Stack sets need cross-account IAM roles**
5. **Nested stacks require templates in S3**
6. **`cfn-signal` must be called** or creation times out
7. **Template max size**: 51,200 bytes

## Related services

- **CDK**  synthesizes to CloudFormation templates
