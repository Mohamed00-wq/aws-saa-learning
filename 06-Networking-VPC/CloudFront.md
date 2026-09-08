![CloudFront](/images/icons/arch/Arch_Amazon-CloudFront_64.svg)

# CloudFront — Content Delivery Network (CDN)

## What it is

CloudFront is AWS's global **Content Delivery Network (CDN)**. It caches content at **edge locations** around the world, so users get content from the location nearest to them instead of from the origin server directly. This reduces latency, offloads traffic from the origin, and adds a layer of security (HTTPS, DDoS protection) in front of your application.

CloudFront sits in front of an **origin** — most commonly an **S3 bucket** (static assets, websites) or an **HTTP/HTTPS origin** such as an Application Load Balancer, EC2 instance, or any web server, including origins outside AWS. It is the standard answer whenever a question mentions "global users," "low latency," "reduce load on origin," or "HTTPS for a static site."

---

## Key concepts

### Edge Locations

CloudFront runs on a global network of **edge locations** (hundreds worldwide, far more than the number of AWS regions). When a user requests content:

- CloudFront checks the nearest edge location for a **cached copy**.
- **Cache hit** — content is served immediately from the edge (fast).
- **Cache miss** — CloudFront fetches the content from the **origin**, caches it at the edge, and serves it. Subsequent requests for the same content are then served from cache.

Edge locations are read/write for cache but are not full AWS regions — they don't run arbitrary compute (except **Lambda@Edge** / **CloudFront Functions**, see below).

### Distributions

A **distribution** is the CloudFront configuration object that ties an origin to caching/access behavior. Two types:

- **Web Distribution** — for websites, S3 static content, dynamic HTTP(S) content, and APIs.
- **RTMP Distribution** — legacy, for media streaming (deprecated by AWS; not tested).

Each distribution gets a default domain name (`d111111abcdef8.cloudfront.net`) and can be mapped to a **custom domain** with an ACM certificate.

### Origins

CloudFront supports two origin types:

- **S3 origin** — used for static assets or an S3 static website. Access can be locked down using **Origin Access Control (OAC)**, so the bucket is only reachable through CloudFront, not directly.
- **Custom origin** — any HTTP(S) backend: ALB, EC2, on-premises server, or a non-AWS server. Used for dynamic content and APIs.

A single distribution can have **multiple origins** with **path-based routing** (e.g. `/images/*` → S3, `/api/*` → ALB) via **cache behaviors**.

### Cache Behaviors

**Cache behaviors** define how CloudFront handles requests for a given path pattern:

- Which origin to route to.
- Whether to forward query strings, cookies, and headers to the origin.
- **TTL (Time To Live)** — how long content stays cached before CloudFront re-checks the origin. Can be set per behavior (min/default/max TTL), or controlled by `Cache-Control`/`Expires` headers from the origin.
- Viewer protocol policy (HTTP, HTTPS-only, redirect HTTP→HTTPS).

Each distribution has one **default cache behavior** and can have additional behaviors for specific path patterns, evaluated in order.

### Invalidations

