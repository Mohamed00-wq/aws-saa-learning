# API Gateway Course  Managed API Front Door

## 1. Purpose

API Gateway is AWS's **fully managed front door for APIs**  it accepts, secures, throttles, caches, transforms, and monitors hundreds of thousands of concurrent API calls, then routes them to backends (Lambda, EC2, ECS, HTTP, other AWS services). It handles **traffic management, authorization, CORS, throttling, versioning, and observability** so you don't build a custom proxy. It's the standard entry point for **serverless and microservice APIs**.

## 2. How it works

- You create an **API**, define **resources/methods** (or routes), and attach **integrations** to backends
- Clients call the **invoke URL** API Gateway routes by stage/route to the target
- Three API types:
  - **REST APIs**  full-featured: API keys, usage plans, **caching**, request/response mapping, WAF, API monetization, SDK generation, request validators
  - **HTTP APIs**  simpler/cheaper/faster subset: OAuth/JWT auth, lower cost & latency, no API keys/usage plans/mapping templates/caching
  - **WebSocket APIs**  persistent **bidirectional** connections for real-time (chat, live dashboards) routes `$connect`/`$disconnect`/messages to Lambda/Kinesis/HTTP
- **Integrations**: Lambda proxy, Lambda (non-proxy + **mapping templates**), HTTP/HTTP-proxy, **AWS service** (e.g., S3, DynamoDB, SQS), Mock, VPC Link (private HTTP to ALB/NLB/Cloud Map)
- **Throttling** via **token-bucket** algorithm → over limit returns **429 Too Many Requests** applies per account (region), per stage/method/route, per client (usage plans + API keys)
- **Caching** (REST only)  provisioned cache at stage level (GB), serves duplicate responses for a TTL, cuts backend load/latency

```
Client → API Gateway (authN/Z, throttle, cache, transform, WAF)
            ├─ Lambda  ├─ HTTP/ALB (VPC Link)  ├─ AWS service  └─ other
        CloudWatch metrics/logs/X-Ray 429 on throttle usage plans + API keys
WebSocket: $connect / messages / $disconnect → routes → Lambda/Kinesis/HTTP
```

## 3. When to use

- **Serverless API front door**  Lambda + API Gateway (most common serverless pattern)
- **Microservice ingress**  unified entry with auth, throttling, versioning across services
- **Private backends in VPC**  **VPC Link** to ALB/NLB/Cloud Map without public load balancer
- **Monetize / meter APIs**  **usage plans + API keys** + quotas per consumer (REST)
- **Reduce backend load/latency**  **caching** responses (REST)
- **Real-time bidirectional apps**  **WebSocket** (chat, notifications, live feeds, collaborative editing)
- **Edge-ish security**  WAF integration, Lambda/Cognito authorizers, JWT/OIDC (HTTP APIs)
- **Cross-origin browser apps**  **CORS** handled at gateway
- **Transform requests**  mapping templates reshape payloads to backend format (REST)

## 4. When NOT to use

- **Simplest cheapest proxy, no keys/cache/mapping** → **HTTP APIs** (not full REST)  lower cost + latency
- **Need API keys, usage plans, caching, WAF, request validators, SDK gen, monetization** → **REST APIs**
- **Long-lived bidirectional** → **WebSocket APIs** (not REST/HTTP)
- **Pure static content / CDN delivery** → CloudFront (+S3/ALB origin), API Gateway not needed
- **Service-to-service inside VPC with no external exposure** → ALB/private ALB or service mesh may suffice
- **Very high-throughput, low-cost passthrough with no API features** → ALB/NLB directly (cheaper at massive scale)
- **GraphQL** → **AWS AppSync**, not API Gateway
- **Legacy SOAP / complex backend orchestration** → may need more than gateway routing (or REST mapping + backend)
- **WebSocket >2h connections**  connections cap at **2 hours**, must reconnect (not for indefinite streams consider AppSync/Kinesis)

## 5. Important features

- **REST vs HTTP**  REST: full features (API keys, usage plans, **caching**, mapping templates, WAF, validators, request validation, SDK generation, API monetization) HTTP: cheaper, faster, OAuth/JWT authorizers only, no keys/cache/mapping
- **Throttling**  **token-bucket** hierarchy: usage-plan client limits → per-method → **account per Region** → AWS regional **429 Too Many Requests** burst = max concurrent before 429 account rate limit raisable via support
- **Usage plans + API keys**  per-consumer **rate + quota** (requests/day etc.) key identifies client can't exceed account limits
- **Caching (REST)**  provisioned **stage cache (GB)** TTL-served responses reduces backend calls + latency **cache disabled by default**
- **Authorization**  IAM (SigV4), **Lambda authorizers** (token/request), **Cognito User Pools**, **JWT/OIDC authorizers** (HTTP APIs) **VPC Link** for private backends
- **WebSocket**  routes (`$connect`, `$disconnect`, custom, `$default`), routes to Lambda/Kinesis/HTTP **connection management** via `@connections` API **500 new conns/s/account**, **2-hour max connection**, **10-min idle timeout**, **32 KB frame / 128 KB message**
- **Integrations**  Lambda proxy/non-proxy, HTTP proxy, **AWS service** integration (direct to S3/DynamoDB/etc.), Mock, **VPC Link → ALB/NLB/Cloud Map**
- **Mapping templates / request validation**  transform payload (JSON/XML), validate requests (REST)
- **CORS, custom domain names, API versioning/stages, canary deployments, request/response models (OpenAPI import/export)**
- **Monitoring**  CloudWatch metrics (`Count`, `4XX`, `5XX`, `Latency`, `IntegrationLatency`), access logs, **X-Ray** tracing, WAF integration
- **Edge-optimized vs regional vs private** endpoints (REST)  edge uses CloudFront for global latency

