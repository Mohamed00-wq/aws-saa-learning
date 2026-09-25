# AWS Health Dashboard  Service Health & Personalized Alerts

## What it is

AWS Health Dashboard provides visibility into the health of AWS services and your specific resources. The **Service Health Dashboard** (public, all services) and **Personal Health Dashboard** (private, your account's resources). For the SAA exam, Personal Health Dashboard is the answer for proactive alerts about AWS events affecting your resources.

## Service Health Dashboard vs Personal Health Dashboard

| | Service Health Dashboard | Personal Health Dashboard |
|---|---|---|
| Visibility | All AWS services, all customers | Only your account's resources |
| Content | Global disruptions, maintenance | Events affecting your resources |
| URL | status.aws.amazon.com | health.aws.amazon.com |
| Personalized | No | Yes |

## Event types

| Type | What it means | Example |
|---|---|---|
| **Issue** | Active disruption | EC2 host hardware failure |
| **Maintenance** | Scheduled maintenance | RDS patching, EC2 host retirement |
| **Notification** | Informational, no impact | Security bulletin, deprecated API |

## Event details

- **Service** affected, **Region** impacted, **Affected resource ARNs**, **Start/end time**, **Description** with recommended actions.
- **Aggregated events**: multiple related events grouped into one summary  reduces noise.

## Integrations

| Integration | How it works |
|---|---|
| **EventBridge** | Health events → route to SNS, Lambda, SQS, Step Functions |
| **SNS** | Subscribe → email, SMS, HTTP notifications |
| **Organizations** | Organization-level events visible to management account |
| **AWS Chatbot** | Notifications → Slack/Chime channels |

```json
{ "source": ["aws.health"], "detail-type": ["AWS Health Event"] }
```

## Proactive notifications

- Health Dashboard sends **email to the account root user** by default.
- Add **alternate contacts** in billing preferences. Use EventBridge for custom automation.

## Multi-account & organizations

- **Management account** sees health events for **all member accounts**.
- Delegate to a **trusted administrator account** for centralized monitoring.

## Pricing

AWS Health Dashboard is **free**  no charge for events, notifications, or API calls.

## Exam domains

- [x] **Secure (30%)**  organization-level visibility, alternate contacts
- [x] **Resilient (26%)**  proactive maintenance alerts, auto-remediation via EventBridge + Lambda
- [x] **High-Performing (24%)**  aggregated events, real-time API access
- [ ] **Cost-Optimized (20%)**  free service, no cost considerations

## Key gotchas

1. **Personal Health Dashboard requires activation**  go to health.aws.amazon.com and enable it
2. **Root user gets emails by default**  add alternate contacts or use EventBridge to actually get notified
3. **Service Health Dashboard is public and generic**  doesn't show your specific affected resources
4. **Organization events only visible in management account**  members see only their own
5. **Health ≠ CloudWatch**  Health shows AWS-side events CloudWatch shows your metrics/logs
6. **EventBridge is the preferred integration**  more flexible than SNS for automation

## Related services

- **CloudWatch**  operational metrics Health explains when AWS infrastructure is the cause
- **EventBridge**  route Health events to automation workflows
- **SNS  email/SMS** notifications for Health events
- **ASG**  Health events can trigger scaling via EventBridge
