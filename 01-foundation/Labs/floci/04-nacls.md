# Lab 04 — NACLs with Floci

> The stateless subnet-level firewall. Unlike Security Groups, NACLs have **Deny rules** and **rule ordering** — and you must allow return traffic on ephemeral ports.

**Reference notes:** `services/NACLs.md`

## 1. Create a custom NACL

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)

NACL_ID=$(aws ec2 create-network-acl --vpc-id $VPC_ID --query 'NetworkAcl.NetworkAclId' --output text)
echo $NACL_ID
```

> **Custom NACL starts by denying everything.** Only if you add rules does traffic flow.

```bash
aws ec2 describe-network-acls --network-acl-ids $NACL_ID --query 'NetworkAcls[0].Entries'
```

You'll see just the implicit `*` deny-all entries (inbound + outbound).

## 2. Rule ordering — lowest number wins

Add an SSH allow rule with number **100**:

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol tcp \
  --rule-action allow \
  --ingress \
  --cidr-block 0.0.0.0/0 \
  --port-range From=22,To=22
```

Add a Denver-IP block with a **lower** number (so it wins):

```bash
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 90 \
  --protocol tcp \
  --rule-action deny \
  --ingress \
  --cidr-block 203.0.113.0/24 \
  --port-range From=22,To=22
```

**Evaluation:** rule 90 (deny, specific IP) is checked *before* rule 100 (allow all). `203.0.113.x` on SSH → denied. Everything else → allowed.

## 3. Ephemeral ports — the stateless gotcha

For outbound responses to return, add the ephemeral range rule:

```bash
# Outbound: allow responses over ephemeral ports (stateless!)
aws ec2 create-network-acl-entry \
  --network-acl-id $NACL_ID \
  --rule-number 100 \
  --protocol tcp \
  --rule-action allow \
  --egress \
  --cidr-block 0.0.0.0/0 \
  --port-range From=49152,To=65535
```

> **Why this exists:** NACLs are stateless. An inbound SSH request's response travels OUTBOUND on a high (ephemeral) port. Without the outbound rule above, the response is dropped → client hangs → timeout.

## 4. Attach NACL to a subnet

```bash
SUBNET_ID=<your-subnet-id>
aws ec2 associate-network-acl --network-acl-id $NACL_ID --subnet-id $SUBNET_ID
```

> A subnet can have **only one** NACL. One NACL can serve **many** subnets.

## 5. Compare with the default NACL

```bash
DEFAULT_NACL=$(aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" --query 'NetworkAcls[?IsDefault==`true`].NetworkAclId' --output text)
aws ec2 describe-network-acls --network-acl-ids $DEFAULT_NACL --query 'NetworkAcls[0].Entries'
```

Default NACL: rule 100 allow-all inbound + allow-all outbound. The `*` deny never gets hit.

## 6. SG + NACL both apply

Both layers must pass for traffic to reach an instance:

```
NACL (subnet, stateless) → SG (instance, stateful) → Instance
```

Traffic is dropped if **either** layer blocks it.

## Cleanup

```bash
aws ec2 delete-network-acl --network-acl-id $NACL_ID
```

## Exam hooks

- NACL **stateless** — pairs of rules (request + response on ephemeral ports)
- Rules **lowest number first**; `*` deny-all is always last
- Explicit **Deny** — SGs have none
- One NACL per subnet, shared across many

## Gotchas to remember

1. Custom NACL denies everything until rules added
2. Silent drop — clients time out, not get a rejection
3. Typical ephemeral range: 49152–65535
4. Changing a NACL affects **all instances** in the subnet immediately