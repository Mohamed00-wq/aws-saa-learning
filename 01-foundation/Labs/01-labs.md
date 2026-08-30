# Lab 01 — EC2 & VPC Foundation (Hands-on with floci CLI)

> **Goal:** Build a full network stack from scratch (VPC → Subnet → IGW → Route Table → Security Group), launch an EC2 instance inside it, attach an Elastic IP, and understand EC2 pricing models — all via AWS CLI on floci (AWS Local Runtime).

**Exam Domains covered:** Design Secure Architectures (30%) · Design Resilient Architectures (26%) · Design Cost-Optimized Architectures (20%)

---

## Prerequisites
- floci running locally (console at `localhost:4500/console/aws`)
- AWS CLI configured (`aws configure` — dummy credentials work fine with floci)

---

## Part 1 — Discover available AMIs

**Task:** List the AMIs available in floci, showing only `ImageId` and `Name` in table format.

<details>
<summary>Hint</summary>

- Base command: `aws ec2 describe-images`
- Use `--owners` so you don't pull images from everywhere
- Use `--query` (JMESPath) to filter to just the fields you want
- `--output table` gives a clean tabular view

</details>

```bash
# write your command here
```

---

## Part 2 — Create an SSH key pair

**Task:** Create a new key pair, save the private key to a `.pem` file, and set correct permissions.

<details>
<summary>Hint</summary>

- Command: `aws ec2 create-key-pair --key-name <name>`
- `KeyMaterial` is the key itself — use `--query 'KeyMaterial' --output text` and redirect (`>`) to a file
- SSH refuses keys with overly open permissions — use `chmod 400`

</details>

```bash
# write your command here
```

---

## Part 3 — Build a dedicated VPC from scratch

**Task:** Create a new VPC dedicated to this lab (not the default VPC), with CIDR block `10.0.0.0/16`.

<details>
<summary>Hint</summary>

- Command: `aws ec2 create-vpc --cidr-block <CIDR>`
- Add `--tag-specifications` to name it (e.g. `saa-lab-vpc`) so it's easy to identify later
- Keep the returned `VpcId` — you'll need it for every following step

</details>

```bash
# write your command here
```

---

## Part 4 — Create a public subnet inside the VPC

**Task:** Create a subnet with CIDR `10.0.1.0/24` inside your VPC, in a specific Availability Zone.

<details>
<summary>Hint</summary>

- Command: `aws ec2 create-subnet --vpc-id <VpcId> --cidr-block <CIDR> --availability-zone <AZ>`
- The subnet's CIDR must fall **within** the VPC's range (`10.0.0.0/16`)

</details>

```bash
# write your command here
```

---

## Part 5 — Internet Gateway

