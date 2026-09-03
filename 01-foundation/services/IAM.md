# IAM — Identity and Access Management

## What it is

IAM is the backbone of every security decision in AWS. It is a **global** service (not region-scoped) that governs two fundamental questions: **who** is making a request (authentication) and **what** they are allowed to do (authorization). Every single AWS API call whether from the console, CLI, SDK, or an internal service-to-service call — passes through IAM evaluation before anything happens.

IAM does not govern data-plane traffic directly (e.g. an EC2 instance reaching out to the internet) it governs control-plane and data-plane API permissions against AWS resources. Understanding IAM is not optional it maps directly to the **Design Secure Architectures** domain which carries the highest weight on the SAA-C03 exam.

---

## Key concepts

### Identities (who can act)

**Root user**
- Created automatically when the AWS account is opened. It is the only identity that can never be deleted or restricted.
- Has **unrestricted access** to every resource and API in the account IAM cannot write a policy that denies the root user.
- Best practice: lock it away immediately. Use it only for tasks that specifically require root (e.g. closing the account, changing the account payment method, registering as an SSHCA certificate authority). Enable MFA on it the moment the account is created.
- Root user credentials are the email + password used to create the account not an access key.

**IAM Users**
- Long-lived identities with a permanent password (console) and/or up to two access key pairs (programmatic access).
- Each user gets its own login profile MFA device, and credential set.
- Users are **not** a best practice for applications or services use roles instead. 
Users exist for people or legacy systems that cannot assume roles.
- Access keys are not rotated automatically; you must manage rotation yourself.

**IAM Groups**
- A flat collection of users. Attaching a policy to a group grants those permissions to every member of the group.
- Groups **cannot be nested**  you cannot put one group inside another. 
This is a common exam trap.
- Groups exist only as a convenience for user management they are not identities that can assume a role or appear in a policy's principal.

**IAM Roles**
- The most important identity concept in AWS for the exam.
- A role is a **temporary identity** no long-term credentials exist. 
When assumed the role produces temporary credentials (via AWS STS) that expire after a set period (default 1 hour configurable up to 12 hours).
- A role is defined by two policies:
  1. **Trust policy**  defines WHO can assume this role (another AWS account an EC2 instance, a federated user, a service like Lambda).
  2. **Permission policy**  defines WHAT the assumed role can do once assumed.
- Roles are the AWS-recommended way to grant an EC2 instance access to S3, DynamoDB, etc.  never hardcode access keys on an instance.
- **Common role types:**
  - **Service roles**  assumed by an AWS service (e.g. EC2 instance profile role, Lambda execution role).
  - **Cross-account roles** assumed by principals in a different AWS account, enabling federated multi-account access.
  - **Federated roles** assumed by users authenticated externally (Google, Active Directory, SAML 2.0) and mapped to IAM via identity federation.
  - **Instance profiles**  a container that wraps a role so it can be attached to an EC2 instance. 
The instance fetches temporary credentials from the instance metadata service (IMDSv1 or IMDSv2) automatically.

### Policies (the rules)

Policies are JSON documents. Every policy has:
- **Version** — always `"2012-10-17"` (current version).
- **Statement** — one or more permission blocks, each containing:
  - **Effect** — `"Allow"` or `"Deny"` (explicit).
  - **Action** — which API calls this covers (e.g. `"s3:GetObject"`, `"ec2:StartInstances"`, or `"*"` for everything).
  - **Resource** — which ARN(s) the action applies to (e.g. `"arn:aws:s3:::my-bucket/*"`). Some actions are not resource-specific and use `"*"` here.
  - **Condition** *(optional)* — extra constraints such as requiring MFA, restricting by source IP, enforcing TLS, limiting by tag, or restricting time of day.

