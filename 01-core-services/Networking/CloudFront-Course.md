# CloudFront Course — Content Delivery Network

## 1. Purpose

CloudFront is AWS's global **Content Delivery Network (CDN)**. It caches content at **edge locations** worldwide so users pull it from the location nearest them instead of from the origin server. Benefits: lower latency, offloaded origin traffic, HTTPS simplification, and built-in DDoS protection (Shield). It's the standard answer for any question mentioning "global users," "low latency," "reduce load on origin," or "HTTPS for a static site."

It front-ends an **origin** — typically an **S3 bucket** (static assets/sites) or an **HTTP origin** (ALB, EC2, on-prem server, or any web server via a custom origin).

## 2. How it works

- A **distribution** is the config object tying an origin to caching/access behavior (one default + extra cache behaviors)
- When a user requests content: nearest **edge location** is checked → **cache hit** (served instantly from edge) or **cache miss** (fetched from origin, cached at edge, served)
- **Cache behaviors** route by path pattern (`/images/*` → S3, `/api/*` → ALB), set TTL, decide which query strings/cookies/headers to forward
- **TTL** controls cache freshness — set per behavior (min/default/max) or via origin `Cache-Control`/`Expires` headers
- **Invalidation** force-discards cached objects; alternatively rename/version files (`app.v2.js`)
- Each distribution gets a default domain (`d111111abcdef8.cloudfront.net`) mapped to your custom domain via **Route 53 alias** + **ACM** cert

```
User → nearest edge → Cache hit? → serve
                     → Cache miss? → fetch from origin → cache at edge → serve
```

## 3. When to use

- **Global/low-latency delivery** of static or dynamic content (media, images, JS, downloads)
- **S3 static website needs HTTPS** — raw S3 website endpoint is HTTP-only; CloudFront + ACM adds HTTPS
- **Offload origin** — cut origin request load and S3 request costs
- **Private content** — signed URLs / signed cookies for premium media or documents
- **Caching dynamic-but-cacheable responses** in front of ALB/EC2 origins
- **Single entry point** for multi-origin apps (static from S3, API from ALB) via path-based behaviors
- **Edge processing** — CloudFront Functions / Lambda@Edge for header injection, URL rewrites, A/B
- **DDoS/WAF defense** at the edge before traffic reaches origin

## 4. When NOT to use

- Content is **user-specific/always dynamic** (checkout, dashboards) with no cacheable fraction — little benefit; an ALB alone may suffice
- You need **country allow/deny for the app itself** — Geo Restriction only limits which edges serve; enforcement must be at the app/WAF too
- Real-time streaming needs — CloudFront is for HTTPS delivery; interactive streaming uses different paths
- You need origin-level access control *without* a CDN — use S3 presigned URLs or direct endpoint access
- Only a small region serves users — a single regional ALB may be simpler/cheaper than global edges

## 5. Important features

- **Origins**: S3 (locked down with **OAC** — Origin Access Control replaces the old OAI) or custom HTTP(S) (ALB, EC2, on-prem); multiple origins per distribution
- **Cache behaviors** — path routing, TTL, forwarded query/cookies/headers, viewer protocol (HTTPS, redirect, or HTTP+HTTPS)
- **Signed URLs** — time-limited access to **one file**; **Signed Cookies** — time-limited access to **many files** without changing URLs
- **OAC** — bucket not public; only that CloudFront distribution can `s3:GetObject`; pair with S3 **Block Public Access**
- **Geo Restriction** — allow/block countries at distribution level (licensing/compliance)
- **Price Classes** — All, 200, 100: limit edge locations for cost (trade-off: latency elsewhere)
- **ACM certificates** must be in **`us-east-1`** (CloudFront is global, reads certs only from N. Virginia)
- **WAF + Shield** integration — filter SQLi/XSS/rate-limit at edge; Standard Shield included, Advanced for enhanced protections
- **CloudFront Functions** (JS only, viewer req/resp, sub-ms) vs **Lambda@Edge** (Node/Python, viewer + origin req/resp, heavier logic, external calls)
- **Route 53 alias** to the distribution (apex domains work; no CNAME limitation)

