# Cognito Course  Amazon Cognito (User & Identity Pools)

## 1. Purpose

Cognito is AWS's **customer identity and access management (CIAM)** service for web and mobile apps: sign-up/sign-in, user directories, social/SAML/OIDC federation, MFA, and **temporary AWS credentials** for your app's users. It is for your **customers**, not your employees  employee SSO is IAM Identity Center, AWS-account access is IAM. Built from two components:

- **User pool**  a user directory + authentication/authorization (tokens, MFA, hosted UI). An OIDC provider.
- **Identity pool**  exchanges an authentication token (or guest access) for **temporary IAM credentials (STS)** to reach AWS resources.

## 2. How it works

- **User pool flow**: user signs up → app authenticates against the pool (or via social provider Google/Facebook/Apple/Amazon, SAML, OIDC) → Cognito returns **ID token**, **access token**, and **refresh token** (JWTs) → app calls API Gateway/AppSync/ALB which **validate the token** as an authorizer.
- Includes **hosted UI / managed login** pages, customizable (branding, Lambda triggers).
- **Identity pool flow**: app presents an authenticated token (or uses guest/unauthenticated) → identity pool requests **STS AssumeRoleWithWebIdentity** → returns temporary AWS creds mapped to an **IAM role** (authenticated role vs guest role, selectable per claims).
- **Role/ABAC**: user claims can be mapped to IAM principal **session tags** → resource policies can allow/deny per-user subsets.

## 3. When to use

- App user **sign-up/sign-in** with social (Google/Apple/Facebook/Amazon) or enterprise (SAML/OIDC) federation.
- **MFA** for customers (SMS, email OTP, TOTP, WebAuthn), password reset, and adaptive authentication based on risk.
- Your users need access to **AWS services** (S3 uploads, DynamoDB, AppSync, Lex…) → identity pool gives scoped temp creds.
- **API Gateway / AppSync / ALB** custom authorizers validating JWTs for your API.
- Avoid building/hosting your own auth + user directory.

## 4. When NOT to use

- **Employees/administrators accessing AWS** → IAM + IAM Identity Center, not Cognito.
- **Server-to-server / machine access** → IAM roles/keys, not Cognito pools.
- You need full control over password storage/scheme or own auth flow → custom auth (Cognito still lets you via Lambda trigger / developer provider, but adds complexity).
- Simple static-site with no login → nothing or CloudFront signed URLs.

## 5. Important features

- **User pools**: sign-up/sign-in, **MFA** (SMS/Email/WebAuthn/TOTP, optional or required, **Adaptive Authentication** risk-based), **hosted UI / managed login**, custom **Lambda triggers** (pre/auth/post-signup, migrate user, custom challenge), **groups** (claim→role mapping), password policy, federation, **passwordless**/one-time-password options, token customization.
- **Identity pools**: authenticated + **guest (unauthenticated)** roles, per-IdP role selection, **ABAC via session tags** (attributes-for-access-control), Cognito Sync, multiple IdP support (social, SAML, OIDC, developer providers).
- **Integrations**: API Gateway Lambda authorizer / Cognito authorizer, ALB (OIDC auth), AppSync, WAF (rate limiting on auth endpoints), Secret-key rotation for OAuth client secret.

## 6. Limitations

- User pool isn't a full IAM policy engine  authorization beyond tokens is up to your app/API.
- **SMS sandbox** in new regions requires phone verification before SMSS messages work (exam-relevant edge case).
- Adaptive authentication / some features are **opt-in** SMS MFA costs per message via SNS.
- Quotas: user count, token sizes, IdP integration limits per pool.
- Not a bot/DDoS defense  pair the login UI with **WAF** + **Shield**.
- Token expiry management (refresh tokens) adds app design work.

## 7. Trade-offs

- **Cognito vs IAM Identity Center**: customers/app users vs workforce/SSO into AWS & business apps  pick by identity audience.
- **Cognito vs custom auth (Django/Node/JWT)**: managed sign-up, MFA, federation, compliance vs full control/portability.
- **User pool vs identity pool** (most-tested distinction): *authentication/tokens* vs *temporary AWS credentials* often used together.
- **Hosted UI vs custom pages**: fast/secure/standard vs branded/self-managed flows.
- **Cognito vs Auth0/Okta CIAM**: AWS-native + cheap deep integration vs richer admin console and global identity features.

## 8. Architecture

```
App (mobile/web)
  │  sign-up / sign-in
  ▼
Cognito User pool (hosted UI / managed login, social+SAML+OIDC federation, MFA)
  │  ID/access/refresh tokens (JWT)
  ├──────────────┬──────────────────────────────┐
  ▼              ▼                              ▼
API Gateway /    Identity pool         ALB (OIDC)
AppSync          (STS AssumeRole       authorizer
(authorizer      WithWebIdentity)
 validates)
                 └──► IAM roles (guest/authenticated, session tags/ABAC)
                      └──► S3 / DynamoDB / AppSync (scoped access)
```

- Front the auth pages with **WAF** (rate limits) behind CloudFront.
- Use identity pool roles scoped by claims for least-privileged user access.

## 9. SAA-C03 Perspective

- **User pool = auth (login, tokens, MFA, federation)** **identity pool = temporary AWS creds (IAM)**. This split appears constantly.
- "Mobile/web app users sign in with Google then access S3" → Cognito user pool (federation) + identity pool (creds).
- Authorization of APIs → **API Gateway Cognito authorizer** (validates tokens) or a Lambda authorizer.
- **Cognito vs IAM Identity Center**: customers vs workforce  read the scenario's audience.
- MFA/adaptive auth and guest access are scenario triggers for Cognito.

Exam trap: "users need AWS credentials after logging in" → identity pool "users just need to log in to a web app" → user pool.