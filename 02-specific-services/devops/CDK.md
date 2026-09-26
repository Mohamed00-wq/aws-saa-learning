# AWS CDK  Cloud Development Kit

## Purpose

AWS CDK is an **open-source IaC framework** that defines AWS infrastructure in **real programming languages** (TypeScript, Python, Java, C#, Go, JavaScript) instead of YAML/JSON. It compiles ("synthesizes") that code into **CloudFormation templates** and deploys them as stacks, so it **wraps CloudFormation rather than replacing it**. For the SAA exam it's the modern answer for **code-defined, testable, reusable infrastructure**, especially in CI/CD pipelines.

## Main use cases

- **Infrastructure built like application code**  classes, loops, conditionals, unit-testable constructs
- **Rapid multi-service stacks** with **L3 patterns** (e.g. `ApplicationLoadBalancedFargateService` provisions ALB + ECS + security groups in one construct)
- **CI/CD for infrastructure** with self-mutating **CDK Pipelines** (CodePipeline-based, multi-account/multi-region)
- **Reusable libraries** shared across teams/environments (constructs as packages)
- **Asset deployment** (bundling Lambda code, Docker images) with automatic S3/ECR staging via **`cdk bootstrap`**
- **Testing infrastructure**: snapshot tests (`cdk synth` vs saved template) and assertion tests (`Template.fromStack`)

## Key features

- **Three construct levels**  **L1 / CfnResource** (1:1 with CF resources), **L2** (opinionated wrappers with sane defaults, e.g. `Bucket`, `Function`, `Table`), **L3** (pre-assembled multi-resource patterns)
- **App -> Stack -> Construct hierarchy** (Stack = 1 CF stack)
- **Workflow**  `cdk synth` (compile to template in `cdk.out/`) -> `cdk diff` (compare deployed state) -> `cdk deploy` -> `cdk destroy`
- **`cdk bootstrap`** one-time per account/region (staging S3 bucket + ECR repo + IAM roles `CDKDeployRole`, file/image publishing roles)
- **Environment & context**: deploy to specific `account`/`region` `cdk.context.json` caches lookups (VPC ID, AMI) via `StringParameter.valueFromLookup` (`cdk context --clear` to refresh)
- **`grant*` methods** auto-generate least-privilege IAM policies so you don't hand-write them
- **CDK Pipelines** are **self-mutating** (the pipeline re-deploys itself on every commit)
- **Pricing**: CDK itself is **free** (you pay only for resources it creates)

## When to use

- Teams that **already write application code** and want type safety + IDE feedback for infrastructure
- **Complex, conditional, or large** infrastructure that's painful in plain YAML
- **Reusable, versioned infrastructure libraries** across many projects
- CI/CD-ified infrastructure deployment (**CDK Pipelines**) across dev/staging/prod accounts
- **Default modern choice for "code-based IaC"** still reviewed as CloudFormation under the hood

## Important limitation

- **CDK is no faster than CloudFormation at runtime**  it inherits every CF constraint (500-resource stacks, template size limits, update/replacement semantics). Requires **`cdk bootstrap`** before first deploy per account/region, **Docker** for asset bundling by default, and **Node runtime** in the pipeline. Context values go stale (must refresh). Cross-stack refs become CF exports that block deletion while imported. More "magic" than plain CF, so generated templates are harder to eyeball.

## SAA relevance

- "**Infrastructure as code with a programming language / type safety**" -> **CDK** (vs YAML/JSON CloudFormation)
- "**Testable, reusable infrastructure and CI/CD infrastructure pipelines**" -> CDK + **CDK Pipelines**
- "**Automatically generated least-privilege IAM for resources**" -> `grant*` methods
- "**2026-era IaC pattern**" -> CDK (Gen 2 apps, watch/ciphers, stack synthesizers), still built on **CloudFormation**
- Exam trick: **CDK = CloudFormation underneath**, both **free** themselves, and `cdk bootstrap` assets live in **S3/ECR**.