## 6. Limitations

- **Cache = stale risk** — content stays cached until TTL expires or invalidated; invalidation costs money past free tier
- **Dynamic-only content doesn't benefit** — caching is the engine; can't cache personalized responses
- **Geo Restriction is coarser** than WAF — country-level only, at the edge distribution level
- **S3 website endpoint HTTP-only by default** — HTTPS requires fronting with CloudFront (or ALB)
- **Public bucket + CloudFront = bypass risk** — users can hit S3 directly, skipping caching/HTTPS/WAF (fix with OAC + Block Public Access)
- **Edge compute is limited** — CloudFront Functions are single-purpose/Javascript-only; Lambda@Edge has no VPC access (original design)
- **Certificates region-locked** — forgetting the `us-east-1` requirement is a classic failure
- **Price class = performance trade-off** — restricting edges can raise latency for excluded regions

## 7. Trade-offs

- **CloudFront vs S3 direct** — global speed/HTTPS/security + extra cost vs simple direct serving
- **CloudFront vs regional ALB** — global edges & caching vs regional request-level routing
- **Invalidation vs versioned filenames** — immediate refresh (bills per invalidation) vs rename objects (cheap at scale, needs URL/cache-busting discipline); mostly HTTP-cache-friendly
- **Signed URL vs Signed Cookie** — one file (explicit) vs many files with unmodified URLs (cookie)
- **CloudFront Functions vs Lambda@Edge** — lightweight/super-fast/JS-only vs heavier/flexible/more-latency. Simple → Functions; complex → Lambda@Edge
- **Long TTL vs short TTL** — faster/cheaper but stale vs fresher but more origin round-trips
- **Price Class All vs 100/200** — performance everywhere vs cost (and latency trade-offs)

## 8. Architecture

Reference patterns:

```
Static:  Route 53 → CloudFront (ACM cert, WAF) → S3 (OAC, Block Public Access, private)
                               ↘ HTTPS, edge caching, geo restriction, signed URLs

Dynamic: Route 53 (alias) → CloudFront
              /images/* → S3 (OAC)
              /api/*    → ALB → ASG → DynamoDB
        Offload: TLS termination at edge, cache cacheable responses, pass-through others

Edge compute: CloudFront Functions for header/auth-checks/rewrites
              Lambda@Edge for origin selection or heavier transforms
```

## 9. SAA-C03 Perspective

Top CloudFront exam details — memorizable and high-yield:

- **ACM cert must be in `us-east-1`** — regardless of origin/user region. Frequently tested
- **"S3 static site + HTTPS + custom domain + global low latency"** → CloudFront + S3 + OAC + ACM
- **OAC (not OAI)** — private S3 origin behind CloudFront, keeps bucket non-public
- **Signed URLs vs Signed Cookies vs S3 presigned URLs** — cheap distractor trap
- **CloudFront Functions for lightweight, Lambda@Edge for heavy** edge logic
- **CloudFront vs Global Accelerator** — CDN caching/HTTPS at edge vs anycast static IP routing (no caching)
- **CloudFront vs S3 Transfer Acceleration** — caching all content vs accelerating a single upload endpoint
- **Route 53 alias → CloudFront** at the apex (no CNAME needed)
- **Price classes** for cost optimization questions
- Pairs with **WAF/Shield** for edge security domain answers

Exam trap: "S3-hosted static site needs HTTPS" → CloudFront (the S3 website endpoint alone is **HTTP-only**). "Cache control + custom origin to ALB" → OAC is S3-only; custom origin needs a different setup. "Fast, ultra-light edge logic" → CloudFront Functions, not Lambda@Edge.