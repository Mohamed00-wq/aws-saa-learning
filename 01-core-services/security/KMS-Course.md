# KMS Course — AWS Key Management Service

## 1. Purpose

KMS is AWS's **central encryption key management** service. It creates, stores, rotates, and controls access to cryptographic keys used across AWS services and in your own applications (encryption at rest). It is the encryption engine behind SSE-KMS for S3/EBS/RDS and most other AWS integrations, and is required for "Encrypt data at rest" questions on the exam.

## 2. How it works

- Keys live in KMS, are stored encrypted in FIPS 140-3 validated HSMs, and never leave the service unencrypted.
- **KMS keys** (formerly CMKs):
  - **Symmetric** (encrypt/decrypt, up to 4,096 bytes, most common)
  - **Asymmetric** (public/private for sign-verify / encrypt-decrypt)
  - **HMAC** (GenerateMac/VerifyMac) and ML-DSA (post-quantum) key types.
- **Key access control**: a **key policy** (resource-based, always consulted) + IAM policies + optional **grants** (temporary, per-service permissions).
- **Envelope encryption**: app calls `GenerateDataKey` → KMS returns plaintext data key + encrypted copy → app encrypts data with the **data key** and stores the encrypted key alongside. Decryption reverses it (`Decrypt` on the key). This avoids sending data to KMS.
- **Key rotation**: automatic (default every year, custom 90–2560 days) or on-demand; same key ID/ARN stays, old key material is kept so old ciphertext still decrypts.

## 3. When to use

- **Encryption at rest** for S3 (SSE-KMS), EBS, RDS, EFS, EKS, Lambda env vars, SQS/SNS, etc.
- Compliance requiring **key rotation** or customer-owned/managed keys.
- **Cross-region DR**: multi-Region keys (same key ID + material) encrypt in one region, decrypt in another without re-encryption or cross-region calls.
- **Cross-account** data sharing via key policies + grants.
- App-level envelope encryption (encrypt payloads client/app-side with SDK).
- Regulatory need for keys in **your own HSM** → CloudHSM custom key store, or external key store.

## 4. When NOT to use

- **Client-side encryption where you don't want KMS calls** → AWS Encryption SDK / S3 client-side encryption (though these still use KMS keys).
- Simple "use default encryption" without key controls → AWS managed keys or AWS owned keys (SSE-S3 -AES256 etc.).
- Storing **secrets/credentials** → use Secrets Manager (it uses KMS under the hood).
- Full control of the HSM hardware itself → CloudHSM directly.
- Huge blobs (>4 KB) directly — must use data keys/envelope encryption, not one-shot Encrypt.

## 5. Important features

- **Key tiers**: AWS owned (no control) · AWS managed (`aws/servicename`, auto-rotated) · **customer managed** (full control, recommended for customization).
- **Multi-Region keys**: same key material/ID across regions for DR and controlled failover.
- **Grants**: temporary, scoped permissions (e.g. service access) without editing policies — with encryption-context / SourceArn constraints.
- **Encryption context**: key-value pair bound to ciphertext; tamper detection + condition key for access control.
- **Rotation**: automatic (custom period), on-demand rotation, and manual via alias repointing / importing new material.
- **Imported key material**: bring-your-own-key, with optional expiry, for external-compliance scenarios.
- **VPC endpoints** for private, no-internet KMS access; **CloudTrail** logs every encrypt/decrypt.

## 6. Limitations

- **4,096-byte (4 KB) per-call limit** for symmetric Encrypt/Decrypt — requires data keys for anything larger.
- **Regional** — a key is in one region (multi-Region keys mitigate but are still "related" keys, managed independently).
- **Key deletion** is destructive: scheduled (7–30 days wait), and deleting a key makes ciphertext unrecoverable.
- **Custom key stores**: no automatic rotation (CloudHSM/manual), multi-Region keys not supported there.
- **Quotas**: per-region request rate limits (can raise via support), key count limits.

## 7. Trade-offs

- **AWS owned vs AWS managed vs customer managed**: cost/control vs rotation control and ciphertext independence (AWS owned = no cost, but rotating AWS-owned keys can break ciphertext; customer-managed = you control rotation).
- **KMS keys vs external (BYOK/imported)**: compliance control vs operational overhead (import, expiry, manual rotation).
- **KMS vs CloudHSM**: managed API + integrated with AWS services vs raw HSM with your keys on dedicated hardware.
- **Single-region vs multi-Region keys**: simplicity vs DR flexibility.
- **KMS vs Secrets Manager**: key management vs credential lifecycle (+rotation of the *secret*, not the key).

## 8. Architecture

```
┌ Application ───────────────────────────────┐
│ 1. GenerateDataKey (KMS key "alias/app")    │
│ 2. data key plaintext → encrypt app data    │
│ 3. store (encrypted data key + ciphertext)  │
│ 4. later: Decrypt(encrypted key) → plaintext│
└────────────────────────────────────────────┘
        │
        ▼ (grant / key policy + IAM)
     AWS KMS (HSM-backed, region-scoped)
        ▲
        │ SSE-KMS (managed key aws/s3, ebs, rds …)
   S3 / EBS / RDS / Lambda / EKS …
```

- Customer-managed key with least-privilege key policy + IAM.
- Multi-Region keys for DR (same ID both sides).
- Use encryption context + grants for service integrations.

## 9. SAA-C03 Perspective

- **Envelope encryption** and the 4 KB limit are the core concepts — know why data keys exist.
- Distinguish **SSE-S3 (AES-256), SSE-KMS, SSE-C** — KMS = rotation, audit, compliance.
- Know key tiers: when the question says "customer managed/controlled rotation" → KMS customer-managed key.
- **Multi-Region keys** for cross-region DR decrypt; **grants** for cross-account/temporary access.
- **KMS vs Secrets Manager**: secrets (credentials) → Secrets Manager; keys → KMS.

Exam trap: encrypting a large file → envelope encryption with data keys, NOT a single KMS Encrypt call.