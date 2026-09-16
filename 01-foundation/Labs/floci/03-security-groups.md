# Lab 03 — Security Groups with Floci

> Create a stateful instance-level firewall, test its allow-only behavior, and see that responses come through automatically (stateful).

**Reference notes:** `services/Security-Groups.md`

## 1. Create a Security Group

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=lab-vpc" --query 'Vpcs[0].VpcId' --output text 2>/dev/null || aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)

SG_ID=$(aws ec2 create-security-group \
  --group-name lab-sg \
  --description "Lab security group" \
  --vpc-id $VPC_ID \
  --query 'GroupId' --output text)
echo $SG_ID
```

## 2. Default behavior — deny all inbound

```bash
aws ec2 describe-security-groups --group-ids $SG_ID --query 'SecurityGroups[0].IpPermissions'
# empty → no inbound allowed
aws ec2 describe-security-groups --group-ids $SG_ID --query 'SecurityGroups[0].IpPermissionsEgress'
# allows all by default
```

> **Custom SG defaults:** inbound = deny all, outbound = allow all. An empty SG blocks everything inbound.

## 3. Add inbound rules

Allow SSH (port 22):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 22 --cidr 0.0.0.0/0
```

Allow HTTP from a specific CIDR (exam favorite — least privilege):

```bash
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 80 --cidr 10.0.0.0/16
```

## 4. SG referencing another SG (the real exam answer)

Allow the app tier (another SG) to reach MySQL on 3306 — without IPs:

```bash
APP_SG=$(aws ec2 create-security-group --group-name app-sg --description "app tier" --vpc-id $VPC_ID --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp --port 3306 \
  --source-group $APP_SG
```

**Verify:**

```bash
aws ec2 describe-security-groups --group-ids $SG_ID --query 'SecurityGroups[0].IpPermissions'
```

> Notice the `UserIdGroupPairs` entry — a reference to `app-sg`, not an IP range. When instances scale in/out, the reference follows them automatically. No rule updates needed.

## 5. Statefulness — prove it

Test the stateful behavior (requires an instance; attach SG at launch):

```bash
# Launch instance with the SG (see lab 01 for key/AMI)
aws ec2 run-instances \
  --image-id ami-00000000 \
  --instance-type t2.micro \
  --security-group-ids $SG_ID \
  --subnet-id <subnet-id>
```

With SG stateful:
- Inbound `TCP 22` allowed → the return traffic (SYN-ACK, packets from server back to client) is **auto-allowed**.
- You do **not** add a rule for the response.

> NACLs (next lab) are the opposite — stateless — and you'd need ephemeral-port rules for the return traffic.

## 6. Add rule to an existing SG (applies immediately)

```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr 0.0.0.0/0
# takes effect immediately on ALL instances using this SG
```

## 7. No Deny rules — only Allow

```bash
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 8080 --cidr 203.0.113.0/24
```

Try to *deny* a specific IP:

```bash
# There is no "deny" flag — SGs only have Allow rules.
# To block a specific bad IP you need a NACL deny rule (lab 04).
```

## Cleanup

```bash
aws ec2 delete-security-group --group-id $APP_SG
aws ec2 delete-security-group --group-id $SG_ID
```

> A SG **cannot be deleted while another SG references it** — delete `app-sg`'s source-group deps first, or you'll get `DependencyViolation`.

## Exam hooks

- SGs **stateful** — responses auto-allowed
- **No Deny rules** — only Allow; anything else silently dropped
- Default outbound allows all; default inbound denies
- Can reference **SG as source** — best practice for tier-to-tier traffic
- Delete fails if still referenced

## Gotchas to remember

1. SG attached to ENI/instance — never to a subnet (that's NACL's job)
2. All rules evaluated — no ordering (unlike NACL)
3. Rule changes propagate instantly
4. Max 5 default per interface, 60 rules per SG
5. Can't block specific IPs with SGs — use NACL deny (or WAF)