## 6. Limitations

- **29-second max integration timeout** (all API types)  long jobs must be async (kick off Step Functions/queue, return 202)
- **Payload limits**  REST/HTTP request payload **10 MB** WebSocket message **128 KB**, frame **32 KB**
- **Caching is REST-only**  HTTP APIs can't cache
- **Usage plans/API keys are REST-only**  not on HTTP APIs
- **WebSocket connections ≤2 hours**, idle timeout **10 min**, 500 new conns/s/account (raisable) must handle reconnect
- **Account throttling is a target, not a hard ceiling** (best-effort) over → **429**
- **HTTP APIs lack** mapping templates, request validation, WAF, API keys, request validators, edge-optimized
- **Not a queue/bus**  synchronous proxy only slow backends need SQS/async pattern
- **Cost at scale**  per-request pricing + cache hours + data transfer ALB cheaper for pure high-volume passthrough
- **WebSocket isn't a general stream**  no replay/ordering guarantees like Kinesis consider AppSync for GraphQL realtime
- **VPC Link still needs an NLB/ALB**  extra component to manage

## 7. Trade-offs

- **REST vs HTTP APIs**  full features (keys/cache/WAF/mapping/monetization) vs **lower cost + latency** for simple Lambda/OAuth front doors (choose HTTP when features unneeded)
- **API Gateway vs ALB**  rich API features, auth, per-consumer throttling, serverless-friendly vs cheaper raw L7 load balancing for high-throughput microservices
- **API Gateway vs CloudFront**  API logic/auth/throttle vs static/edge caching of content (often **CloudFront → API Gateway** together for global APIs)
- **Edge-optimized vs regional**  global latency via CloudFront vs data stays in region (compliance)
- **Caching on vs off**  less backend load/cost/latency vs stale responses + cache-hour cost
- **Usage plans (keys) vs Lambda/Cognito auth**  metering/quota per consumer vs user identity/authorization (keys ≠ security)
- **WebSocket vs AppSync**  custom routes/messaging vs managed GraphQL subscriptions realtime
- **29s sync vs async+202**  simple flows vs long-running (queue/Step Functions then callback)
- **VPC Link vs public ALB**  private backend without public ALB vs simpler public origin

## 8. Architecture

Reference API patterns:

```
Serverless public API:
  clients → API Gateway (REST/HTTP, Cognito/JWT authorizer, throttle, cache)
              → Lambda (proxy) → DynamoDB / S3
  CloudWatch alarms on 4XX/5XX/latency usage plans per partner API key

Private microservice API:
  clients → API Gateway → VPC Link → internal ALB → ECS services (no public ALB)

Real-time:
  browser/app → WebSocket API ($connect/messages) → Lambda → broadcast via @connections
              (or Kinesis for ordered fan-out)

Global API:
  Route53/CloudFront (edge) → API Gateway (regional) → Lambda/ALB
  long jobs: API → SQS/Step Functions → async worker → (optional callback)
```

## 9. SAA-C03 Perspective

API Gateway appears in **serverless, performance, security, and cost** scenarios (Domains 1, 3, 4):

- **"Front door for Lambda / serverless API"** → **API Gateway + Lambda**
- **"Meter & throttle per API consumer, quotas, monetize APIs"** → **REST API + usage plans + API keys**
- **"Cache API responses to cut backend load/cost"** → **REST API stage caching**
- **"Cheaper/faster simple API, OAuth/JWT only"** → **HTTP APIs**
- **"Real-time bidirectional (chat/live updates)"** → **WebSocket APIs**
- **"Private backend in VPC without public LB"** → **VPC Link → ALB/NLB**
- **Throttling → HTTP 429** raise **account rate limit** via support hierarchy client → method → account → AWS
- **29-second timeout** → async pattern (SQS/Step Functions + 202)
- **WAF + Lambda/Cognito authorizers** for security layering **X-Ray** for tracing
- **vs AppSync** (GraphQL) and **vs CloudFront/ALB** (feature vs cost vs static)

Exam traps: "29-second max integration timeout"  long-running must be async "cache is REST-only, not HTTP" "API keys are for metering, not authentication" "WebSocket connections max 2 hours, 10-min idle, 128 KB messages" "429 = throttled, raise account limit" "HTTP API = lower cost when you don't need usage plans/caching".