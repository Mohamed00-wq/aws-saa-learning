# Lab — S3 Foundation & Data Lifecycle (Conceptual Walkthrough)

> **Goal:** Understand how to build a complete object storage stack in S3 from scratch — bucket → versioning → encryption → lifecycle rules → static website hosting → object access. This walkthrough explains each step conceptually so you understand the "why" before you touch the CLI.

**Exam domains covered:** Design Secure Architectures (30%) · Design Resilient Architectures (26%) · Design Cost-Optimized Architectures (20%)

---

## Part 1 — Create an S3 bucket

**What you need to understand:**
- A **bucket** is a globally-unique container for objects. The name is unique across **all AWS accounts and regions**, not just yours.
- Bucket names follow strict rules: 3–63 characters, lowercase letters, numbers, hyphens, periods; must start and end with a letter or number.
- Choose a **region** at creation — this is where the data physically resides. The bucket is accessible from anywhere, but it lives in one region. Cross-region access costs more and is slower.
- A bucket **cannot be renamed** after creation. If you need a different name, create a new bucket and copy the objects.

**Why this matters for the exam:** bucket naming, region placement, and the fact that a bucket can't be renamed are all tested. Also, "publicly accessible bucket" is the default configuration if you don't block it — most exam scenarios require Bucket Policy + Block Public Access reasoning.

---

## Part 2 — Bucket Versioning

**What you need to understand:**
- By default, overwriting an object destroys the old version. **Versioning** preserves every version of an object, including deleted "versions".
- When versioning is enabled:
  - Uploading the same key creates a **new version** — the previous one is retained.
  - Deleting an object creates a **delete marker** — the object disappears from view, but the data and all prior versions are still stored (and billed).
  - You can restore a deleted object by removing the delete marker or promoting a prior version.
- **MFA Delete** can be added so permanent deletion requires an MFA token — defense against accidental or malicious deletion.

**Exam note:** versioning cannot be disabled once enabled — only suspended. Suspending stops new versions from being created, but existing versions remain until explicitly deleted.

---

## Part 3 — Encryption at rest

**What you need to understand:**
Encryption is configured at the bucket level (default encryption) and applied to every new object:

| Method | Who manages the key | Use case |
|---|---|---|
| **SSE-S3** | AWS | Default. Zero config, AES-256. |
| **SSE-KMS** | You (via KMS) | Need to control/rotate keys, audit usage, or restrict decryption privileges. |
| **SSE-C** | You (raw keys) | You must hold the keys; S3 discards them after encrypting — you send the key with every request. |
| **Client-side** | You | Encrypt before upload; S3 never sees plaintext. |

**Exam trap:** the question "which encryption if you must provide your own key but don't want KMS?" → **SSE-C**. "You need per-key rotation and audit logs" → **SSE-KMS**. "Just encrypt it, no management" → **SSE-S3**.

---

## Part 4 — Access control

**What you need to understand:**
There are multiple independent layers of access control:

- **IAM policies** — control which principals (users/roles) in your account can call S3 APIs.
- **Bucket policies** — JSON attached to the bucket. Can grant access to specific IAM principals, and — crucially — **cross-account** principals and **anyone** (public).
- **ACLs** — legacy, coarse-grained (object-level) grants. Avoid in favor of bucket policies.
- **Block Public Access** — an account/bucket-wide override that blocks public access even when a policy grants it.

**Access evaluation:** an S3 request succeeds if any applicable policy grants permission **AND** no deny applies. Explicit denies win; Block Public Access acts as an additional override.

**Exam note:** granting another AWS account access requires a **bucket policy** — IAM alone cannot grant cross-account access.

---

## Part 5 — Storage classes & data lifecycle

**What you need to understand:**
S3 has multiple storage classes trading cost vs retrieval speed vs resilience:

