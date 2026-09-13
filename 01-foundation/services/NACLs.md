# NACL — Network Access Control List

## What it is

A **stateless** subnet-level firewall that controls traffic in/out of every instance in the subnet. VPC has an optional **default NACL** that allows all in/out — subnets with no explicit NACL use it. Apply the same NACL to multiple subnets; a subnet can have **only one** NACL.

## Rule evaluation

- Rules are **numbered** and evaluated **lowest number first** until a **match**.
- `*` is the implicit **deny-all** catch-all rule — always last, cannot be edited or deleted.
- Missing numbers are allowed (skipped); gaps don't matter.
- Each rule is either **Allow** or **Deny** — explicit Deny is possible (unlike SG).

```
Rules:  100 allow ssh     → match → allow
        200 deny 3306     → match → deny
         *   deny all     → only if nothing matched
```

## Limitations

- **Stateless**: return traffic must be explicitly allowed by the **response rule** (ephemeral ports). Two sets of rules: **Inbound** and **Outbound**.
- **Dropped or Denied traffic is silently dropped** — no rejection back to the client.

## Default NACL vs custom

| | Default NACL | Custom NACL |
|---|---|---|
| Default rules | Allow all in + out (rules 100) | Denies all traffic |
| Catch-all (`*`) | Deny all — never hit by default | Deny all |
| Editing | Only editable with the rules you add first | ... |

## Common gotchas (exam classics)

- NACLs are **stateless** — you must allow ephemeral **outbound/inbound response ports**. Stateless = pairs of rules.
- For HTTP (80) you need inbound 80 **and** outbound high range (49152–65535) to let responses return (or the other way, depending on the direction).
- **Security Groups and NACLs both apply** to an instance: traffic must pass **both** layers (SG stateful allow + NACL allow/deny).
- **Persistence**: blocked traffic — NACLs drop silently, which is useful where "silence" is wanted.

## Exam domains

- [x] **Secure (30%)** — stateless vs stateful, explicit deny, rule ordering
- [x] **Resilient (26%)** — attach NACL to multiple subnets/AZ for DR vs an SG-per-instance
- [x] **High-Performing (24%)** — number ranges, ephemeral ports, rule matching speed
- [x] **Cost-Optimized (20%)** — no cost; shared NACL vs per-instance SG

## Key gotchas

1. NACL is **stateless** — allow both request + response (ephemeral ports)
2. Rules evaluated **lowest number first**; `*` deny-all always last
3. Explicit **Deny** allowed (e.g., block IPs) — SGs have none
4. A subnet can have **only one** NACL, but **shared across many** subnets
5. Changing a NACL affects **all instances** in the subnet at once
6. Default NACL allows everything; **custom NACL denies everything** until you add rules
7. Ephemeral port range 49152–65535 often needs explicit allow for responses
8. No security cost — often used to **block known bad IPs** (deny)
9. Traffic must pass **both NACL and SG** layers
10. **Silent drop** — clients see a timeout, not a rejection

## Related services

- **Security Groups** — stateful instance-level firewall, evaluated after NACL
- **VPC** — subnets the NACL is attached to
- **Route tables** — dictate where traffic can flow within the VPC
- **VPC Flow Logs** — observe bypasses / where NACLs drop traffic