**Policy types:**
- **Identity-based policies**  attached to a user, group, or role. Defines what that identity can do.
- **Resource-based policies**  attached directly to a resource (S3 bucket policy, SQS queue policy, KMS key policy). Defines who can access that resource. Resource-based policies are unique because they can grant access to principals in **other AWS accounts** directly.
- **Permission boundaries**  a maximum permission ceiling set on a user or role. The effective permission is the **intersection** of the identity policy and the permission boundary. If the boundary says `"Allow s3:*"` but the identity policy says `"Allow ec2:*"`, neither is allowed  the intersection is empty.
- **Session policies**  a policy passed when a role is assumed (via `AssumeRole`), further restricting the session. The effective permission is the intersection of the role's identity policy and the session policy.
- **AWS Organizations SCPs (Service Control Policies)**  organization-wide permission boundaries applied at the OU or account level. An SCP does not grant permissions  it sets the maximum ceiling for what any identity in that account/OU can do. If an SCP denies `"s3:*"`, no one in that account (including root) can use S3, regardless of what their local policies say.
- **ACLs (Access Control Lists)**  an older, less common mechanism. Used by S3 (bucket ACL, object ACL) and VPC (network ACLs). Most workloads should use policies instead.

**Managed vs Inline:**
- **AWS-managed policies**  pre-built by AWS for common use cases (e.g. `AmazonS3ReadOnlyAccess`). They are versioned and updated by AWS; you cannot modify them.
- **Customer-managed policies** — policies you create, version, and control. Recommended for production use because you can review and audit them.
- **Inline policies**  embedded directly into a single user, group, or role. They exist in a 1-to-1 relationship and are not reusable. AWS discourages their use except for specific scenarios (e.g. a user that needs permissions unique to that user alone and nowhere else).

### Policy evaluation logic

This is one of the most important things to understand for the exam:

1. **Default deny**  by default, all requests are denied. Nothing is allowed unless an explicit Allow exists.
2. **Explicit Deny always wins**  if ANY applicable policy contains an explicit Deny for the request, the request is denied, period. It does not matter how many Allows exist elsewhere.
3. **Explicit Allow is needed**  after checking for Denys, there must be at least one explicit Allow for the request to be permitted.
4. **SCP boundary**  before evaluating identity or resource policies, the request is first checked against the account/OU SCPs. If the SCP does not include the action in its implicit deny (and does not explicitly deny it) evaluation continues.
5. **Permission boundary**  if a permission boundary is attached to the identity the final permission is the intersection of the identity policy and the boundary.
6. **Resource-based policy**  evaluated alongside identity policies. 
If either grants an Allow (and neither has a Deny) the action is permitted.
7. **Session policy**  when assuming a role with a session policy the result is the intersection of the role's policies and the session policy.

**The shorthand for exams:** Deny always wins → Default is deny → Allow must be explicit → SCPs set the outer ceiling → Permission boundaries and session policies restrict further.

---

## Architecture deep dive

### How IAM evaluation flows in practice

When an IAM-authenticated request hits an AWS service:
1. AWS authenticates the caller and determines their identity (user, role session, federated identity).
2. AWS collects all applicable policies: identity-based policies attached to the caller, any resource-based policies on the target resource, SCPs from the account hierarchy, permission boundaries, and any session policy.
3. **Deny evaluation first:** scan all applicable policies for any explicit Deny. If found → request denied immediately.
4. **Allow evaluation:** check if at least one policy contains an explicit Allow for the requested action and resource.
5. If an Allow is found AND no Deny exists → request permitted.
6. If no Allow is found → request denied (default deny).

### STS (Security Token Service) — the engine behind roles

When a role is assumed, STS is the service that actually issues the temporary credentials. The flow:
1. The caller (user, application, or AWS service) sends an `AssumeRole` request to STS.
2. STS evaluates the role's trust policy to confirm the caller is allowed to assume it.
3. STS evaluates the permission policies and applies any session policy (intersection logic).
4. STS returns temporary credentials: access key, secret key, and session token.
5. The caller uses these temporary credentials to make API calls. They expire after the configured duration.

STS is also used by:
- **Web Identity Federation** (Login with Google/Amazon → get temporary AWS credentials).
- **SAML 2.0 federation** (corporate Active Directory → STS → temporary AWS credentials).
- **Cross-account access** (assume a role in another account).
- **EC2 instance profiles** (the instance metadata service is a form of STS under the hood).

