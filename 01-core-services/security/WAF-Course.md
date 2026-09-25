# WAF Course  AWS Web Application Firewall

## 1. Purpose

WAF protects web applications by **filtering and monitoring HTTP(S) traffic** at the application layer (L7). It blocks common web exploits (SQL injection, XSS), bot traffic, malformed requests, and abusive clients (rate limiting) before they reach your origin. Pairs with Shield (network layer DDoS) to form the edge security stack.

## 2. How it works

- You define a **web ACL (protection pack)**  a set of **rules**  and associate it with protected resources (CloudFront, ALB, API Gateway REST/HTTP, App Runner, Global Accelerator, AppSync, Cognito user pools, Verified Access).
- **Rule** = statement (what to match) + action **rule group** = reusable set of rules added to a web ACL.
- Web ACLs are evaluated against each request and apply an **action**:
  - **Allow / Block**  allow or stop the request
  - **Count**  observe/count without blocking (used for testing)
  - **CAPTCHA / Challenge**  interrupt non-human/clients: send puzzle or browser check, issue a token token-carrying requests pass.
- Each web ACL consumes **Web ACL Capacity Units (WCUs)**  max **5,000 WCUs** per ACL  so rule groups have a capacity cost.
- Requests matched are aggregated per **aggregation key** (IP, header, query string, etc.) for rate-based behavior.

## 3. When to use

- Web apps behind CloudFront/ALB needing **SQLi/XSS** protection.
- **Rate limiting** (per IP/session), **geo-blocking**, IP allow/deny lists.
- **Bot management**: block scrapers, credential-stuffing, and scripted clients (Bot Control).
- **Testing/tuning** new protections before enforcement (Count mode, migration window: ECDNE)?
- Central, org-wide policy via **AWS Firewall Manager**.

## 4. When NOT to use

- **Network/transport (L3/L4) DDoS**  WAF is app layer only use **Shield** (Standard/Advanced).
- Protecting non-HTTP(S) services (TCP/UDP, internal APIs)  WAF can't inspect those.
- Full network firewall rules, NAT, routing  that's Network Firewall / NACL / SG territory.
- Massive volumetric floods at the network edge  Shield/Global Accelerator + scale architecture beat WAF for that.
- Simple IP access control with SG/NACL if you don't need app-layer rules.

## 5. Important features

- **Managed rule groups** (AWS Managed Rules)  e.g. `AWSManagedRulesCommonRuleSet`, SQL, XSS, IP reputation, `AWSManagedRulesBotControlRuleSet`, Admin Protection, Known Bad Inputs. Versioned set to **Count** to test before enforcing.
- **Rate-based rules** with scope-down + aggregation keys (configurable threshold) Bot Control targeted rules give lower-latency, ML-based dynamic rate limiting.
- **Rule actions**: Allow, Block, Count, CAPTCHA, Challenge.
- **Labels** + scope-down statements to filter matching scope.
- **IP sets / regex sets / GEO match** (country it operates in).
- Marketplace rule groups (e.g. partner AI/API security rule groups, 2026 trend).
- Integration with **Shield Advanced** (auto application-layer mitigation adds a managed rule group) and **Firewall Manager** (org-wide).
- Logging to S3, CloudWatch Logs, Data Firehose metrics in CloudWatch visualization via Athena.

## 6. Limitations

- **L7 only**  cannot mitigate IP-level floods or network-layer attacks.
- **5,000 WCU cap** per web ACL  heavy managed groups can crowd out custom rules.
- **Region-scoped**, except CloudFront (global edge) where the ACL is tied to the distribution.
- Pricing: per web ACL + per rule + per **million requests inspected**  costs rise with traffic Bot Control / CAPTCHA billed separately.
- Rate-based rules act on request *counts*, not per-window burst precision aggressive thresholds need tuning to avoid false positives.
- No memory/persistence of client state beyond token-based mechanisms.

## 7. Trade-offs

- **WAF vs Shield**: L7 inspection/weak filtering vs L3/L4 DDoS absorption  use both together Shield Advanced even bundles WAF use.
- **Managed vs custom rules**: fast/safe baseline vs exact fits managed groups cost capacity and can misfire → run in Count first.
- **CAPTCHA/Challenge vs Block**: block = simple/firm but friction & collateral (IP sharing) challenge = verify humans without locking out.
- **CloudFront-WAF vs ALB-WAF**: edge (global, cache, absorbs floods) vs regional (lower one-hop latency, still inspects at origin edge).
- **WAF inspections vs raw throughput**: every rule adds latency + per-request cost  keep ACLs lean.

## 8. Architecture

```
Internet
   │
   ▼
CloudFront / Global Accelerator ──> WAF web ACL (rules: managed groups, Bot Control,
   │                                  rate-based, geo/IP sets, CAPTCHA/Challenge)
   ▼
ALB
   │
   ▼
EC2 ASG / Lambda (origin, protected only by SG → WAF is the L7 filter in front)
   └── Logs/WAF → S3 + Athena (analytics), CloudWatch (metrics/alarms)
```

- Put WAF at the edge (CloudFront) to filter before origin.
- Enable **Shield Advanced** automatic application-layer mitigation to auto-scale rules during attacks.
- Use **Firewall Manager** to enforce the same ACL across the organization.

## 9. SAA-C03 Perspective

- Know which resources can be WAF-associated (CloudFront, ALB, API Gateway, Global Accelerator, App Runner)  scenario answers hinge on this.
- Common protections: **SQLi/XSS, rate limiting (DDoS-at-L7), geo blocking, bot control**.
- Web ACL → rules/rule groups → actions (Allow/Block/Count/CAPTCHA/Challenge) is the mental model.
- **WAF vs Shield** split: L7 web exploits → WAF network/volume DDoS → Shield. "Application/HTTP layer attack" = WAF "SYN flood/volumetric" = Shield.
- New console naming: web ACLs are also called **protection packs** capacity = **WCU**.

Frequent exam trap: a DDoS scenario at the network (L3/L4) layer with "distributed volumetric flood"  the answer is Shield, not WAF.