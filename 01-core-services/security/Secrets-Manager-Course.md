# Secrets Manager Course  AWS Secrets Manager

## 1. Purpose

Secrets Manager stores, retrieves, and **automatically rotates** sensitive values (database passwords, API keys, OAuth tokens, certificates) so they never live in code, config files, or images. Backed by KMS encryption, IAM access control, and multi-region replication. Everything "store credentials securely" on the exam points here.

## 2. How it works

- A **secret** is key/value or JSON (up to 64 KB) encrypted at rest with a KMS key (default: AWS managed `aws/secretsmanager` customer-managed for cross-account/key-policy control).
- **Access**: IAM policies + optional **resource policies** (for cross-account) retrieved over TLS.
- **Versioning with staging labels**: `AWSCURRENT` (active), `AWSPREVIOUS` (last known good), `AWSPENDING` (during rotation).
- **Rotation**:
  - **Managed rotation**  the integrating service rotates for you (RDS/Aurora/Redshift managed secrets), no Lambda needed.
  - **Rotation by Lambda**  a rotation function executes 4 steps: `createSecret` (new value → AWSPENDING) · `setSecret` (apply to DB/service) · `testSecret` (verify) · `finishSecret` (move AWSCURRENT). Strategies: **single user** or **alternating users** (two users/clone).
- **Client-side caching** (SDK) reduces API calls and cost apps keep using cached value across rotations.

## 3. When to use

- **Database credentials** for RDS, Aurora, Redshift, DocumentDB (tight integration + rotation + RDS Proxy).
- Any **API key / password / OAuth token / SSH key** your apps consume.
- **Automatic rotation** requirement for compliance or operational hygiene.
- **Cross-account** secret sharing (resource policy + KMS key policy) and **multi-region replication** for DR.
- Accessing secrets from **Lambda/ECS/EKS** via the Workload Credentials Provider SDK.

## 4. When NOT to use

- **Non-secret config** (endpoints, feature flags, plain settings) → SSM Parameter Store / AppConfig (cheaper, no rotation).
- Small static values with no rotation need → SSM Parameter Store (`SecureString`).
- BYO-HSM / raw key management → KMS or CloudHSM (Secrets Manager is for *values*, not keys).
- AWS access credentials/roles → IAM, never store IAM user keys as secrets.

## 5. Important features

- **Automatic rotation** on a schedule (rate/cron), as often as every 4 hours.
- **Random password generation** with complexity rules.
- Staging labels / rollback to any prior version.
- **Multi-region replication** (promote replica to standalone) for DR and latency.
- Cross-account access via resource policies + custom KMS keys (use `secretsmanager.<region>.amazonaws.com` as `kms:ViaService`).
- Integrations: RDS/Aurora/Redshift, Lambda, EventBridge, CloudFormation/DynamicReference, CodeBuild/GitHub Actions, CloudTrail audit.
- **Managed external secrets**: rotate partner-held secrets (e.g. New Relic) without your own logic.

## 6. Limitations

- **Cost**: per-secret monthly fee + per-API-call charges  parameter store is cheaper for high-read, low-value config.
- **Rotation functions** are your responsibility for non-managed secrets (Lambda code, permissions, VPC networking).
- Secret **sharing** is manual (resource policy + key policy)  not a global secret store across orgs automatically.
- Value size cap ~64 KB not designed for binary certs/blob storage.
- No built-in audit dashboard beyond CloudTrail/event history.

## 7. Trade-offs

- **Secrets Manager vs SSM Parameter Store**: rotation, versioning labels, multi-region + auto-rotation vs near-free, hierarchical config, plaintext/SecureString. Exam picks Secrets Manager for rotation/cross-account secrets, Parameter Store for config.
- **Single vs alternating user rotation**: simpler with brief auth-less window vs zero-downtime for server farms (requires a superuser secret).
- **Managed vs Lambda rotation**: zero-maintenance for supported services vs flexibility for custom targets.
- **Caching vs live reads**: lower cost/latency vs always-fresh values (accept brief stale).

## 8. Architecture

```
App (Lambda/ECS/EC2, IAM role) ──> Secrets Manager ──> KMS (decrypt) ──> secret value
        │ (caching SDK)
        └──> RDS / Aurora / Redshift (DB creds)
Lambda rotation function (VPC, SG) ──> secret 4-step rotation ──> updates DB creds + secret
Secret replicated to secondary Region for DR failover
```

- Least-privilege IAM (`secretsmanager:GetSecretValue` on specific ARN).
- Customer-managed KMS key for cross-account or key-policy control.
- Cache + retry logic use separate app user (not master) for DB access.

## 9. SAA-C03 Perspective

- "Store + **rotate** credentials" → Secrets Manager (the rotation word is the trigger Parameter Store can't rotate).
- "Store config / non-secret parameters" → SSM Parameter Store.
- **Rotation failure / fresh creds across Region** → multi-region replication + caching strategy.
- **Cross-account secret access** → resource policy + KMS key policy grant.
- Integration with **RDS Proxy / Aurora Secrets** for managed rotation answers.

Exam trap: they offer "SSM Parameter Store $0.40/month vs Secrets Manager" – correct pick is Secrets Manager whenever rotation or lifecycle of credentials is mentioned.