### Instance Profiles and the metadata service

When you attach a role to an EC2 instance via an instance profile:
1. The instance profile is a container that holds the role ARN.
2. The instance is configured to make credential requests to the **instance metadata service** at `169.254.169.254`.
3. The metadata service issues temporary STS credentials on behalf of the instance, automatically rotating them before expiration.
4. **IMDSv2** (recommended) requires a session token obtained via a PUT request, adding protection against SSRF-based credential theft that was possible with IMDSv1.
5. **Exam trap:** instance profile ≠ role. The instance profile is the wrapper; the role is the actual identity. But when people say "attach a role to an EC2 instance," they mean attaching an instance profile containing that role.

### IAM Access Analyzer

A service that identifies:
- **External access** — resources (S3 buckets, KMS keys, IAM roles, VPC endpoints, etc.) that are shared with external principals (other AWS accounts or the public).
- **Unused access** — IAM policies, roles, and credentials that have not been used in a configurable time period.
- It uses a provable security model based on Zelkova (a formal verification engine) to reason about which resources are accessible from outside the account.

### IAM Identity Center (formerly AWS SSO)

- Provides single sign-on to multiple AWS accounts, business applications (Salesforce, Office 365), and custom SAML 2.0 applications.
- Integrates with AWS Organizations to manage access centrally.
- Assigns permission sets (collections of policies) to users/groups per account.
- Works with external identity providers (Okta, Azure AD, etc.) via SAML.

---

## Exam domain(s)

- [x] **Design Secure Architectures (30%)** — this is the core service for the largest exam domain. Every security question likely touches IAM in some way.
- [ ] Design Resilient Architectures (26%) — roles for service-to-service access without credential management overhead.

---

## Advanced gotchas & edge cases

1. **Implicit deny is not an explicit Deny.** If a policy simply does not mention an action, that is an implicit deny. An explicit Deny is a statement with `"Effect": "Deny"`. The exam tests whether you understand this distinction.

2. **An explicit Deny in a resource-based policy overrides an Allow in the identity-based policy.** The evaluation logic applies equally regardless of where the Deny comes from — identity-based, resource-based, SCP, or permission boundary.

3. **SCPs do NOT grant permissions.** An SCP can only restrict. If an account has no local policy allowing `s3:GetObject`, adding an SCP that allows it still does nothing — the account-level policy must also allow it.

4. **Permission boundaries and session policies reduce permissions, never expand them.** Both work via intersection logic — they can only take away, never give.

5. **Groups cannot be nested.** The exam frequently uses a scenario where "a group contains another group" as a wrong answer. Groups only contain users.

6. **Root user cannot be restricted by IAM policies.** The only way to limit root is via SCPs and Organizations policies.

7. **IAM is global; IAM policies are evaluated against global and regional services.** An IAM policy granting `s3:GetObject` works for buckets in any region because S3 is a global service with regional endpoints.

8. **Cross-account access does not require creating users in both accounts.** You create a role in the target account with a trust policy allowing the source account, and users in the source account assume that role.

9. **Access keys are account-scoped, not user-scoped.** A user can have at most two active access key pairs at a time.

10. **IAM policy variables and wildcards are allowed.** `"Resource": "arn:aws:s3:::${aws:username}/*"` lets a user access only their own S3 prefix — a powerful exam concept.

---

## Exam-style questions

**Q1.** An EC2 instance needs to read objects from an S3 bucket. What is the AWS-recommended way to grant this access?
- A) Store access keys in a configuration file on the instance
- B) Attach an IAM Role via an Instance Profile to the instance with an S3 read policy
- C) Create an IAM user for the application and embed the credentials in the AMI
- D) Attach an S3 bucket policy that allows the instance's public IP

<details><summary>Answer</summary>
**B** — IAM Roles with Instance Profiles are the AWS-recommended pattern: no long-term credentials on the instance, automatic credential rotation by STS, and the least privilege principle is enforced. Storing access keys (A, C) violates security best practices and is explicitly called out by AWS. D assumes a static public IP, which is fragile and not how EC2-S3 access is designed.
</details>