**Task:** Create an Internet Gateway and attach it to the VPC (without this step, the instance can't reach/be reached from the internet).

<details>
<summary>Hint</summary>

- Two separate steps: `create-internet-gateway` then `attach-internet-gateway`
- `attach-internet-gateway` needs `--vpc-id` and `--internet-gateway-id`

</details>

```bash
# write your command here
```

---

## Part 6 — Route Table

**Task:** Create a route table, add a route to `0.0.0.0/0` via the Internet Gateway, and associate it with the subnet.

<details>
<summary>Hint</summary>

- 3 steps: `create-route-table` → `create-route` → `associate-route-table`
- Without the association, the route table exists but has no effect on the subnet

</details>

```bash
# write your command here
```

---

## Part 7 — Security Group

**Task:** Create a new security group inside your VPC, and open port 22 (SSH) for inbound access.

<details>
<summary>Hint</summary>

- `create-security-group` needs `--group-name` and `--vpc-id`
- Then `authorize-security-group-ingress` to add the inbound rule (`--protocol tcp --port 22`)
- Without this step, the SG is "empty" (deny by default) even with the correct key

</details>

```bash
# write your command here
```

---

## Part 8 — Launch the EC2 instance

**Task:** Launch a `t2.micro` instance inside the subnet and SG you built.

<details>
<summary>Hint</summary>

- Command: `aws ec2 run-instances`
- You need: `--image-id`, `--instance-type`, `--key-name`, `--subnet-id`, `--security-group-ids`, `--count`
- Confirm state afterward with `describe-instances`

</details>

```bash
# write your command here
```

---

## Part 9 — Elastic IP

**Task:** Assign your instance a static IP (Elastic IP) instead of relying on a regular public IP.

<details>
<summary>Hint</summary>

- Two steps: `allocate-address --domain vpc` then `associate-address --instance-id <id> --allocation-id <id>`
- On floci, `describe-instances` may not reflect the update immediately — verify with `describe-addresses --allocation-ids <id>` instead

</details>

```bash
# write your command here
```

---

## Part 10 — EC2 Pricing Models (conceptual + optional hands-on)

**Task:** Understand the differences between pricing models, and try one hands-on (Spot, for example).

<details>
<summary>Hint — On-Demand</summary>

Same `run-instances` command as Part 8 — no extra config needed.

</details>

<details>
<summary>Hint — Spot Instance</summary>

- Command: `aws ec2 request-spot-instances --instance-count 1 --type "one-time" --launch-specification '{...}'`
- `launch-specification` takes the same info as `run-instances` (ImageId, InstanceType, KeyName, SubnetId, SecurityGroupIds) as JSON
- Confirm with `describe-spot-instance-requests`

</details>

<details>
<summary>Hint — Reserved Instance</summary>

- `describe-reserved-instances-offerings --instance-type <type>` to see available offerings
- `purchase-reserved-instances-offering --reserved-instances-offering-id <id> --instance-count 1`
- In real AWS this is a real financial commitment — just get the concept: committing to 1/3 years for a discount

</details>

<details>
<summary>Hint — Dedicated Host/Instance</summary>

Same `run-instances` but add `--placement Tenancy=dedicated`

</details>

```bash
# write the commands here for whichever type you choose to try
```

---

## Quick Recap — Pricing Models (for the exam)

| Model | Description | Best for |
|---|---|---|
| On-Demand | Pay per hour/second, no commitment | Unpredictable workloads, short tests |
| Reserved | 1-3 year commitment on a specific instance type, big discount | Steady, predictable workloads |
| Savings Plans | Commit to $/hour spend, more flexible than Reserved | Steady workloads with flexibility on instance type |
| Spot | Huge discount (70-90%), but AWS can reclaim it | Interruption-tolerant workloads (batch jobs, testing) |
| Dedicated Host/Instance | Dedicated hardware just for you | Compliance/licensing requirements |

---

## Exam-style Questions

<details>
<summary>Q1: A company needs to run a batch processing job that can tolerate interruptions, at the lowest possible cost. Which pricing model fits best?</summary>

**Answer:** Spot Instances — cheapest option, and a good fit since the workload tolerates interruption.

</details>

<details>
<summary>Q2: What's the key difference between an Elastic IP and a standard auto-assigned public IP?</summary>

**Answer:** An Elastic IP is static and owned by you (persists even if you stop/terminate the instance and reattach it elsewhere), while a regular public IP changes or is released on instance stop/start.

</details>

<details>
<summary>Q3: Why does an EC2 instance in a newly created VPC have no internet access by default, even after attaching an Internet Gateway?</summary>

**Answer:** You also need a route table with a route to `0.0.0.0/0` via the IGW, associated with the subnet — the IGW alone isn't enough.

</details>

<details>
<summary>Q4: If an Elastic IP is allocated but not associated with any running instance, what happens?</summary>

**Answer:** AWS charges you for it (idle Elastic IP) — a classic point under Cost-Optimized Architectures.

</details>

---

## Related Services
[[IAM]] · [[EC2]] · [[EBS]] · [[Auto-Scaling]] · [[ELB]] · [[VPC]]