| Class | Cost | Retrieval | Notes |
|---|---|---|---|
| **Standard** | Highest | Instant | Frequently accessed data |
| **Intelligent-Tiering** | Auto | Instant | Auto-moves rarely-accessed data to lower tiers; no retrieval fee |
| **Standard-IA** | Low | Instant | Infrequent, still needs fast access |
| **One Zone-IA** | Lower | Instant | Single AZ, re-creatable data only |
| **Glacier Instant** | Low | Millisecond | Archived but needed occasionally |
| **Glacier Flexible** | Lower | Minutes–hours | Archive |
| **Glacier Deep Archive** | Lowest | 12–48 hours | Long-term compliance archive |

**Lifecycle rules** automate transitions:
- Standard → Standard-IA/One Zone-IA: minimum **30 days** old.
- → Glacier Flexible/Instant: minimum **90 days old**.
- Expiration: delete after N days, often 7 years for compliance.

**Why this matters for the exam:** lifecycle rules + storage classes are the single most-tested cost-optimization topic in S3. Quoting, the key numbers are 30/90/365 and the Deep Archive retrieval time (12–48h).

---

## Part 6 — Static website hosting

**What you need to understand:**
A bucket can serve a **static site** (HTML/CSS/JS) without any server:

1. Enable **Static website hosting** on the bucket.
2. Specify an **index document** (`index.html`) and optional **error document**.
3. Attach a bucket policy allowing `s3:GetObject` for all principals (`*`).
4. Access via `<bucket>.s3-website-<region>.amazonaws.com`.

**Exam traps:**
- The S3 website endpoint is **HTTP only** — no HTTPS. For HTTPS, put **CloudFront** in front with an ACM certificate.
- The website endpoint is distinct from the REST API endpoint (`s3.<region>.amazonaws.com/<bucket>`). They behave differently (website endpoint only serves GET/HEAD for the index/error docs; no HTTPS).
- Setting the bucket public for website hosting requires explicitly allowing `s3:GetObject` in a bucket policy — and Block Public Access must be off (unusual, deliberate).

---

## Part 7 — Object sharing with presigned URLs

**What you need to understand:**
- A **presigned URL** grants temporary access to a private object without exposing the bucket.
- Generated via SDK/CLI with a credential and an expiration (seconds to hours — max 7 days for SigV4 with IAM credentials).
- Anyone with the URL can access the object until it expires.
- Roles with temporary credentials produce shorter expirations (max 7 days for longer-lived credentials; role-based is limited).

**Exam note:** presigned URLs are for time-bound access to private objects (e.g. "user downloads this report for 24 hours"). They do not change the bucket's underlying policy.

---

## Part 8 — Handling large objects (Multipart Upload & Transfer Acceleration)

**What you need to understand:**
- **PUT** limit: 5 GB per call. Objects up to **5 TB** require **Multipart Upload** — split into parts (min 5 MB each, except the last), upload in parallel, combine.
- Multipart recovers from failures: only the failed part needs retrying, not the whole file.
- **S3 Transfer Acceleration** routes uploads over the AWS backbone via edge locations — faster for large transfers across long distances. Endpoint: `<bucket>.s3-accelerate.amazonaws.com`.

**Exam trap:** "which enables uploading 10 GB files?" → Multipart. "Which speeds up large cross-region uploads?" → Transfer Acceleration.

---

## Part 9 — Event-driven workflows

**What you need to understand:**
S3 can emit **event notifications** (Put, Post, Copy, Delete) to **Lambda**, **SQS**, **SNS**, or **EventBridge**:
- **Lambda** — process each object (thumbnail, validation, transformation).
- **SQS** — queue events for batch workers.
- **SNS** — notify subscribers.
- **EventBridge** — route/filter to other services.

**Exam note:** "when a file lands in S3, process it" is almost always S3 event → Lambda. For decoupling/throttling, route through SQS.

---

## Recap — What you built and why

