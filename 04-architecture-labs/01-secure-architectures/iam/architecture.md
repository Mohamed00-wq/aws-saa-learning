# IAM Roles Architecture

## 1. IAM Role Access Flow

```text
                    ┌─────────────────┐
                    │    Principal    │
                    │                 │
                    │ User / EC2 /    │
                    │ Lambda / AWS    │
                    │ Account        │
                    └────────┬────────┘
                             │
                             │ sts:AssumeRole
                             ▼
                    ┌─────────────────┐
                    │   Trust Policy  │
                    │                 │
                    │ WHO can assume  │
                    │ the Role?       │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    IAM Role     │
                    │                 │
                    │ Temporary       │
                    │ Credentials     │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ Identity Policy │
                    │                 │
                    │ WHAT can the    │
                    │ Role do?        │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │ AWS Resource    │
                    │                 │
                    │ S3 / DynamoDB / │
                    │ EC2 / etc.      │
                    └─────────────────┘
```

## 2. Permission Evaluation

Effective permissions can be restricted by additional policy mechanisms:

```text
                     IAM Role
                        │
                        ▼
                Identity Policy
                        │
             ┌──────────┴──────────┐
             │                     │
             ▼                     ▼
   Permissions Boundary       Session Policy
             │                     │
             └──────────┬──────────┘
                        │
                        ▼
                 Effective Access
                        │
                        ▼
                  AWS Resource
```

At the organization level:

```text
AWS Organization
       │
       ▼
      SCP
       │
       ▼
AWS Account
       │
       ▼
 IAM Identity / Role
       │
       ▼
 AWS Resource
```

## 3. EC2 Role Architecture

```text
┌──────────────┐
│     EC2      │
└──────┬───────┘
       │
       │ AssumeRole
       ▼
┌────────────────────┐
│ EC2 IAM Role       │
│                    │
│ Trust Policy       │
│ Permission Policy  │
└──────────┬─────────┘
           │
           │ Temporary Credentials
           ▼
      ┌──────────┐
      │    S3    │
      └──────────┘
```

**Purpose:** Allow EC2 to access AWS resources without storing long-term access keys on the instance.

## 4. Lambda Execution Role

```text
┌──────────────┐
│    Lambda    │
└──────┬───────┘
       │
       │ AssumeRole
       ▼
┌────────────────────────┐
│ Lambda Execution Role  │
│                        │
│ Trust: Lambda          │
│ Permissions:           │
│ - CloudWatch Logs      │
│ - S3                   │
│ - DynamoDB             │
└───────────┬────────────┘
            │
            ▼
     AWS Resources
```

**Purpose:** Give a Lambda function the permissions required during execution.

## 5. Cross-Account Role

```text
┌─────────────────┐
│    Account A    │
│                 │
│ User / Role     │
└────────┬────────┘
         │
         │ AssumeRole
         ▼
┌─────────────────┐
│    Account B    │
│                 │
│ IAM Role        │
│                 │
│ Trust Policy    │
└────────┬────────┘
         │
         ▼
   AWS Resources
```

**Purpose:** Allow controlled access between AWS accounts without creating long-term credentials in the target account.

## 6. ReadOnly Role

```text
User
 │
 │ AssumeRole
 ▼
ReadOnly Role
 │
 ├── s3:Get*
 ├── s3:List*
 ├── ec2:Describe*
 └── rds:Describe*
```

The role can inspect resources while its permissions do not allow normal write operations.

## 7. Key Architecture Rules

```text
Trust Policy
    ↓
WHO can enter?

Identity Policy
    ↓
WHAT can the identity do?

Resource Policy
    ↓
WHO can access the resource?

Permissions Boundary
    ↓
WHAT is the maximum permission?

SCP
    ↓
WHAT is allowed by the Organization/Account?

Session Policy
    ↓
WHAT is allowed during this temporary session?
```

## 8. Design Principle

Use IAM Roles and temporary credentials whenever possible instead of embedding long-term AWS access keys in applications, servers, or source code.

Apply **Least Privilege** by granting only the permissions required for the workload.
