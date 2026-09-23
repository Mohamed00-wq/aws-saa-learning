# AWS Service Catalog — Approved Service Catalogs

## What it is

AWS Service Catalog lets organizations create and manage **curated catalogs of approved AWS services** for end users. Developers launch pre-approved infrastructure without needing full AWS access. Administrators define what's available, users pick from a menu. For the SAA exam, it's the answer for governing and standardizing resource provisioning, restricting what users can create, and enforcing tagging.

## Core components

| Component | What it is |
|---|---|
| **Product** | A deployable item backed by a CloudFormation template |
| **Portfolio** | A collection of related products |
| **Constraint** | Rules applied to a product (launch, notification, tag update, template) |
| **TagOption** | Predefined tags applied automatically on launch |

## How it works

1. Admin creates a **Product** (upload CloudFormation template).
2. Admin adds it to a **Portfolio** and sets **Constraints**.
3. Admin shares the portfolio with **IAM users/roles** or **AWS Organizations**.
4. End user browses, selects, fills parameters, and launches — CloudFormation provisions resources.

## Constraints

| Constraint | What it does |
|---|---|
| **Launch** | Assigns an **IAM role** — users don't need direct IAM permissions |
| **Notification** | Sends SNS notifications on launch/update |
| **Tag update** | Controls whether users can modify tags post-launch |
| **Template** | Restricts which parameter values users can provide |

- **Launch constraint is the key governance feature** — users can launch EC2 without EC2 permissions.

## TagOptions

- Library of predefined tag key-value pairs. Attached to products/portfolios, applied on launch.
- Users select from allowed values — can't invent new tags.

## Sharing

| Method | Scope |
|---|---|
| **IAM sharing** | Specific users, roles, or groups |
| **Organizations sharing** | All accounts in the organization |
| **Account sharing** | Specific AWS accounts (cross-account) |

- Shared portfolios appear as **imported**. Organizations sharing is the most scalable.

## CloudFormation + Terraform

- **CloudFormation**: native support — upload directly. **Terraform**: wrap in CloudFormation template or Lambda-backed custom resource.
- Service Catalog is a **governance layer** on top of any IaC tool.

## Pricing

AWS Service Catalog is **free** — pay only for underlying resources provisioned.

## Exam domains

- [x] **Secure (30%)** — launch constraints, template constraints, tag governance
- [x] **Resilient (26%)** — Organizations sharing, CloudFormation-backed products
- [x] **High-Performing (24%)** — self-service provisioning, reduce ticket-based bottlenecks
- [x] **Cost-Optimized (20%)** — TagOptions for cost allocation, restrict expensive types

## Key gotchas

1. **Launch constraints mean users don't need IAM permissions** — the launch role does the work
2. **Service Catalog doesn't manage resources after launch** — CloudFormation does
3. **Shared portfolios are imported, not copied** — changes propagate to all accounts
4. **End users can't see the template** — only product description and parameters
5. **TagOptions are not enforced after launch** — use AWS Config for ongoing compliance
6. **Template constraints restrict parameter values**, not resources
7. **Service Catalog is region-specific** — share across regions separately

## Related services

- **CloudFormation** — products are backed by CF templates
- **IAM** — launch constraints, portfolio sharing permissions
- **AWS-Config** — enforce ongoing tag compliance post-launch
- **AWS-Organizations** — share portfolios across all accounts
