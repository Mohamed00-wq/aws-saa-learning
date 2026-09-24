# AWS AppSync Course — Fully Managed GraphQL + Real-Time Sync

## 1. Purpose

AWS AppSync is a **fully managed GraphQL API service** that securely queries, aggregates, and updates data from **multiple sources** (DynamoDB, Lambda, relational RDS, HTTP endpoints, OpenSearch) and adds **real-time subscriptions** (WebSocket/MQTT), **offline sync** with **conflict resolution**, and caching — a serverless backend layer for web/mobile applications. For SAA it's the answer to **"GraphQL API / real-time data sync / offline-first mobile app / aggregate multiple backends"**.

## 2. How it works

- **Schema-first** — you define a **GraphQL schema** (types, queries, mutations, subscriptions); **resolvers** (keyed to schema fields — written in VTL or JavaScript) map each operation to a **data source**
- **Data sources** — **DynamoDB, Lambda, Amazon RDS (relational), HTTP (any REST), OpenSearch/Elasticsearch** — AppSync executes resolvers against them and merges results into one response
- **Real-time** — **Subscriptions** over **WebSocket** (GraphQL) or, with **AWS AppSync Events** (added 2024), **MQTT/WebSocket pub-sub channels** for high-scale event fan-out (no schema required)
- **Offline + sync** — client SDKs (Amplify) cache locally and sync when back online, using built-in **conflict resolution** (auto-merge, optimistic concurrency, last-writer-wins)
- **Authorization modes** — per-operation: **API keys**, **IAM**, **Amazon Cognito User Pools** (+ groups, cognito claims), **OIDC**, or a **Lambda authorizer**; fine-grained with resolvers
- **Platform** — **AWS Amplify** integration (codegen, client config), caching layers, CloudWatch + **X-Ray** tracing, audit logs, endpoints in your VPC via Data API/VPC integration, AWS AppSync is serverless

```
Mobile/web app (Amplify) → AppSync GraphQL endpoint (HTTPS + WebSocket/MQTT)
  ├─ resolvers → DynamoDB | Lambda | RDS | HTTP | OpenSearch | EventBridge
  ├─ subscriptions → real-time pushes; offline sync + conflict resolution
  └─ auth: API key | IAM | Cognito | OIDC | Lambda; cached; traced with X-Ray
```

## 3. When to use

- **Mobile/web apps that want GraphQL** — flexible client-driven queries, few-round-tripping, typed schema
- **Real-time data** — chat, live scores/stocks, dashboards, collaborative apps, IoT telemetry (subscriptions/push)
- **Offline-first apps** — clients work offline and sync when connectivity returns (conflict resolution matters)
- **Aggregating many backends behind one API** — GraphQL federates DynamoDB + Lambda + HTTP into a single endpoint
- **Serverless / Amplify stacks** — natural fit with Cognito, Lambda, DynamoDB

## 4. When NOT to use

- **Simple REST CRUD only** → **API Gateway + Lambda** (lighter; REST clients)
- **Backend-to-backend event fan-out (no API for clients)** → **EventBridge / SNS**
- **No need for GraphQL flexibility or real-time** — plain synchronous HTTP is fine
- **Teams not familiar with GraphQL/Javascript resolvers** — the abstraction adds learning curve
- **Pure server/serverless high-GBP synchronous calls best served by Step Functions** (orchestration), not a GraphQL API

## 5. Important features

- **GraphQL** — typed schema, queries, mutations, subscriptions; **data federation** across multiple sources in one request
- **Real-time subscriptions** — WebSocket (and **AppSync Events** MQTT) for pushes — chat/dashboards/live feeds
- **Offline sync & conflict resolution** — auto-merge, optimistic concurrency, last-writer-wins
- **Multiple data sources** — DynamoDB, Lambda, RDS, HTTP, OpenSearch, EventBridge; **VPC data source support**
- **Auth options** — API key, IAM, Cognito (groups & claims), OIDC, Lambda authorizer
- **Caching** (per-resolver), **CloudWatch + X-Ray**, schema auditing, OpenTelemetry-ready
- **AWS Amplify** codegen & client SDKs; serverless — scales automatically, pay-per-request

## 6. Limitations

- **GraphQL complexity** — schema/resolver design effort; VTL-or-JS resolvers add a skill requirement
- **Request/response size limits** (e.g., 1 MiB payload) and resolver execution-time limits
- **Real-time scale** — subscriptions are stateful (WebSocket); AppSync Events adds serverless MQTT path but configuration/limits apply
- Some **data-source-specific** limits; HTTP resolver needs endpoint reachability (or VPC integration)
- Cost at very high request volumes; caching adds cost/complexity to tune

## 7. Trade-offs

- **AppSync vs API Gateway (+Lambda)** — client-driven GraphQL, real-time, offline-sync vs REST/HTTP-first with route-level control (many teams use both)
- **AppSync vs EventBridge/SNS** — the client-facing GraphQL API + sync vs backend event bus / pub-sub (different layers; AppSync Events is the hybrid)
- **Resolvers vs Lambda-everywhere** — resolver-level integration (GraphQL-native) vs writing full Lambda handlers for each operation
- **Online-only vs offline-capable** — offline sync buys UX at the cost of conflict-resolution logic

## 8. Architecture

```
Mobile-first app:
  Amplify client → AppSync (GraphQL, auth via Cognito User Pools)
  ├─ mutations → DynamoDB (primary store)
  ├─ subscriptions (WebSocket/AppSync Events) — live feed updates to all clients
  └─ offline: client mutates locally → sync engine replays → conflict resolution
Admin pipeline: RDS source for relational reads federated in the same query; HTTP → third-party API
Observability: CloudWatch + X-Ray; per-operation caching
```

## 9. SAA-C03 Perspective

- **"GraphQL API / real-time data / offline sync for mobile"** → **AWS AppSync**
- **"WebSocket subscriptions / push updates (chat, live dashboards)"** → AppSync subscriptions / **AppSync Events**
- **"One API aggregating DynamoDB + Lambda + HTTP/RDS"** → AppSync multi-source resolvers
- **"Offline-first app that syncs later"** → AppSync offline sync + conflict resolution
- **"REST CRUD API only"** → **API Gateway**; **"server-side event fan-out"** → **EventBridge/SNS**

Exam traps: "AppSync is just an HTTP API gateway" → **no, GraphQL with resolvers/real-time/offline sync**; "real-time requires polling" → **no, push subscriptions**; "conflict resolution is manual" → **built-in strategies**; "AppSync can only use Lambda" → **no, many data sources incl. DynamoDB/RDS/HTTP/OpenSearch**.