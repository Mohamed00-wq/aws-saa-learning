# CDK  Cloud Development Kit

## What it is

AWS CDK is an open-source framework that defines cloud infrastructure using **real programming languages**  TypeScript, Python, Java, C#, Go  instead of YAML/JSON. Under the hood, CDK synthesizes your code into a CloudFormation template and deploys it as a stack. It doesn't replace CloudFormation  it wraps it.

## Languages

| Language | Package |
|---|---|
| TypeScript | `aws-cdk-lib` |
| Python | `aws-cdk-lib` |
| Java | `software.amazon.awscdk` |
| C# (.NET) | `Amazon.CDK.Lib` |
| Go | `github.com/aws/aws-cdk-go` |

## Constructs  the building blocks

| Level | What it is | Example |
|---|---|---|
| **L1 (CfnResource)** | 1:1 mapping to CloudFormation resource | `CfnBucket`, `CfnFunction` |
| **L2 (Resource)** | Opinionated wrapper with defaults and helpers | `Bucket`, `Function`, `Table` |
| **L3 (Pattern)** | Pre-assembled multi-resource patterns | `RestApi`, `ApplicationLoadBalancedFargateService` |

- **App** → **Stack** → **Construct** hierarchy. Stack maps 1:1 to a CloudFormation stack.

## CDK vs CloudFormation

| | CDK | CloudFormation |
|---|---|---|
| Language | TypeScript, Python, Java, C#, Go | YAML or JSON only |
| Abstractions | L1/L2/L3 constructs | Intrinsic functions |
| Type safety | Yes (IDE catches errors at write time) | No (errors at deploy time) |
| Reusability | Classes, libraries, packages | Nested stacks, modules |
| Underlying engine | CloudFormation | CloudFormation |

## App & stack lifecycle

1. `cdk synth`  compile → CloudFormation template (in `cdk.out/`).
2. `cdk deploy`  upload template → create/update stack.
3. `cdk diff`  compare current template vs deployed state.
4. `cdk destroy`  delete the stack and all resources.

## CDK Pipelines

- **Self-mutating CI/CD pipeline** built on **CodePipeline**. Define the pipeline as code  it updates itself on push.
- Stages: Source → Build → UpdatePipeline → Deploy (multiple accounts/regions).

## Context & environments

- **Environments**: stack deployed to specific `account` + `region` combination.
- **Context** (`cdk.context.json`): cached lookup values (VPC ID, AMI). `cdk context --clear` forces re-lookup.
- **SSM Parameter Store**: read values at synth time via `StringParameter.valueFromLookup`.

## Testing

- **Snapshot tests**: `cdk synth` → compare template against saved snapshot.
- **Assertions**: `Template.fromStack(stack)` → assert resource count and properties.

## `cdk bootstrap`

- One-time setup per account/region  creates S3 bucket (staging assets) + ECR repo (container images).
- Creates IAM roles: `CDKDeployRole`, `CDKFilePublishingRole`, `CDKImagePublishingRole`.

## Pricing

CDK itself is **free**  you pay only for the AWS resources it creates.

## Exam domains

- [x] **Secure (30%)**  grant methods for least privilege, context caching, environment separation
- [x] **Resilient (26%)**  CDK Pipelines for multi-account/region deployment, snapshot testing
- [x] **High-Performing (24%)**  L3 constructs for rapid development, asset bundling
- [x] **Cost-Optimized (20%)**  reusable constructs across environments, skip dev resources

## Key gotchas

1. **`cdk bootstrap` is required** in each account/region before first deploy
2. **Context values are cached**  `cdk.context.json` can go stale
3. **CDK uses CloudFormation**  every CF limitation is also a CDK limitation (500 resources, template size)
4. **`grant*` methods** auto-generate IAM policies  don't write them by hand
5. **Cross-stack references create CloudFormation exports**  can't delete exporting stack while imports exist
6. **Asset bundling uses Docker by default**  make sure Docker is running
7. **CDK Pipelines are self-mutating**  the pipeline updates itself this is by design

## Related services

- **CloudFormation**  CDK's underlying engine synthesizes to CF templates
- **SSM**  parameter store for context lookups
- **S3**  asset staging bucket created by bootstrap
