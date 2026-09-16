# Lab 10 — NAT with Floci

> Let private-subnet instances reach the internet (outbound) while staying unreachable from the outside. NAT Gateway is AZ-scoped, IPv4-only — use Egress-only IGW for IPv6.

**Reference notes:** `services/NAT.md`

## 1. Build the baseline: VPC + 2 subnets

```bash
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --query 'Vpc.VpcId' --output text)

PUBLIC=$VPC_ID-subnet-public
PRIVATE=$VPC_ID-subnet-private

PUB_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
PRI_SUB=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1a --query 'Subnet.SubnetId' --output text)
echo "public: $PUB_SUB / private: $PRI_SUB"
```

## 2. Internet Gateway for the public subnet

```bash
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

## 3. Public route table (→ IGW), Private route table (→ NAT later)

```bash
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
PRI_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUB_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_SUB
aws ec2 associate-route-table --route-table-id $PRI_RT --subnet-id $PRI_SUB
```

## 4. NAT Gateway (in the PUBLIC subnet, with an EIP)

```bash
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

NAT_ID=$(aws ec2 create-nat-gateway \
  --subnet-id $PUB_SUB \
  --allocation-id $EIP_ALLOC \
  --query 'NatGateway.NatGatewayId' --output text)
echo $NAT_ID
```

> NAT GW **must live in a public subnet** and **requires an Elastic IP**. It translates the source IP of private instances → its own public IP.

**Wait for available:**

```bash
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_ID
```

## 5. Point the private subnet at the NAT

```bash
aws ec2 create-route --route-table-id $PRI_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_ID

aws ec2 describe-route-tables --route-table-ids $PRI_RT --query 'RouteTables[0].Routes'
# 0.0.0.0/0 → nat-...   ← private egress path
```

## 6. The traffic flow

```
Private instance (10.0.2.50)
   → route 0.0.0.0/0 → NAT GW (10.0.1.x, public subnet)
      → NAT replaces source IP with its EIP
      → IGW → Internet
      → response → NAT GW → back to private instance
```

- Outbound: allowed (NAT translates).
- Inbound: **impossible** — there's no route *into* the private subnet from the internet. That's the security win.

## 7. NAT Gateway parity check — "one per AZ"

```bash
# In production you deploy a NAT GW PER AZ so an AZ failure
# doesn't take other AZs' internet down.
# (Route from private subnet-B should point to NAT GW in AZ B.)
```

**Cost reality:** `$0.045/hr` + `$0.045/GB` processed. For S3/DynamoDB, use a **Gateway Endpoint — free**, no internet path (see lab 11).

## 8. IPv6 — Egress-only IGW

```bash
# NAT GW/Instance work ONLY with IPv4.
# For IPv6 outbound-only from a private subnet:
aws ec2 create-egress-only-internet-gateway --vpc-id $VPC_ID
# → route ::/0 → eigw-xxxx in the private route table
```

## Cleanup

```bash
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_ID
aws ec2 release-address --allocation-id $EIP_ALLOC
aws ec2 delete-subnet --subnet-id $PUB_SUB
aws ec2 delete-subnet --subnet-id $PRI_SUB
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-route-table --route-table-id $PUB_RT
aws ec2 delete-route-table --route-table-id $PRI_RT
aws ec2 delete-vpc --vpc-id $VPC_ID
```

## Exam hooks

| | NAT Gateway | NAT Instance |
|---|---|---|
| Managed by | AWS | You (EC2) |
| Bandwidth | up to 45 Gbps | instance type |
| AZ | AZ-scoped — one per AZ | any AZ |
| SG rules | No (route tables control) | Yes |
| Failover | auto within AZ | ASG/manual |
| Source/dest check | n/a | must disable |

- NAT is **IPv4 only** → Egress-only IGW for IPv6 (stateful, outbound-only)
- Gateway Endpoints (free) for S3/DynamoDB — bypass NAT entirely
- Connection tracking default timeout **350s**

## Gotchas to remember

1. Shared NAT GW = AZ failure breaks all other AZs
2. NAT Instance requires **source/dest check disabled** — won't work without it
3. NAT GW doesn't support SG rules
4. Monitor `PacketsDropCount` (connection tracking saturation)