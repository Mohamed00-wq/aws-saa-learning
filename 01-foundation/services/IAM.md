# IAM — Identity and Access Management

![IAM](/images/icons/arch/Arch_AWS-Identity-and-Access-Management_64.svg)

## What it is

IAM is the backbone of every security decision in AWS. It is a **global** service (not region-scoped) that governs two fundamental questions: **who** is making a request (authentication) and **what** they are allowed to do (authorization). Every single AWS API call whether from the console, CLI, SDK, or an internal service-to-service call — passes through IAM evaluation before anything happens.

IAM does not govern data-plane traffic directly (e.g. an EC2 instance reaching out to the internet) it governs control-plane and data-plane API permissions against AWS resources. Understanding IAM is not optional it maps directly to the **Design Secure Architectures** domain which carries the highest weight on the SAA-C03 exam.

## Identities

- **Root user**: created at account open, unrestricted, can't be denied by IAM, never deleted. Lock it away + MFA, only for root-only tasks (close account, change payment). Credentials = email + password.
- **IAM Users**: long-lived, permanent password + up to two access keys. For people/legacy only — **use roles for apps/services**. Keys aren't auto-rotated.
- **IAM Groups**: flat collection of users, grants policy to all members. **Cannot be nested** (exam trap). Not an identity (can't assume roles / be a principal).
- **IAM Roles**: the most important concept. **Temporary identity** — assumed via **STS**, credentials expire (default 1h, up to 12h). Two policies:
  1. **Trust policy** — WHO can assume (account, EC2, federated, Lambda)
  2. **Permission policy** — WHAT the assumed role can do
  - **Best practice**: grant EC2 access to S3 via role, never hardcode keys.
  - Types: **service roles**, **cross-account roles**, **federated roles** (SAML/Google), **instance profiles** (wrapper to attach role to EC2; instance fetches creds from IMDS).

## Policies

Every policy: **Version** (`2012-10-17`) + **Statement** (Effect `Allow`/`Deny`, **Action**, **Resource** ARN, optional **Condition**).

| Type | Attached to | Notes |
|---|---|---|
| **Identity-based** | user/group/role | what the identity can do |
| **Resource-based** | resource (S3, SQS, KMS) | can grant access to **other accounts** directly |
| **Permission boundary** | user/role | **ceiling** — effective permission = **intersection** |
| **Session policy** | role assume | restricts session, intersection |
| **SCP** | OU/account | org-wide ceiling, grants nothing, can even limit **root** |

**Managed vs inline:** AWS-managed (pre-built, versioned, can't modify) · Customer-managed (you control, recommended) · Inline (embedded 1:1, not reusable).

## Policy evaluation logic

1. **Default deny** — nothing allowed unless explicit Allow
2. **Explicit Deny always wins** — any Deny anywhere → denied
3. **Explicit Allow required** — after Deny check, need one Allow
4. **SCP checked first** — sets outer ceiling
5. **Permission boundary** — intersection with identity policy
6. **Resource-based policy** evaluated alongside identity — Allow from either permits
7. **Session policy** — intersection on assume

**Shorthand:** Deny wins → default deny → Allow must be explicit → SCPs are outer ceiling → boundaries/session restrict further.

## STS — engine behind roles

1. Caller sends `AssumeRole` → 2. STS checks trust policy → 3. applies permission + session policies → 4. returns temp creds (key, secret, token) → 5. expire after duration.
Also powers: Web Identity Federation, SAML 2.0, cross-account, EC2 instance profiles.

## IAM Access Analyzer

Identifies **external access** (resources shared outside account) and **unused access** (policies/creds not used). Uses Zelkova formal verification.

## IAM Identity Center (SSO)

Single sign-on across accounts + apps + SAML. Integrates with Organizations; assigns **permission sets** per user/group. Works with Okta, Azure AD via SAML.

## Exam domains

- [x] **Secure (30%)** — core service for the largest domain
- [ ] Resilient (26%) — roles for service-to-service access

## Key gotchas

1. **Implicit deny ≠ explicit Deny** — only explicit `"Effect":"Deny"` overrides
2. A **Deny in any policy** (identity, resource, SCP, boundary) beats any Allow
3. **SCPs grant nothing** — they only restrict
4. **Permission boundaries/session policies never expand** — intersection only reduces
5. **Groups can't be nested** — only contain users
6. **Root can't be restricted by IAM** — only via SCP/Organizations
7. **IAM is global**, policies work across regions
8. **Cross-account** = role + trust policy, no duplicate users needed
9. Max **two access keys** per user
10. Policy **variables** allowed: `arn:aws:s3:::${aws:username}/*` (user-only access)


## Related services

- [[EC2]] — roles via instance profiles for secure service access
- [[S3]] — resource-based bucket policies; cross-account via roles
- [[STS]] — issues temp creds for role assumption
- [[Organizations]] — SCPs across accounts
- [[KMS]] — key policies are resource-based
- [[Cognito]] — identity pools issue temp IAM creds
