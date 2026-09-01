# (IAM) Identity and Access Management

## What it is
IAM (Identity and Access Management) controls WHO can access WHAT 
in your AWS account. It's a global service (not region-specific) 
that lets you manage authentication (who you are) and authorization 
(what you're allowed to do) across all AWS resources.

## Key concepts

### Core entities
1. **Root user** - created when you set up the AWS account, has
   full unrestricted access. Best practice: never use for daily
   tasks, enable MFA, lock away credentials
2. **Users** - individual identities (people or applications) with
   long-term credentials (password for console, access keys for CLI/API)
3. **Groups** - collections of users; permissions attached to a
   group apply to all members (cannot nest groups)
4. **Roles** - temporary identities assumed by users, services, or
   applications (no long-term credentials, uses temporary tokens
   via STS). Used for: EC2 instances needing AWS access, cross-account
   access, federated users (Google/SAML login)

### Policies
- **JSON documents** that define permissions
- **Identity-based policies**: attached to users/groups/roles
- **Resource-based policies**: attached to resources (e.g. S3
  bucket policy), define who can access THAT resource
- **Managed policies**: AWS-managed (predefined) or Customer-managed
  (you create/control)
- **Inline policies**: embedded directly in a single user/group/role
  (1-to-1, not reusable)

### Policy structure (key elements)
- **Effect**: Allow or Deny
- **Action**: which API calls (e.g. s3:GetObject)
- **Resource**: which AWS resource(s) the policy applies to (ARN)
- **Condition**: optional extra constraints (e.g. IP range, MFA required)

### Evaluation logic
- Explicit **Deny** always wins over Allow, no matter where it comes from
- By default, everything is denied unless explicitly allowed
- Final decision = union of all applicable policies, with any
  explicit Deny short-circuiting to Deny

### Security best practices
- Enable **MFA** (Multi-Factor Authentication) on root and all users
- Follow **least privilege principle**: grant only what's needed
- Use **IAM roles** instead of sharing/hardcoding long-term credentials
- Use **IAM Access Analyzer** to identify unused/overly broad permissions
- Rotate access keys regularly; use Policy Simulator to test permissions

### IAM Identity Center (formerly AWS SSO)
- Centralized access management across multiple AWS accounts
  (used with AWS Organizations)

## Key commands (AWS CLI)

# Create a user
aws iam create-user --user-name dev-user

# Create a group and add user to it
aws iam create-group --group-name Developers
aws iam add-user-to-group --user-name dev-user --group-name Developers

# Attach a managed policy to a group
aws iam attach-group-policy \
  --group-name Developers \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create a role (trust policy defines who can assume it)
aws iam create-role \
  --role-name EC2-S3-Access \
  --assume-role-policy-document file://trust-policy.json

# Attach policy to the role
aws iam attach-role-policy \
  --role-name EC2-S3-Access \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile and add role (needed to attach role to EC2)
aws iam create-instance-profile --instance-profile-name EC2-S3-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name EC2-S3-Profile \
  --role-name EC2-S3-Access

# List users' access keys (for rotation audits)
aws iam list-access-keys --user-name dev-user

## How it works
1. A user/service makes a request to AWS (e.g. "list S3 buckets")
2. IAM authenticates the identity (password, access key, or
   temporary token from an assumed role)
3. IAM evaluates all applicable policies (identity-based +
   resource-based + any SCPs from Organizations)
4. If any explicit Deny applies → request denied
5. Else if at least one Allow applies → request permitted
6. Else (no explicit Allow) → denied by default
7. For EC2 needing AWS access: instance is launched with an
   Instance Profile containing a Role → app on the instance gets
   temporary credentials automatically via instance metadata
   (no hardcoded keys needed)

## Exam domain(s) checklist
- [ ] Design Secure Architectures (30%) — this is THE core service
      for this domain: least privilege, roles vs users, policy
      evaluation logic, cross-account access
- [ ] Design Resilient Architectures (26%) — roles for service-to-
      service access without credential management overhead

## Lab notes
(يتوضاف بعد ما تكمل الـ hands-on)

## Exam-style questions

**Q1:** An EC2 instance needs to read objects from an S3 bucket. 
What is the AWS-recommended way to grant this access?
<details><summary>Answer</summary>Attach an IAM Role (via an 
Instance Profile) to the EC2 instance with an S3 read policy — 
never hardcode access keys on the instance.</details>

**Q2:** A user has an identity-based policy that ALLOWS s3:GetObject 
on a bucket, but the bucket's resource-based policy explicitly 
DENIES access to that same user. What is the outcome?
<details><summary>Answer</summary>Access is denied. An explicit 
Deny in ANY applicable policy always overrides any Allow, 
regardless of where the Allow comes from.</details>

**Q3:** True or False: IAM Groups can contain other IAM Groups.
<details><summary>Answer</summary>False - IAM Groups cannot be 
nested; a group can only contain users, not other groups.</details>

## Related services
[[EC2]] [[S3]] [[STS]] [[Organizations]] [[KMS]]