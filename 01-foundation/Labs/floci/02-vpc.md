# Lab 02 — VPC with Floci

> Build an isolated network from scratch: VPC → subnet → Internet Gateway → attach. This is the skeleton everything else sits on.

**Reference notes:** `services/VPC.md`, `services/Route-tables.md`

## 1. Create the VPC

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16 --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=lab-vpc}]'
```

Take note of the `VpcId` (e.g. `vpc-xxxx`). Save it:

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=lab-vpc" --query 'Vpcs[0].VpcId' --output text)
echo $VPC_ID
```

**Verify:**

```bash
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[0].[CidrBlock,State,IsDefault]'
```

## 2. Create a public subnet

```bash
SUBNET_ID=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.0.1.0/24 \
  --availability-zone us-east-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=lab-public}]' \
  --query 'Subnet.SubnetId' --output text)
echo $SUBNET_ID
```

**Verify — AWS reserves 5 IPs per subnet:**

```bash
aws ec2 describe-subnets --subnet-ids $SUBNET_ID --query 'Subnets[0].[AvailabilityZone,AvailableIpAddressCount,CidrBlock]'
```

- A `/24` has 256 addresses, `AvailableIpAddressCount` should be **251** (5 reserved).

## 3. Create + attach an Internet Gateway

```bash
IGW_ID=$(aws ec2 create-internet-gateway --query 'InternetGateway.InternetGatewayId' --output text)
echo $IGW_ID

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

> **Common trap:** creating the IGW is not enough — you MUST attach it. Verify:

```bash
aws ec2 describe-internet-gateways --internet-gateway-ids $IGW_ID --query 'InternetGateways[0].Attachments'
```

## 4. Make the subnet public (routine)

Floci lets you mark a subnet public directly:

```bash
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_ID --map-public-ip-on-launch
```

Check:

```bash
aws ec2 describe-subnets --subnet-ids $SUBNET_ID --query 'Subnets[0].MapPublicIpOnLaunch'
```

## 5. Route table with IGW route

```bash
RT_ID=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id $SUBNET_ID
```

**Verify — the local route + internet route:**

```bash
aws ec2 describe-route-tables --route-table-ids $RT_ID --query 'RouteTables[0].Routes'
```

Expected: `10.0.0.0/16 → local` (always present) and `0.0.0.0/0 → igw-...`.

## 6. What makes a subnet truly "public"?

A subnet is public **only when all three** are true:

1. Route table has `0.0.0.0/0 → IGW`
2. Instances get public IPs (`MapPublicIpOnLaunch`)
3. Security group allows inbound

Test by attaching the route table then launching an instance (see [01-ec2.md](01-ec2.md)).

## 7. Default VPC inspection

```bash
aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId'
```

> Default VPC comes pre-wired (public subnet + IGW). Custom VPCs start locked down — floci behaves the same way.

## Cleanup

```bash
aws ec2 delete-subnet --subnet-id $SUBNET_ID
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID
aws ec2 delete-route-table --route-table-id $RT_ID
aws ec2 delete-vpc --vpc-id $VPC_ID
```

## Exam hooks

- Subnet = **one AZ**; VPC = **all AZs** in region
- **5 reserved IPs** per subnet (first 4 + last)
- IGW is horizontally scaled + redundant — no scaling needed
- CIDR `/16`–`/28`; CIDR cannot be changed after creation
- Peering is **not transitive**

## Gotchas to remember

1. IGW must be attached — creation alone does nothing
2. Route table association is required for traffic to flow
3. Default VPC allows everything; custom VPC starts locked down
4. VPC Flow Logs capture allowed/denied traffic at the ENI level