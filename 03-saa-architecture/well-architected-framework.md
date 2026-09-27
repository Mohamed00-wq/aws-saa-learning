# AWS Well-Architected Framework

## 1. Purpose

What problem does this concept solve? What is the main goal of using this concept in a cloud architecture?

The AWS Well-Architected Framework (WAF) is a **set of best practices for designing and operating workloads in the cloud**, distilled from thousands of real customer architectures. It answers the question "how do I know if my cloud architecture is actually good?" The goal is a consistent way to evaluate an architecture against proven principles, find risks, and improve before problems happen. It is the **umbrella concept** over every other course in this folder, the six pillars of operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability each map to a design area of the SAA exam.

## 2. How It Works

Explain the concept in simple terms.

- What happens?
  - You compare your architecture against best practices organized into **six pillars**. Each pillar has design principles, best practices, and review questions.
  - You run a **Well-Architected Review** (using the AWS Well-Architected Tool) that scores how aligned each pillar is and flags high- and medium-risk items.
  - You create a remediation plan for the highest-risk findings and re-review after changes.
- Why does it work?
  - Because problems are found **proactively by design review** instead of reactively in production. The tool gives a neutral, consistent, repeatable checklist that any team can run.
- What is the main idea behind it?
  - An architecture is judged on six qualities, not just "does it work?" Trade-offs between pillars are explicit and informed. The general design principles apply across all pillars, stop guessing capacity, test systems at production scale, automate, allow for evolutionary architectures, drive architectures using data, and improve through game days.

The six pillars and their focus:

| Pillar | Core focus | Related course in this folder |
|---|---|---|
| Operational Excellence | Run, monitor, improve | `operational-excellence.md` |
| Security | Protect data, systems, assets | `security.md` |
| Reliability | Recover, perform correctly, meet demand | `high-availability.md`, `fault-tolerance.md`, `disaster-recovery.md` |
| Performance Efficiency | Use resources efficiently as demand changes | `performance.md`, `scalability.md`, `elasticity.md` |
| Cost Optimization | Minimize cost, maximize value | `cost-optimization.md` |
| Sustainability | Minimize environmental impact and energy use | `sustainability.md` |

## 3. Trade-offs

What do you gain?
- A **proven, repeatable review process** that catches weak spots before they cause outages or bill shock
- **Consistent scoring and language** across teams so architecture debates become evidence-based
- **Prioritized action plan** (high versus medium risk) so teams fix the important things first
- Direct **exam alignment**, WAF concepts and the review tool appear in SAA questions

What do you sacrifice?
- **Review effort**  a full Well-Architected Review takes time and honest answers from the team
- **Scoring is guidance, not a guarantee**  a "well-architected" score does not mean the system cannot fail
- **Trade-offs must still be made**  pillars conflict (security vs cost, reliability vs performance), the framework does not decide for you
- **Tooling and process overhead**  reviews, remediation tracking, and re-reviews require discipline to keep alive

## 4. AWS Services That Work With This Concept

- **AWS Well-Architected Tool**  free tool (Region-specific, `awswaf:` prefix) to perform and score reviews, track improvements, and run custom defined risks and milestones
- **AWS Well-Architected Lens**  specialized versions for common patterns (Serverless Lens, SaaS Lens, HPC Lens, IoT Lens, hybrid networking, financial services)
- **AWS Trusted Advisor**  automated best-practice checks aligned with the framework (cost, performance, security, fault tolerance, service limits)
- **AWS Health Dashboard**  AWS-side events and maintenance for operational excellence and reliability planning
- **AWS Sustainability / Customer Carbon Footprint Tool**  sustainability pillar data and measurement
- **AWS Cloud Adoption Framework (CAF)**  organizational readiness that complements WAF
- Every AWS service course in this repo ties back to one or more pillars

## 5. When to Use

Use this concept when:
- You are **designing a new workload** and want to avoid rework later
- You are **reviewing an existing workload** before production or after major changes
- You need a **prioritized list of risks and fixes** (high vs medium risk findings)
- Compliance or internal governance requires documented architecture reviews
- Exam questions mention "Well-Architected," "review your architecture," "AWS Well-Architected Tool," "pillars," or "best practice framework," then the answer is almost always related to WAF and its six pillars

## 6. When NOT to Use

Avoid or reconsider this concept when:
- The question is about a **specific service behavior** (pick the service course, not the framework)
- A prototype or lab has no production risk, a full formal review is overkill, quick checklist review is enough
- The "best answer" is clearly a **direct mitigation** (e.g. add Multi-AZ, enable encryption) rather than "run a review"
- You have no way to act on findings, a review without remediation just consumes effort
- The scenario is really about a specialized lens area, then prefer that lens (e.g. Serverless Lens) over the generic framework