**Q2.** A company has two AWS accounts: Account A (development) and Account B (production). Developers in Account A need read-only access to an S3 bucket in Account B. What is the simplest and most secure approach?
- A) Create IAM users in Account B for every developer in Account A
- B) Share an access key from Account B with Account A developers
- C) Create a role in Account B with a trust policy allowing Account A, and an S3 read permission policy
- D) Make the S3 bucket public

<details><summary>Answer</summary>
**C** — Cross-account roles are the AWS-recommended pattern. A role in Account B trusts Account A; developers in Account A assume the role and receive temporary credentials scoped to S3 read-only. This avoids provisioning users in multiple accounts, avoids long-lived credentials, and follows least privilege.
</details>

**Q3.** A developer creates an IAM policy granting full access to all S3 buckets. The account administrator has an SCP that denies all S3 access. What happens when the developer tries to list S3 buckets?
- A) The developer's policy allows it, so it succeeds
- B) The SCP has no effect on IAM users
- C) Access is denied — the SCP sets the maximum permission ceiling
- D) The SCP and the IAM policy cancel each other out

<details><summary>Answer</summary>
**C** — SCPs set the outer boundary. If the SCP denies `s3:*`, no identity in that account can use S3, regardless of what their local policies allow. The SCP does not need to be "stronger" — it operates at a different level and restricts what is even possible within the account.
</details>

**Q4.** True or False: An explicit Deny in a resource-based policy on an S3 bucket overrides an explicit Allow in the caller's identity-based IAM policy.
- A) True — resource-based policies always take precedence
- B) False — identity-based policies always take precedence
- C) True — explicit Deny in ANY policy overrides any Allow, regardless of source
- D) False — both are evaluated and the Allow wins

<details><summary>Answer</summary>
**C** — The statement is true but the reason in A is wrong. The correct answer is C: explicit Deny always wins, regardless of whether it comes from an identity-based policy, resource-based policy, SCP, or permission boundary. There is no "precedence" between policy types — a Deny from any source short-circuits to Deny.
</details>

**Q5.** An IAM permission boundary is set to allow only S3 and DynamoDB operations. A user has an identity-based policy allowing S3 and EC2 operations. What is the effective permission?
- A) S3 and DynamoDB (union of both)
- B) S3 only (intersection of both)
- C) S3, DynamoDB, and EC2 (all policies are additive)
- D) Deny all — the policies conflict

<details><summary>Answer</summary>
**B** — The effective permission is the intersection: S3 is allowed by both, DynamoDB is allowed only by the boundary, EC2 is allowed only by the identity policy. Since the boundary acts as a ceiling, only S3 survives the intersection. This is why permission boundaries are powerful for delegated administration — they limit what a developer-created role can do even if the developer grants broader permissions.
</details>

**Q6.** A Solutions Architect needs to ensure that all IAM access keys in the organization are rotated at least every 90 days. Which approach is most appropriate?
- A) Enable AWS Config rule to detect unrotated keys
- B) Manually rotate keys every 90 days
- C) Use IAM Access Analyzer to identify unused access
- D) Set an organization-wide SCP requiring key rotation

<details><summary>Answer</summary>
**A** — AWS Config with a managed rule (e.g. `access-keys-rotated`) can continuously monitor and flag keys that have not been rotated, providing automated compliance. B is manual and error-prone. C identifies unused access but does not enforce rotation. D is not possible — SCPs cannot enforce operational actions like key rotation; they only control permissions.
</details>

---

## Related services

- [[EC2]] — instances use IAM roles via instance profiles for secure access to other AWS services
- [[S3]] — resource-based bucket policies are an IAM mechanism; cross-account bucket access uses IAM roles
- [[STS]] — the engine that issues temporary credentials when roles are assumed
- [[Organizations]] — manages SCPs that set permission ceilings across accounts
- [[KMS]] — IAM policies control who can use KMS keys; key policies are a form of resource-based policy
- [[Cognito]] — provides identity pools that issue temporary IAM credentials to end users