```
S3 Bucket (globally unique name, region-scoped)
 ├── Versioning: enabled → protects against overwrite/deletion
 ├── Encryption: SSE-S3 / SSE-KMS → data protected at rest
 ├── Policies: IAM + bucket policy + Block Public Access → least privilege
 ├── Storage class & lifecycle: Standard → IA → Glacier → expire → cost control
 ├── Static hosting: index.html served as a website (CloudFront for HTTPS)
 └── Event notification: Put → Lambda for automated processing
```

Every piece serves a purpose:
- **Bucket** — the container, globally unique.
- **Versioning** — recoverability against accidental deletion.
- **Encryption** — confidentiality at rest.
- **Access control** — least privilege, no accidental public exposure.
- **Storage classes + lifecycle** — pay only for what each object needs.
- **Website hosting / presigned URLs** — how objects reach users.

---

## Exam-style Questions

**Q1.** A company needs to store 40 GB files with the ability to resume failed uploads. What should they use?
- A) S3 PUT with a single request
- B) Multipart Upload
- C) S3 Transfer Acceleration
- D) S3 Standard-IA

<details><summary>Answer</summary>
**B** — Multipart Upload splits the object into parts that can be uploaded (+retried) independently. A single PUT (A) caps at 5 GB. Acceleration (C) speeds transfers but doesn't provide resume/retry, and storage class (D) is irrelevant to upload mechanics.
</details>

**Q2.** Which S3 capability protects against accidental object deletion?
- A) Bucket policies
- B) Versioning
- C) SSE-KMS
- D) Lifecycle rules

<details><summary>Answer</summary>
**B** — versioning preserves every version including delete markers, so a delete can be reversed by removing the marker / restoring a version. Policies (A) and encryption (C) do not counter deletion, and lifecycle (D) actively deletes.
</details>

**Q3.** An S3 bucket stores compliance data that must be kept for 7 years, is rarely accessed, and requires the LOWEST storage cost. What storage class should be used?
- A) S3 Glacier Deep Archive
- B) S3 Standard-IA
- C) S3 Standard
- D) S3 One Zone-IA

<details><summary>Answer</summary>
**A** — Glacier Deep Archive is the lowest-cost class and fits long-term archive (12–48h retrieval acceptable here). Standard/IA (B, C) cost more; One Zone-IA (D) is not appropriate for compliance data that must not lose a single AZ.
</details>

**Q4.** An Architect must serve a static website from S3 over HTTPS with a custom domain. What is the correct approach?
- A) Enable static hosting and use the default `.s3-website-` URL
- B) Put CloudFront in front of S3 and attach an ACM certificate
- C) Use S3 Transfer Acceleration
- D) Upload the site as encrypted objects

<details><summary>Answer</summary>
**B** — the S3 website endpoint (A) is HTTP-only. CloudFront + ACM provides HTTPS on a custom domain while S3 stays the origin. Acceleration (C) is about upload speed, not HTTPS serving.
</details>

**Q5.** A developer generates files that must be downloadable by a client for 24 hours but not publicly accessible afterward. What should they use?
- A) Make the bucket public
- B) Generate a presigned URL with a 24-hour expiry
- C) Use SSE-C encryption
- D) Enable versioning

<details><summary>Answer</summary>
**B** — presigned URLs grant temporary, time-limited access to specific objects without making the bucket public. Public bucket (A) exposes everything; encryption (C) and versioning (D) don't control access duration.
</details>

**Q6.** Which of the following is TRUE about lifecycle transitions?
- A) Objects can go straight to Glacier on the day they are written
- B) Transition to Standard-IA requires the object to be at least 30 days old
- C) Lifecycle rules only delete objects, never move them
- D) Transition to Glacier requires the object to be at least 30 days old

<details><summary>Answer</summary>
**B** — lifecycle transitions require minimum ages: 30 days for IA transitions and 90 days for Glacier. Rule C is false — rules both transition and expire. Rule A/D have the wrong age thresholds.
</details>

---

## Related Services

[[S3]] · [[CloudFront]] · [[Lambda]] · [[IAM]] · [[KMS]] · [[EC2]] · [[Glacier]]