If you update content at the origin but the edge cache still holds the old version (TTL hasn't expired), you can force a refresh with an **invalidation** — explicitly telling CloudFront to discard cached copies of specific paths (e.g. `/images/*`) so the next request goes back to the origin.

- Invalidations cost money past a free monthly allotment; frequent invalidations of many files can get expensive.
- Alternative to invalidation: **versioned file names** (e.g. `app.v2.js`) so new content is a "new" object CloudFront hasn't cached yet — often cheaper and preferred at scale.

### Signed URLs & Signed Cookies

To restrict access to private content served through CloudFront:

- **Signed URLs** — grant time-limited access to a **single file**. Good for individual downloads (e.g. a paid PDF).
- **Signed Cookies** — grant time-limited access to **multiple files** (e.g. all videos in a course) without changing URLs, by setting cookies that CloudFront checks before serving content.

Both are generated using a CloudFront key pair or trusted signer and are the standard way to serve private content through a CDN (contrast with S3 presigned URLs, which work directly against S3 without CloudFront).

### Origin Access Control (OAC)

**OAC** (replacing the older Origin Access Identity/OAI) locks an S3 origin down so that:

- The S3 bucket is **not publicly accessible** directly.
- Only the CloudFront distribution (using OAC credentials) can retrieve objects from the bucket.
- The bucket policy is updated to allow `s3:GetObject` only for the CloudFront service principal tied to that specific distribution.

This is the recommended pattern any time CloudFront fronts an S3 bucket that shouldn't be publicly reachable on its own.

### Geo Restriction

CloudFront can **allow-list or block-list countries** at the distribution level (geo restriction), useful for licensing or compliance requirements (e.g. "content only available in the EU").

### Price Classes

You can limit which edge locations are used to control cost:

- **Price Class All** — all edge locations globally (best performance, highest cost).
- **Price Class 200 / 100** — subsets of regions (e.g. excluding South America, Australia) at lower cost, with reduced performance for excluded regions.

---

## Architecture deep dive

### CloudFront ↔ S3

The most common CloudFront pattern: an S3 bucket holds static assets or a static website, and CloudFront:

- Caches content at edge locations for **low latency worldwide**.
- Serves content over **HTTPS** (the raw S3 website endpoint is HTTP-only).
- Uses **OAC** to keep the bucket private and force all traffic through CloudFront.
- Reduces GET requests hitting S3 directly, lowering S3 request costs.

**Exam pattern:** "S3-hosted static site needs HTTPS + custom domain + global low latency" → CloudFront in front of S3 with OAC and an ACM certificate.

### CloudFront ↔ ALB / EC2 (dynamic content)

CloudFront isn't just for static content. In front of an ALB or EC2 origin, it can:

- Cache **cacheable dynamic responses** (e.g. product listing pages) while passing through non-cacheable requests (e.g. checkout) to the origin.
- Terminate TLS at the edge, reducing origin load.
- Provide a single entry point/domain for a multi-origin application (static assets from S3, API calls from ALB) using path-based cache behaviors.

### CloudFront ↔ ACM (certificates)

To serve a **custom domain** over HTTPS, you need an **ACM certificate**. Important exam detail: for CloudFront, the certificate **must be requested/imported in the `us-east-1` (N. Virginia) region**, regardless of where your origin or users are — CloudFront is a global service and only reads certificates from `us-east-1`.

### CloudFront ↔ Route 53

CloudFront distributions are commonly pointed to by a **Route 53 alias record** at the apex or subdomain of a custom domain (e.g. `www.example.com` → CloudFront distribution), which works with the zone apex unlike a plain CNAME.

### CloudFront ↔ WAF & Shield

- **AWS WAF** can be attached to a CloudFront distribution to filter malicious requests (SQL injection, XSS, rate limiting) at the edge, before they reach the origin.
- **AWS Shield** (Standard, included free) protects CloudFront against DDoS attacks automatically; **Shield Advanced** adds enhanced protections and cost protection.

### Lambda@Edge and CloudFront Functions

Both let you run code at the edge to customize requests/responses, but differ in scope:

| Feature | CloudFront Functions | Lambda@Edge |
|---|---|---|
| Language | JavaScript only | Node.js / Python |
| Use case | Lightweight: header manipulation, URL rewrites, simple auth checks | Heavier logic: origin selection, external calls, body manipulation |
| Runtime | Sub-millisecond, runs on every edge location | Higher latency, runs on a subset of edge locations |
| Trigger points | Viewer request/response only | Viewer and origin request/response |

**Exam heuristic:** simple, high-scale, low-latency edge logic → CloudFront Functions. Complex logic needing more compute/language flexibility → Lambda@Edge.

---

## Exam domain(s)

- [x] **Design Secure Architectures (30%)** — OAC for private S3 origins, signed URLs/cookies, WAF integration, HTTPS via ACM (us-east-1 requirement)
- [x] **Design Resilient Architectures (26%)** — multi-origin failover, Shield DDoS protection, global edge redundancy
- [x] **Design High-Performing Architectures (24%)** — edge caching, TTL tuning, price classes, Lambda@Edge/CloudFront Functions
- [x] **Design Cost-Optimized Architectures (20%)** — price classes, reducing origin requests via caching, invalidation cost trade-offs vs versioned filenames

---

## Advanced gotchas & edge cases

1. **ACM certificates for CloudFront must be in `us-east-1`.** This is one of the most frequently tested details — regardless of where your users or origin are.

2. **The default S3 website endpoint is HTTP-only.** CloudFront (with ACM) is the standard way to add HTTPS to an S3-hosted static site.

3. **OAC vs public bucket.** If a bucket is public and also fronted by CloudFront, users can bypass CloudFront entirely by hitting S3 directly — defeating caching, HTTPS enforcement, and WAF protection. OAC + Block Public Access closes this gap.

4. **Invalidations aren't free at scale.** For frequently changing files, versioned object keys are often more cost-effective than repeated invalidations.

5. **Signed URLs vs Signed Cookies.** Single file → signed URL. Multiple files without changing each URL → signed cookies. A common distractor is presigned S3 URLs, which are a different, S3-only mechanism.

6. **CloudFront Functions vs Lambda@Edge.** Don't over-choose Lambda@Edge for something a lightweight CloudFront Function could do faster and cheaper — the exam sometimes tests recognizing the "lightweight" cue.

7. **TTL and stale content.** A long TTL means faster performance but content can go stale until it expires or is invalidated. This is a caching trade-off question pattern.

8. **Price classes affect performance, not just cost.** Restricting price class excludes edge locations in some regions, which can increase latency for users there.

9. **CloudFront is a global service**, not tied to a single region — this is why its ACM certs live in `us-east-1` and why it appears as a single distribution regardless of origin location.

---

## Exam-style questions

**Q1.** A company hosts a static website on S3 and wants HTTPS with a custom domain, plus low latency for users worldwide. What is the recommended architecture?
- A) Enable S3 static website hosting and use its default endpoint
- B) Place CloudFront in front of the S3 bucket, with an ACM certificate and OAC
- C) Use S3 Transfer Acceleration
- D) Move the site to EC2 with a load balancer

