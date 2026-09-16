# Lab 00 — Floci Setup (Run Once)

> Floci is a free, open-source local AWS emulator (drop-in LocalStack replacement). No account, no auth token. All services on `localhost:4566`.

## Prerequisites

- **Docker** (with docker socket available)
- **AWS CLI** — `aws --version`
- Optionally the **Floci CLI** for convenience

## Option A — Docker Compose (recommended)

```yaml
# compose.yaml
services:
  floci:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
```

```bash
docker compose up -d
```

## Option B — Docker run

```bash
docker run -d --name floci \
  -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  floci/floci:latest
```

## Option C — Floci CLI

```bash
curl -fsSL https://floci.io/install.sh | sh
floci start
eval $(floci env)   # exports AWS_* env vars
```

## Environment variables

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
```

> Any non-empty credentials work. Region can be anything; floci supports any region at `localhost:4566`.

## Verify it's working

```bash
aws sts get-caller-identity
# Account: 000000000000
aws s3 mb s3://smoke-test-bucket && aws s3 ls
```

## Data reset

Floci is in-memory by default. To wipe everything:

```bash
docker compose down -v    # or: docker rm -f floci
docker compose up -d
```

**Storage modes:** `memory` (default), `persistent`, `wal` via `FLOCI_STORAGE_MODE`.

## Reading order

| Lab | Topic | Services touched |
|---|---|---|
| [01-ec2.md](01-ec2.md) | EC2 compute | EC2, Key Pairs |
| [02-vpc.md](02-vpc.md) | Networking foundation | VPC, IGW |
| [03-security-groups.md](03-security-groups.md) | Stateful firewall | EC2 SG |
| [04-nacls.md](04-nacls.md) | Stateless subnet firewall | NACL |
| [05-ebs.md](05-ebs.md) | Block storage | EBS, Snapshots |
| [06-elb.md](06-elb.md) | Load balancing | ELB, Target Groups |
| [07-iam.md](07-iam.md) | Identity & access | IAM, STS |
| [08-amis.md](08-amis.md) | Machine images | EC2, AMI, Snapshots |
| [09-auto-scaling.md](09-auto-scaling.md) | Auto scaling | ASG, Auto Scaling |
| [10-nat.md](10-nat.md) | Network address translation | NAT GW |
| [11-route-tables.md](11-route-tables.md) | Routing | VPC, Route Tables |
| [12-pricing.md](12-pricing.md) | EC2 pricing models | EC2 (conceptual) |