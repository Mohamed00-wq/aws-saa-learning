# IAM Course — Identity and Access Management

## 1. Purpose

IAM is the backbone of every security decision in AWS. It is a **global** service (not region-scoped) that answers two questions for every API call: **who** is calling (authentication) and **what** they may do (authorization). Every AWS request — console, CLI, SDK, service-to-service — passes through IAM evaluation first. Maps directly to the **Design Secure Architectures** domain (30% of SAA-C03).

## 2. How it works

```
Request → IAM evaluation → Allow / Deny → API executes
```

- **Identities**: Root user · IAM users (long-lived, up to 2 access keys) · Groups (contain users only, **can't be nested**) · Roles (temporary, assumed via STS).
- **Policies**: JSON with `Version`, `Statement` (`Effect`, `Action`, `Resource`, optional `Condition`). Attached to identities (identity-based) or resources (resource-based).
- **Roles** = trust policy (WHO can assume) + permission policy (WHAT they can do). Assumed via STS with temp creds (default 1h, up to 12h).
- **Evaluation logic**: default deny → any explicit **Deny wins** → explicit Allow needed → SCPs first (outer ceiling) → permission boundaries/session policies intersect → resource-based policies combine with identity.

## 3. When to use

- Every account, always — it is mandatory, not optional.
- Granting EC2/Lambda/ECS access to other services **via roles** (never hardcode keys).
- **Cross-account access**: role in target account + trust policy, no duplicate users.
- **RBAC/least privilege**: per-identity policies, permission boundaries to cap developers.
- Multi-account control via **SCPs** (with Organizations/Control Tower).

## 4. When NOT to use

- **App/mobile user authentication** → use Cognito (IAM is for AWS identities, not your customers).
- **Employee SSO across apps** → use IAM Identity Center (previously SSO).
- **Long-lived programmatic access** where possible — prefer roles; static keys for people only.
- **Granular data-plane filtering** (rate limits, SQLi) → WAF, not IAM policies.
- Root user for daily work — lock it away with MFA, use only for root-only tasks.

## 5. Important features

- **Policy types**: AWS-managed (pre-built, versioned), customer-managed (you control, recommended), inline (embedded 1:1).
- **Policy evaluation**: explicit Deny always overrides; SCPs grant nothing, only restrict (can even limit root); boundaries/session policies only reduce.
- **STS** (`AssumeRole`): powers roles, cross-account, instance profiles, web identity/SAML federation.
- **IAM Access Analyzer**: finds external (cross-account) access and unused access (Zelkova formal verification).
- **IAM Identity Center**: SSO across accounts + SAML apps (Okta, Azure AD).
- **Policy variables**: `arn:aws:s3:::${aws:username}/*` for per-user access.
- **Condition keys**: `aws:SourceArn`, `aws:PrincipalOrgID`, IP, MFA-present, etc.
- **Instance profiles**: wrapper attaching a role to EC2; instance fetches creds from IMDS.

## 6. Limitations

- **Root can't be restricted by IAM** — only by SCP/Organizations.
- **Groups can't be nested** and are not identities (can't be principals / assume roles).
- **Access keys aren't auto-rotated**; max 2 per user.
- IAM policies can't do complex "first match wins" logic — evaluation is fixed (Deny wins, allow from any source).
- No data-plane filtering (can't inspect packet contents); that's NACL/Security Group/WAF territory.
- IAM is global, but resources it protects are region-scoped.

## 7. Trade-offs

- **Identity-based vs resource-based**: identity = who; resource = what resource grants directly (S3 bucket policy = cross-account without roles; but both needed for some flows).
- **Users vs roles**: users = permanent credentials for people; roles = short-lived temp creds for apps/services (better security, but sts-to-role latency).
- **Manageable policies vs inline**: shared/managed is easier to audit and update; inline is 1:1 but not reusable.
- **SCP vs permission boundary**: SCPs set org-wide ceiling; boundaries set per-account/identity ceiling — both restrict, never grant.

## 8. Architecture

```
                                        IAM evaluation (global)
                                              │
IAM role (instance profile) ← EC2 ──────► S3/DynamoDB (resource policy)
        │
Lambda role ← Lambda ────► SQS/SNS/KMS (grants, key policies)
        │
Cross-account: Account B role (trust: Account A) ← Account A IAM user/role
```

- Services get roles (never keys); resources get resource-based policies or key policies.
- Use SCPs at OU level for guardrails (e.g. block DeleteBucket in prod).
- Permission boundaries on developer roles to enforce least privilege.

## 9. SAA-C03 Perspective

- **Role > access key** in any scenario asking for EC2/Lambda to access another service.
- **Groups can't be nested** and **SCPs grant nothing** are frequent traps.
- **Cross-account**: role + trust policy (not cross-account users or shared keys).
- **Deny wins** over any Allow, from any policy type.
- Know how **users vs groups vs roles vs federation** fit a given scenario.
- Access Analyzer for "find external access" and "least privilege" questions.

Frequent exam trap: they ask to secure access between services — always pick **IAM roles** (temp creds), not hardcoded access keys or a second account.