<details><summary>Answer</summary>
**B** — CloudFront adds global edge caching, native HTTPS via ACM, and OAC secures the S3 origin. The S3 website endpoint alone (A) is HTTP-only.
</details>

**Q2.** A Solutions Architect requests an ACM certificate for a CloudFront distribution but the certificate isn't showing up as available to attach. What is the likely cause?
- A) The certificate is in the wrong region — it must be requested in `us-east-1`
- B) CloudFront doesn't support ACM certificates
- C) The distribution needs to be deleted and recreated
- D) ACM certificates take 48 hours to become active

<details><summary>Answer</summary>
**A** — CloudFront only reads ACM certificates from the `us-east-1` region, regardless of where the origin or users are located.
</details>

**Q3.** A media company wants to let a paying user download exactly one specific video file for a limited time, without giving access to any other content. What should they use?
- A) A signed cookie
- B) A signed URL
- C) Geo restriction
- D) An S3 bucket policy

<details><summary>Answer</summary>
**B** — Signed URLs grant time-limited access to a single file. Signed cookies (A) are for granting access to multiple files at once.
</details>

**Q4.** After updating a JavaScript file at the origin, users are still receiving the old cached version through CloudFront. What are two ways to address this? (Choose two)
- A) Create a CloudFront invalidation for the file's path
- B) Wait for the TTL to expire, or use a new versioned file name going forward
- C) Disable the distribution
- D) Change the origin's region

<details><summary>Answer</summary>
**A and B** — an invalidation forces an immediate refresh; alternatively, waiting for TTL expiry or shipping new content under a versioned filename avoids stale cache without an invalidation cost.
</details>

**Q5.** A company needs lightweight, high-scale logic (e.g., redirecting based on a request header) to run at every CloudFront edge location with minimal latency. What should they use?
- A) Lambda@Edge
- B) CloudFront Functions
- C) AWS Lambda (regional)
- D) AWS Step Functions

<details><summary>Answer</summary>
**B** — CloudFront Functions are designed for lightweight, sub-millisecond logic running on every edge location. Lambda@Edge (A) suits heavier logic but with higher latency and fewer edge locations.
</details>

**Q6.** A company wants to ensure its S3-backed CloudFront distribution cannot be bypassed by users hitting the S3 bucket directly. What should they configure?
- A) S3 Transfer Acceleration
- B) Origin Access Control (OAC) with the bucket policy restricting access to CloudFront, plus Block Public Access
- C) A longer TTL on the cache behavior
- D) A signed cookie

<details><summary>Answer</summary>
**B** — OAC restricts the bucket to only be readable via the specific CloudFront distribution, and Block Public Access prevents direct public access to the bucket.
</details>

**Q7.** An application needs to protect a CloudFront-fronted website from SQL injection and cross-site scripting attempts at the edge. What should be attached to the distribution?
- A) AWS Shield Standard
- B) AWS WAF
- C) Amazon GuardDuty
- D) AWS Config

<details><summary>Answer</summary>
**B** — AWS WAF filters malicious requests like SQL injection and XSS and can be attached directly to a CloudFront distribution. Shield (A) protects against DDoS, not application-layer attacks like SQLi/XSS.
</details>

**Q8.** A global company wants to restrict its CloudFront edge locations used to only North America and Europe to control cost, accepting some latency increase for other regions. What should they configure?
- A) Geo restriction
- B) A lower CloudFront price class
- C) Origin Access Control
- D) A shorter TTL

<details><summary>Answer</summary>
**B** — Price classes control which set of edge locations serve the distribution, trading off cost against global performance. Geo restriction (A) blocks/allows viewers by country, it doesn't limit which edge locations are used.
</details>

---

## Related services

- [[S3]] — most common CloudFront origin; OAC secures the bucket behind the distribution
- [[ACM]] — issues the HTTPS certificate CloudFront uses (must be in us-east-1)
- [[Route 53]] — alias records point custom domains to CloudFront distributions
- [[WAF]] — attached to CloudFront to filter malicious web requests
- [[Shield]] — DDoS protection built into CloudFront (Standard) or enhanced (Advanced)
- [[Lambda]] — Lambda@Edge runs edge-triggered functions for heavier request/response logic
- [[EC2]] / [[ALB]] — common custom origins for dynamic content behind CloudFront
- [[VPC]] — origins inside a VPC (e.g. behind an ALB) reached by CloudFront over the public endpoint