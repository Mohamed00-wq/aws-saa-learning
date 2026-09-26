# AWS Amplify  Full-Stack Web and Mobile Development

## Purpose

AWS Amplify is the **front-end web and mobile development platform** on AWS. It bundles everything needed to go from Git repo to live app: **frontend hosting with CI/CD** (**Amplify Hosting** on the CloudFront global edge network) plus **full-stack backend** (auth, real-time data, storage, functions) defined in **TypeScript** (**Amplify Gen 2** is the current model, code-first). It removes the AWS plumbing so teams can ship web and mobile apps (React, Next.js, Vue, React Native, Flutter, Swift, etc.) without cloud expertise.

## Main use cases

- **Hosting static and server-rendered apps** with Git-based continuous deployment (Next.js, Nuxt, React, Vue, Angular, SPA, static sites)
- **Full-stack app backends** defined in TypeScript: **authentication** (Cognito), **real-time APIs / data** (AppSync GraphQL + DynamoDB), **Lambda functions**, **S3 storage** for files/images
- **CI/CD with environments** per Git branch (dev, staging, prod), **PR previews** before merge
- **Mobile cross-platform apps** (React Native, Flutter, Android, iOS, Swift) sharing one AWS backend
- **Visual UI building** (Amplify Studio, Figma-to-React components, headless CMS to manage data/users)
- **Adding AWS features an app fast** via Amplify Libraries and UI components (auth, storage, pub/sub, predictions, push notifications, analytics)

## Key features

- **Amplify Hosting** managed service: connect a Git repo, **continuous deployment** on every push, custom domains + SSL, **branch-based environments**, PR previews, atomic deploys/cache invalidation, password protection, SSR/SSG/ISR hosting (zero-config Next.js/Nuxt), global edge via **CloudFront**
- **Amplify Gen 2 backend (code-first)**: define `backend.ts` data models, auth, and logic in TypeScript per-developer **cloud sandboxes** deploy frontend + backend together
- **Backend building blocks** work on managed AWS: **Cognito** (auth), **AppSync** (real-time GraphQL), **DynamoDB** (data), **Lambda** (functions), **S3** (storage)
- **Amplify CLI** for project management, **Amplify Libraries** + **UI components**, **Amplify Studio** visual editor and data browser
- **CDK integration** to reach 200+ AWS services **bring your own pipelines** for cross-account/multi-region deploys
- **Pricing**: free tiers and pay-as-you-go (hosting ~$0.15/GB served, ~$0.01/build-min SSR ~$0.20/GB-hour optional WAF $15/app/month). New customers get free-tier credits.

## When to use

- App team wants a **Git-driven full-stack deploy without managing S3/CloudFront/CICD themselves**
- Need rapid **web or mobile app delivery** with auth + real-time data + storage on AWS
- Want a single platform for **backend, hosting, and CI/CD** instead of stitching App Runner/CodePipeline/S3 etc.
- **Best answer when the scenario is "build/deploy a web or mobile app"** or "connect app to AWS backend services"

## Important limitation

- It's **opinionated and frontend-locked**: best when tied to Amplify's supported frameworks/workflows. **Custom/advanced control** (orchestration latency, custom container workloads, non-web backends) is better served by **Elastic Beanstalk, App Runner, ECS, or plain S3+CloudFront**. Backend resources invoke managed services (Cognito/AppSync/DynamoDB/Lambda), so you inherit their limits and **per-use price**. Note **Amplify Gen 1 (CLI/Studio) is in maintenance mode and reaches end of life May 1, 2027** (new projects should use **Gen 2**). Hosting is not for heavy containerized or long-running server workloads.

## SAA relevance

- "**Build, deploy, and host a full-stack web/mobile app** on AWS" -> **AWS Amplify** (Hosting + backend)
- "**Git-based continuous deployment for web apps** with environments and PR previews" -> Amplify Hosting
- "**Frontend app needing auth, API, and storage** without deep AWS setup" -> Amplify (backed by Cognito, AppSync, DynamoDB, S3, Lambda)
- "**SSR app with zero-config deployment**" -> Amplify Hosting (Next.js/Nuxt)
- "**Host a static site**" -> **S3 static website + CloudFront** (a classic SAA pattern, simpler than Amplify when no app CI/CD is needed)
- Exam traps: for **pure static hosting** prefer S3+CloudFront for **full-stack app with CI/CD** use **Amplify** it is not a container platform (Beanstalk/ECS/App Runner for that).