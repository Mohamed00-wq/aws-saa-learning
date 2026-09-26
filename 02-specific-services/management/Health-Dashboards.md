# AWS Health Dashboard  Service Health and Personalized Alerts

## Purpose

AWS Health Dashboard gives visibility into the **health of AWS services and your specific account resources**. It has two views  the **Service Health Dashboard** (public, covers all AWS services and all customers) and the **Personal Health Dashboard** (private, shows events affecting **your account's** resources, ARNs, and Regions). For the SAA exam it's the answer for **proactive alerts and AWS-side explanations** when services report issues.

## Main use cases

- **Service Health Dashboard**  public status page (status.aws.amazon.com) to see broad AWS-wide disruptions or maintenance without logging in
- **Personal Health Dashboard**  private view (health.aws.amazon.com) of issues affecting **your resources** with affected resource ARNs and recommended actions
- **Proactive notifications**  get email / chat alerts before or during maintenance events that affect your resources (e.g. EC2 host retirement, RDS patching)
- **Automation and remediation**  feed AWS Health events into **EventBridge** to trigger SNS, Lambda, SQS, or Step Functions
- **Organization-wide visibility**  management account sees health events for all member accounts, optionally with a delegated trusted admin account
- **Audit trail** of AWS-side events via Health API and CloudTrail

## Key features

- **Three event types**  **Issue** (active disruption), **Maintenance** (scheduled work, e.g. host retirement), **Notification** (informational, e.g. security bulletin, deprecation)
- **Event details**  affected Region, service, specific resource ARNs, start/end times, and recommended actions
- **Aggregated events**  multiple related events grouped to reduce noise
- **Integrations**  **EventBridge** (events with source `aws.health`, detail-type `AWS Health Event`), **SNS** (email/SMS/HTTP), **AWS Chatbot** (Slack/Chime/MS Teams), Health API
- **Organization support**  events visible to the **management account** for all member accounts or a delegated administrator
- **Alternate contacts**  add extra email contacts in billing preferences (beyond the default root-user email)
- **Pricing**  free (no charge for events, notifications, or API calls)

## When to use

- Want to know **why AWS-managed resources failed** or when **maintenance will occur** (the distinction from CloudWatch, which shows your metrics/logs)
- Need **real-time automation on AWS-side events** (e.g. rebalance, notify, failover on EC2 host issues) via EventBridge
- **Multi-account orgs** centralizing health visibility in one place
- Exam scenario "**AWS event affected my resource, how do I get notified/API data**" -> Personal Health Dashboard + EventBridge

## Important limitation

- **Service Health Dashboard is generic and public**  it tells you AWS-wide status, not your specific resources (that's the Personal view)
- **Personal Health Dashboard must be enabled/used at health.aws.amazon.com**  it is not on by default as a feed superhighway and **default notifications go to the root user email only**
- **Not the same as CloudWatch**  Health reports **AWS-side events** (service outages, maintenance) while CloudWatch reports your **application/metrics/logs**
- **Organization events are only aggregated in the management account**  member accounts still see their own
- No control over AWS-side maintenance timing (you react, not schedule)

## SAA relevance

- "**Get alerted when AWS plans maintenance or when a service affects my resources**" -> **Personal Health Dashboard** with EventBridge -> SNS/Lambda
- "**Public AWS-wide status page**" -> **Service Health Dashboard** (status.aws.amazon.com)
- "**Centralize health across an organization**" -> management account or **delegated administrator**
- "**Route Health events to automation**" -> **EventBridge** with `aws.health` events + Health API
- Exam trap: **AWS Health = AWS-side event visibility**, **CloudWatch = your telemetry** prefer **EventBridge** (not just SNS) for automation, and remember org events are management-account-scoped.