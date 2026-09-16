# Lab 11 — Route Tables with Floci

> The routing decision-maker of the VPC. Longest-prefix-match evaluation, main vs custom tables, and the local route that always wins.

**Reference notes:** `services/Route-tables.md`, `services/VPC.md`

## 1. Create VPC + subnets

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)

PUB_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PRI_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
```

## 2. The implicit local route

```bash
MAIN_RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" --query 'RouteTables[0].RouteTableId' --output text)

aws ec2 describe-route-tables --route-table-ids $MAIN_RT --query 'RouteTables[0].Routes'
```

You'll see exactly one route:

```
Destination: 10.0.0.0/16   Target: local
```

> The **local route is automatic**, cannot be deleted, and covers the whole VPC CIDR. Any VPC-internal traffic matches it (most-specific) and never leaves the VPC.

## 3. Main route table — the silent default

Both new subnets are implicitly associated with the main table (you didn't create custom ones yet):

```bash
aws ec2 describe-route-tables --route-table-ids $MAIN_RT --query 'RouteTables[0].Associations[*].SubnetId'
```

> **Gotcha:** any subnet you create without an explicit association silently uses the main route table. That's how misconfigurations happen — traffic goes somewhere you didn't intend.

## 4. Longest prefix match — create two competing routes

```bash
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

aws ec2 create-route --route-table-id $MAIN_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 describe-route-tables --route-table-ids $MAIN_RT --query 'RouteTables[0].Routes'
```

Now two routes: `10.0.0.0/16 → local` and `0.0.0.0/0 → igw-...`.

- Traffic to `10.0.1.5` → matches `10.0.0.0/16` (more specific) → **stays in VPC** (`local`)
- Traffic to `8.8.8.8` → matches `0.0.0.0/0` → **goes to IGW**

> **Longest prefix wins.** `/0` is lowest priority; specific CIDRs always override it. No first-match ordering (unlike NACL).

## 5. Custom route tables — 1:1 association rule

```bash
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUB
```

**Rules:**
- A subnet has **exactly one** route table
- A route table can serve **many** subnets

```bash
# Prove 1:1 — try to associate a second table to the same subnet
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PRI_SUB >/dev/null
# then try another
aws ec2 associate-route-table --route-table-id $MAIN_RT --subnet-id $PRI_SUB
```

## 6. Gateway Endpoint routes — prefix lists are most specific

```bash
# In floci-compatible installs:
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --service-name com.amazonaws.us-east-1.s3 \
  --route-table-ids $PUB_RT $PRI_SUB

aws ec2 describe-route-tables --route-table-ids $MAIN_RT --query 'RouteTables[0].Routes'
```

Traffic to S3 matches the endpoint (prefix list) route — most-specific matching picks it over NAT/IGW. **Gateway Endpoints are free** and this avoids NAT per-GB charges (cost-optimization exam answer).

## 7. Changing a route = instant effect

```bash
# Remove the IGW route from the main table (both subnets without custom RT lose internet instantly)
aws ec2 create-route-table --vpc-id $VPC_ID
aws ec2 delete-route --route-table-id $MAIN_RT --destination-cidr-block 0.0.0.0/0
```

Route table changes take effect **immediately** — no instance reboot. This is the classic "cut off the internet" troubleshooting/managed drift technique.

## Cleanup

```bash
aws ec2 delete-route-table --route-table-id $PUB_RT
aws ec2 delete-subnet --subnet-id $PUB_SUB
aws ec2 delete-subnet --subnet-id $PRI_SUB
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-vpc --vpc-id $VPC_ID
```

## Exam hooks

1. **Longest prefix wins** — `/0` lowest priority
2. Subnet = exactly **one** route table; subnet's default = the main table
3. Local route (`VPC CIDR → local`) always exists, can't be removed
4. Peering needs routes **in both VPCs** + no overlapping CIDRs — and is **not transitive**
5. Gateway Endpoint routes (prefix lists) are more specific than NAT → pick-free S3 access
6. Route updates apply instantly — can blackhole traffic if misconfigured

## Gotchas to remember

1. `0.0.0.0/0 → IGW` makes a subnet public only if instances ALSO get public IPs + SG allow
2. Main route table silently applies to any unassociated subnet
3. NAT is IPv4-only — Egress-only IGW handles `::/0`