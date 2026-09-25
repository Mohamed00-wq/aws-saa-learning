# Shield Course  AWS Shield (DDoS Protection)

## 1. Purpose

Shield is AWS's **managed DDoS (Distributed Denial of Service) protection**. It shields internet-facing applications from network/transport (L3/L4) and volumetric attacks  SYN floods, reflection/amplification, UDP floods  automatically, and (Advanced tier) adds application-layer (L7) mitigation, expert support, and cost protection. Standard is **free and always-on** Advanced is a paid, org-wide subscription.

## 2. How it works

- **Shield Standard**: automatically enabled for every AWS customer, no cost, no setup. Protects against common L3/L4 attacks targeting AWS edge infrastructure (CloudFront, Route 53, Global Accelerator, Elastic IPs, ALB/ELB).
- **Shield Advanced** ($3,000/month per organization + data-transfer-out fees, **12-month commitment**):
  - Expanded L3/L4 mitigation for your **protected resources**: EC2 Elastic IPs, ELB (ALB/NLB/CLB), CloudFront, Global Accelerator, Route 53 hosted zones.
  - **Application-layer (L7) DDoS protection**  2026: being migrated to the **AWS WAF Anti-DDoS managed rule group** with ML traffic profiling that reacts in seconds (phased auto-upgrade legacy system retires **Jan 1, 2027**).
  - **Shield Response Team (SRT)** 24/7 for attack triage and custom mitigations (requires Business/Enterprise support).
  - **DDoS cost protection**: service credits offset scale-out/data-transfer spikes caused by verified attacks.
  - **Health-based detection** (Route 53 health checks) improves accuracy and unlocks **proactive engagement**.
  - **DDoS attack flow logs** (2026): packet-level details during attacks to S3/CloudWatch Logs/Data Firehose for forensics.

## 3. When to use

- **Standard**: everyone, automatically  no decision needed.
- **Advanced**: internet-facing, revenue-critical apps (e.g. e-commerce, financial, gaming) where a DDoS could cost money or reputation.
- Compliance/legal requirements for DDoS protection and documented mitigation.
- Want **cost protection** against attack-driven bill spikes (auto-scaling + data transfer during a flood).
- Need **SRT 24/7** escalation and proactive engagement.

## 4. When NOT to use

- **Internal-only / private traffic**  no internet exposure, Shield adds nothing.
- Attack is at the **application layer only** (SQLi, XSS, propagated abuse) → that's **WAF**, not Shield (Advanced includes WAF rule group + subscription covers standard WAF fees on protected resources, but the protection mechanism is WAF).
- You need full network firewall/routing control → Network Firewall, NACLs.
- Third-party multi-cloud DDoS (Cloudflare/Akamai)  Shield is AWS-native only.

## 5. Important features

- **Always-on L3/L4 mitigation** at AWS edge, untouched by your architecture.
- **Advanced extras**: SRT, actionable findings/attack diagnostics (describe-attack APIs, global threat dashboard), cost protection, health-based detection/proactive engagement, attack flow logs, **bundled WAF fees** (web ACL + rules + first 50B requests/month on protected resources, ≤1,500 WCUs).
- **Automatic L7 mitigation**: attach managed WAF rule group that auto-scales during DDoS (adds ~150 WCUs to the ACL).

## 6. Limitations

- **Protects AWS-hosted resources only**  no on-prem/other-cloud.
- **$3,000/month + 12-month commitment**  expensive for small/low-risk workloads.
- Advanced covers **registered resources** you must explicitly add protection to each (EIP/ELB/CloudFront/etc.).
- **L3/L4 ≠ L7**: standard tier does nothing at the app layer even Advanced needs WAF rules for app-layer.
- Cost protection requires meeting the subscription terms (verified attacks on protected resources).
- Standard expires after the 30-day free trial of Advanced no free Advanced tier beyond that.

## 7. Trade-offs

- **Standard vs Advanced**: free/automatic vs $3k/mo for L7, SRT, cost protection  pick Advanced only when downtime cost justifies it.
- **Shield Advanced vs CloudFront+WAF only**: WAF at edge handles many L7 abuse cases cheaper Shield adds L3/L4 + bill protection + experts.
- **AWS-native (Shield) vs third-party CDN (Cloudflare/Akamai)**: tight AWS integration vs multi-cloud/full-stack Shield doesn't absorb attacks bound for on-prem.
- **Proactive engagement** requires health checks adds monitoring overhead but speeds detection.

## 8. Architecture

```
Internet ──> AWS edge (Shield Standard absorbs L3/L4 floods, always-on)
                 │
                 ▼
        CloudFront / Global Accelerator / Route 53  ← put edge here
                 │
   Shield Advanced protects the edge resource + ALB/EIP
        WAF web ACL (Anti-DDoS managed rule group) ── L7 auto-mitigation
                 │
                 ▼
             ALB ──> EC2 ASG (origin)
        │
        └─ Route 53 health checks (health-based detection → proactive engagement)
        └─ DDoS attack flow logs → S3 / CloudWatch / Firehose (forensics)
```

- Front everything with CloudFront or Global Accelerator so floods are absorbed at the edge.
- Enable Advanced on the edge resources (CloudFront recommended single point), attach WAF for L7.

## 9. SAA-C03 Perspective

- **Standard = free/automatic Advanced = paid subscription with SRT, cost protection, L7 now via WAF managed rule group.**
- Layer mapping: L3/L4 volumetric/SYN → **Shield** L7 SQLi/XSS/bot → **WAF** "both" → Shield Advanced + WAF.
- "DDoS cost protection" and "Shield Response Team" are Advanced-tier identifiers in scenarios.
- Advanced requires **12-month commitment, $3,000/mo**  small/single-app scenarios likely choose Standard + WAF.
- Know protected resources: EC2 (EIP), ELB, CloudFront, Global Accelerator, Route 53.

Exam trap: "SYN flood / amplification attack" → Shield (not WAF) "SQL injection" → WAF (not Shield).