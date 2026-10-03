# IAM Policy Logic

```text
Trust Policy
    ↓
WHO can assume the Role?

Identity Policy
    ↓
WHAT can the User/Role do?

Resource Policy
    ↓
WHO can access the Resource?

Permissions Boundary
    ↓
WHAT is the maximum permission?

SCP
    ↓
WHAT is allowed at the Organization/Account level?

Session Policy
    ↓
WHAT is allowed in this temporary session?
```

## Common IAM Role Patterns

```text
EC2 Role
    ↓
EC2 assumes the Role
    ↓
Temporary Credentials
    ↓
Access AWS Resources
```

```text
Lambda Execution Role
    ↓
Lambda assumes the Role
    ↓
Temporary Credentials
    ↓
Access AWS Resources
```

```text
ReadOnly Role
    ↓
User/Service assumes the Role
    ↓
Read-only permissions
    ↓
Describe / Get / List
```

```text
Cross-Account Role
    ↓
Principal from Account A
    ↓
Assume Role in Account B
    ↓
Temporary Credentials
    ↓
Access permitted resources in Account B
```

## Role Logic

```text
                 Trust Policy
                      ↓
              WHO can assume?
                      ↓
                  IAM Role
                      ↓
              Identity Policy
                      ↓
               WHAT can it do?
                      ↓
              AWS Resources
```

## Key Principle

> **Trust determines who can assume the Role.**
>
> **Permissions determine what the Role can do.**
>
> **Boundaries and SCPs can limit the maximum effective permissions.**
