# CDK — Cloud Development Kit

## What it is

The AWS Cloud Development Kit is an open-source framework that lets you define cloud infrastructure using **real programming languages** — TypeScript, Python, Java, C#, and Go — instead of YAML/JSON templates. Under the hood, CDK synthesizes your code into a CloudFormation template and deploys it as a stack. You get the full power of a programming language (loops, conditionals, abstractions, type safety, IDE completion) while CloudFormation handles the actual resource orchestration.

CDK is the modern way to build IaC on AWS. It doesn't replace CloudFormation — it wraps it. Every CDK app produces a CloudFormation template, and every CDK deployment creates/updates a CloudFormation stack. For the SAA exam, know what CDK is, how constructs work, how it relates to CloudFormation, and when it's the right choice.

## Languages

| Language | Package |
|---|---|
| TypeScript | `aws-cdk-lib` |
| Python | `aws-cdk-lib` |
| Java | `software.amazon.awscdk` |
| C# (.NET) | `Amazon.CDK.Lib` |
| Go | `github.com/aws/aws-cdk-go` |

TypeScript and Python are the most commonly used. TypeScript has the best construct library support.

## Constructs — the building blocks

CDK organizes infrastructure into **constructs** — composable building blocks that represent one or more AWS resources.

### Construct levels

| Level | What it is | Example |
|---|---|---|
| **L1 (CfnResource)** | 1:1 mapping to a CloudFormation resource | `CfnBucket`, `CfnFunction` |
| **L2 (Resource)** | Higher-level, opinionated wrapper with sensible defaults and helpers | `Bucket`, `Function`, `Table` |
| **L3 (Pattern)** | Pre-assembled multi-resource patterns | `RestApi`, `QueueDlq`, `ApplicationLoadBalancedFargateService` |

- L1 = raw CloudFormation properties in code (verbose, but covers everything).
- L2 = AWS best practices baked in (permissions, encryption, naming), most commonly used.
- L3 = complete architectures in one line (API Gateway + Lambda + DynamoDB wired together).

### Construct tree

```
App
 └── Stack
      └── Construct (VPC)
           └── Subnet
           └── Subnet
```

- **App**: root of the CDK app, holds one or more stacks.
- **Stack**: maps 1:1 to a CloudFormation stack. Deploy unit.
- **Construct**: reusable building block inside a stack. Can contain other constructs.

## App & stack lifecycle

1. `cdk synth` — compile your code → produce a CloudFormation template (in `cdk.out/`).
2. `cdk deploy` — upload the template to CloudFormation → create/update the stack.
3. `cdk diff` — compare current template against the deployed stack.
4. `cdk destroy` — delete the stack and all its resources.

## CDK vs CloudFormation

| | CDK | CloudFormation (YAML/JSON) |
|---|---|---|
| Language | TypeScript, Python, Java, C#, Go | YAML or JSON only |
| Abstractions | L1/L2/L3 constructs | Intrinsic functions, mappings |
| Reusability | Classes, libraries, packages | Nested stacks, modules |
| Type safety | Yes (IDE catches errors at write time) | No (errors at deploy time) |
| Boilerplate | Low (L2/L3 constructs) | High (verbose YAML) |
| Underlying engine | CloudFormation | CloudFormation |
| Learning curve | Programming knowledge needed | YAML/JSON, simpler mental model |

## CDK Pipelines

- **Self-mutating CI/CD pipeline** for deploying CDK apps — built on **CodePipeline**.
- Define the pipeline **as code** in CDK — it updates itself when you push changes.
- Stages: Source → Build → UpdatePipeline → Deploy (multiple stages/accounts/regions).
- Built-in support for: CodeBuild, GitHub, ECR, manual approvals.
- Use for: automated multi-account, multi-region deployments with rollback support.

## Environment variables & context

- **Environments**: stack is deployed to a specific `account` + `region` combination.
  - Can be hardcoded, or use `CDK_DEFAULT_ACCOUNT` / `CDK_DEFAULT_REGION` environment variables.
- **Context** (`cdk.context.json`): key-value pairs for lookups (e.g. VPC ID, latest AMI).
- `context.json` values are cached — `cdk context --clear` forces re-lookup.
- **SSM Parameter Store**: read values at synth time via `StringParameter.valueFromLookup`.

## Default VPC

