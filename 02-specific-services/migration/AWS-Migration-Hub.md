# AWS Migration Hub — Discover, Plan & Track Migrations in One Place

## Purpose

AWS Migration Hub provides a **single place to track the status of your application migrations to AWS** — whatever tools you use (AWS or partner). It gives you **discovery + planning visibility**: see your existing servers, group them into **applications**, run migration tasks with tools like **AWS Application Migration Service (MGN)** and **AWS Database Migration Service (DMS)**, and watch progress from one dashboard/API. **IMPORTANT (current state): AWS Migration Hub is no longer open to new customers as of November 7, 2025 — AWS points new users to AWS Transform.** For the SAA: Migration Hub = "**one dashboard to track/status migration progress across AWS/partner tools (no per-use cost)**" — and partners offer **Strategy Recommendations, Orchestrator, and Refactor Spaces** for plan/automate/refactor.

## Main use cases

- **Track/triage a migration project at an org or portfolio level** — status of each application (Not started / In progress / Completed) from a **single console/home region**
- **Integrate multiple migration tools** — MGN (lift & shift servers), **DMS** (databases), partner tools via ProgressUpdateStream/API — into one view
- **Discover & plan** — import into **AWS Application Discovery Service** (agent/agentless inventory, OS/comms) so you know what to migrate before moving
- **Strategy & orchestration** — *Strategy Recommendations* (choose rehost/replatform/refactor paths), *Orchestrator* (templated server/app migration workflows), *Refactor Spaces* (incremental application refactoring to microservices)
- **Status reporting for leadership** — centralized, tool-agnostic application migration tracking across accounts

## Key features

- **Central tracking** — Applications + MigrationTasks; states reported by tools (per server/db); dashboards; **no additional AWS charge for the hub itself** (you pay for the tools/resources they use)
- **Tool integration on AWS & beyond** — AWS Application Migration Service (rehost), AWS DMS (databases), SMS/Snowball-era and AWS/partner tools all push status via **ProgressUpdateStream** + **NotifyMigrationTaskState**
- **Discovery** — Application Discovery Service agents/agentless data (server inventory, software, network connections) feeds portfolio planning
- **Companion suites** — **Migration Hub Strategy Recommendations** (7R guidance), **Migration Hub Orchestrator** (automated templated migrations), **Migration Hub Refactor Spaces** (incremental microservice refactor), **AWS Transform** (the 2025 successor)
- **Home Region model** — pick a **home Region** that stores/coordinates your tracking data before any write action; **CloudTrail** audit logging

## When to use

- You're running a **multi-tool migration** (server + database + partner) and want **status in one place**
- You need to **discover your on-prem portfolio** before planning (ADS) and track **per-application progress**
- Use **Strategy Recommendations** to decide 7R approach, **Orchestrator** for repeatable automated migration workflows, **Refactor Spaces** for app-splitting
- (Security/console) — a migration program office/reporting view for execs

## Important limitation

- **Closed to new customers as of November 7, 2025** — new migrations should use **AWS Transform**; existing hub users can keep it but future-facing designs plan around Transform/companion tools. It is a **tracking/orchestration layer, not a mover** — **it does not move data** (that is MGN/DMS/Snow/Transfer etc.), and requires a **Home Region** before any writes, with status only as current as the tools push it. Resources/tools used still bill normally (DMS/MGN instances etc.).

## SAA relevance

- "**Track/status of migrations across AWS + partner tools** in one console" → **AWS Migration Hub** (legacy) / **Transform** (current)
- "**Discover on-prem servers/plan migration**" → **Application Discovery Service** (agent/agentless) → portfolio in Migration Hub
- "**Rehost/lift-and-shift servers**" → **AWS Application Migration Service (MGN)**; "**database migrations**" → **AWS DMS** — both report into Migration Hub
- "**Choose migration strategy (7R)**" → Migration Hub Strategy Recommendations; "**automated migration workflows**" → Orchestrator; "**incremental refactor to microservices**" → Refactor Spaces
- Exam traps: Migration Hub **tracks/plans** (it does NOT move anything); **DMS moves databases, MGN moves servers** and they *feed* the hub; know the **2025 EOL → Transform** current-state fact; home-Region requirement for writes; Migration Hub = tracking, Application Migration Service = actual server replication.