# Security Groups — Stateful Instance Firewall

## What it is

A **stateful** virtual firewall attached to an **ENI/instance** (never a subnet) that controls inbound and outbound traffic. Every instance **must** have at least one SG; multiple SGs can be attached to one instance, and one SG can be shared across many instances. It's effectively the first network security layer an instance sees.

## Key properties

- **Stateful** — if you allow inbound, the response is automatically allowed, no explicit return rule needed.
- **No Deny rules** — only Allow. Anything not explicitly allowed is silently dropped.
- **All rules evaluated** — unlike NACL, there's no ordering; all Allow rules are additive.
- Can reference **other SGs** as source/destination (not just IPs) — "allow 3306 from sg-app".

## Inbound vs outbound

| | Inbound | Outbound |
|---|---|---|
| Scope | Traffic coming **in** to instance | Traffic going **out** from instance |
| Uses | Allow ports/protocols/IPs | Allow internet/EIP/targets |
| Default (custom SG) | Deny all | **Allow all** |

- If you create a new (non-default) SG: **inbound is denied by default**, **outbound is allowed by default**.

## Default SG behavior

- VPC's default SG allows **all inbound from itself** (same group) and all outbound.
- Custom SGs start with no inbound rules → lock down.

## Common patterns

- **Port ranges + protocols**: e.g. inbound `TCP 22 (SSH) from 0.0.0.0/0`, `TCP 80/443 from 0.0.0.0/0`.
- **SG → SG references**: allow web tier to talk to app tier without IPs; when instances scale, the reference follows them.
- **Source as SG** is evaluated logically (any source matching the referenced group), enabling dynamic security without changing rules.

## Persistence & limitations

- **Stateful** → no ephemeral-port rules needed for the return traffic.
- **Can't block specific IPs** → for that you need a NACL deny (or WAF).
- Can reference up to 5 SGs per resource (60 rules per SG).
- Attaching an SG to *multiple instances* affects all of them; changing a rule affects all attached instances immediately.

## Exam domains

- [x] **Secure (30%)** — stateful vs stateless, SG references, limiting to required ports
- [x] **Resilient (26%)** — SG references for ASG/scale groups, no single point of failure
- [x] **High-Performing (24%)** — immediate propagation of rule changes, no re-provisioning
- [x] **Cost-Optimized (20%)** — no charge; minimal SGs per instance vs NACL per subnet

## Key gotchas

1. SGs are **stateful** — responses auto-allowed; NACLs are stateless
2. **No Deny rules** — explicit deny is a NACL job
3. **Default outbound allows all** in a custom SG; default inbound denies
4. Deleting a SG **fails if still referenced** by another SG's rule
5. One instance **can have multiple SGs** (each is additive)
6. Rule changes apply **immediately** to all attached instances
7. Can reference **SG as source** — best practice for tier-to-tier traffic
8. Exceeding 5 SG per interface / 60 rules → attach fewer, wider SGs
9. Nothing blocked with a Deny — test outbound for what's allowed
10. **Ephemeral ports** handled automatically (stateful) — contrast with NACL

## Related services

- **NACL** — stateless subnet firewall for explicit deny + subnet-wide control
- **VPC** — subnets you attach the SGs and NACLs to
- **ELB** — SG on target instances controls what the ALB can reach
- **ASG** — SGs referenced for scaling tiers
- **VPC Flow Logs** — observe SG accepts/denies
- **WAF** — for protocol-layer blocking SGs can't do