- `Vpc.fromLookup(stack, 'Vpc', { ... })` — looks up an existing VPC in your account.
- CDK **caches** the result in `cdk.context.json` (don't commit secrets here).
- For new VPCs: `new Vpc(stack, 'Vpc', { ... })` — creates with public/private subnets across AZs.

## Asset bundling

- CDK packages local files (code, configs, layers) as **assets** → uploads to S3 → referenced by CloudFormation.
- For Lambda: bundles your function code and dependencies automatically.
- For ECS: builds Docker images, pushes to ECR, updates task definitions.
- `BundlingOptions`: Docker-based bundling for non-JS runtimes (Python with native deps, Go binaries).

## Stacks & apps

- **Single-stack app**: one stack, one `cdk deploy`, simple.
- **Multi-stack app**: multiple stacks in one app — deploy independently or in dependency order.
  - Stack B imports from Stack A → CDK knows the order.
- **Cross-stack references**: use L2 construct imports/exports — CDK handles `Outputs`/`Fn::ImportValue` automatically.

## Testing

- **Snapshot tests**: `cdk synth` → compare template against saved snapshot → catch unintended changes.
- **Assertions**: `Template.fromStack(stack)` → assert resource count, property values.
  - `template.hasResourceProperties('AWS::Lambda::Function', { Runtime: 'python3.12' })`
- **Integration tests**: deploy to a test account → validate live resources → destroy.

## CDK CLI commands

| Command | What it does |
|---|---|
| `cdk init` | Scaffold a new CDK project (language, gitignore, dependencies) |
| `cdk synth` | Synthesize the CloudFormation template |
| `cdk diff` | Show what will change vs. deployed state |
| `cdk deploy` | Deploy the stack(s) |
| `cdk destroy` | Delete the stack(s) |
| `cdk bootstrap` | Provision S3 + ECR resources for CDK deployments in an account/region |
| `cdk list` | List all stacks in the app |
| `cdk context` | View/clear cached context values |

## `cdk bootstrap`

- One-time setup per account/region — creates an S3 bucket (for staging assets) and an ECR repository (for container images).
- Required before `cdk deploy` in a new account/region.
- Creates IAM roles: `CDKDeployRole`, `CDKFilePublishingRole`, `CDKImagePublishingRole`.
- **Trusted accounts**: allow other accounts to deploy into this one (for cross-account pipelines).

## L2 construct examples

```typescript
// S3 bucket with encryption, versioning, lifecycle rules
const bucket = new s3.Bucket(this, 'MyBucket', {
  encryption: s3.BucketEncryption.S3_MANAGED,
  versioned: true,
  lifecycleRules: [{ expiration: Duration.days(90) }],
});

// Lambda function (CDK bundles code, creates role, sets permissions)
const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.PYTHON_3_12,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda'),
});

// Grant the Lambda read access to the bucket (automatically adds IAM policy)
bucket.grantRead(fn);
```

## Pricing

CDK itself is **free** — you pay only for the AWS resources it creates. There's no charge for the CDK CLI, synthesis, or the constructs.

## Exam domains

- [x] **Secure (30%)** — CDK roles (least privilege via grant methods), context caching for lookups, environment separation
- [x] **Resilient (26%)** — CDK Pipelines for automated multi-account/region deployment, snapshot testing for drift prevention
- [x] **High-Performing (24%)** — L3 constructs for rapid development, asset bundling, construct libraries for consistency
- [x] **Cost-Optimized (20%)** — conditions and context to skip dev resources, reusable constructs across environments

## Key gotchas

1. **`cdk bootstrap` is required** in each account/region before first deploy — forget it and deploy fails
2. **Context values are cached** — `cdk.context.json` can stale; clear with `cdk context --clear`
3. **CDK uses CloudFormation under the hood** — every CDK limitation is also a CDK limitation (500 resource limit, template size, etc.)
4. **`grant*` methods are your friend** — don't write IAM policies by hand when L2 constructs offer `grantRead`, `grantWrite`, etc.
5. **Snapshot tests catch surprises** — run `cdk synth` + snapshot tests before every PR
6. **Cross-stack references create CloudFormation exports** — can't delete the exporting stack while imports exist
7. **Asset bundling uses Docker by default** — make sure Docker is running locally for non-JS lambdas
8. **CDK Pipelines are self-mutating** — the pipeline updates itself on deploy; this is by design, not a bug
9. **L1 constructs = raw CloudFormation** — use them when L2 doesn't exist yet, but expect more verbose code
10. **Don't commit `cdk.context.json` secrets** — use SSM Parameter Store for sensitive lookups

## Related services

- [[CloudFormation]] — CDK's underlying engine; CDK synthesizes to CloudFormation templates
- [[SSM]] — parameter store for CDK context lookups, deployment orchestration
- [[S3]] — asset staging bucket created by bootstrap, template storage
- [[CodePipeline]] — CDK Pipelines builds on CodePipeline for CI/CD
- [[Lambda]] — most common L3 pattern: auto-bundled Lambda functions
- [[IAM]] — grant methods auto-generate IAM policies
- [[ECR]] — CDK bootstrap creates ECR repo for Docker image assets
