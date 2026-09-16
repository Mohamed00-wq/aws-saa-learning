# Lab 06 — ELB with Floci

> Load balancing locally. Create a target group, register targets, and see health checks + routing in action.

**Reference notes:** `services/ELB.md`

## 1. Create a Target Group

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query 'Vpcs[0].VpcId' --output text)

TG_ARN=$(aws elbv2 create-target-group \
  --name lab-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --health-check-path / \
  --health-check-interval-seconds 30 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
echo $TG_ARN
```

## 2. Create an ALB (Layer 7 — content routing)

```bash
SUBNETS=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].SubnetId' --output text | tr '\t' ' ')
SG_ID=$(aws ec2 create-security-group --group-name lb-sg --description "lb sg" --vpc-id $VPC_ID --query 'GroupId' --output text)

ALB_ARN=$(aws elbv2 create-load-balancer \
  --name lab-alb \
  --type application \
  --subnets $SUBNETS \
  --security-groups $SG_ID \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)
echo $ALB_ARN
```

**Verify state** (provisioned → active):

```bash
aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN --query 'LoadBalancers[0].[State.Code,Type,DNSName]'
```

## 3. Create a listener & forward to the target group

```bash
LISTENER_ARN=$(aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --query 'Listeners[0].ListenerArn' --output text)
```

## 4. Register targets

```bash
INSTANCE_ID=$(aws ec2 run-instances --image-id ami-00000000 --instance-type t2.micro --query 'Instances[0].InstanceId' --output text)

aws elbv2 register-targets --target-group-arn $TG_ARN --targets Id=$INSTANCE_ID
```

## 5. Health checks — the difference between ELB and ASG health

```bash
aws elbv2 describe-target-health --target-group-arn $TG_ARN --query 'TargetHealthDescriptions[*].TargetHealth'
```

> EC2 health check = hardware/OS only. **ELB health check = application serving**, drives replacement in ASGs (lab 09).

## 6. Add path-based routing (ALB vs NLB superpower)

Create a second target group + route `/api/*` to it:

```bash
TG_API=$(aws elbv2 create-target-group --name lab-api-tg --protocol HTTP --port 80 --vpc-id $VPC_ID --query 'TargetGroups[0].TargetGroupArn' --output text)

# HTTP listener rule: /api/* → api-tg, everything else → default tg
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 10 \
  --conditions Field=path-pattern,Values=/api/* \
  --actions Type=forward,TargetGroupArn=$TG_API
```

**Verify rules (evaluated by priority, default catches the rest):**

```bash
aws elbv2 describe-rules --listener-arn $LISTENER_ARN --query 'Rules[*].[Priority,Conditions,Actions]'
```

## 7. Sticky sessions (L5 concept)

```bash
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes Key=stickiness.enabled,Value=true,Key=stickiness.type,Value=lb_cookie,Key=stickiness.lb_cookie.duration_seconds,Value=3600

aws elbv2 describe-target-group-attributes --target-group-arn $TG_ARN
```

> Best practice: externalize session state (ElastiCache/DynamoDB) and avoid sticky sessions. Sticky = one client pinned to one target behind `AWSALB` cookie.

## 8. NLB for comparison (Layer 4)

```bash
NLB_ARN=$(aws elbv2 create-load-balancer --name lab-nlb --type network --subnets $SUBNETS --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 describe-load-balancers --load-balancer-arns $NLB_ARN --query 'LoadBalancers[0].[Type,State.Code]'
```

> NLB = raw TCP/UDP, ultra-low latency, **static IPs**, preserves source IP, cross-zone off by default (and costs when enabled on NLB — free on ALB).

## Cleanup

```bash
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $NLB_ARN
aws elbv2 delete-target-group --target-group-arn $TG_ARN
aws elbv2 delete-target-group --target-group-arn $TG_API
```

## Exam hooks

| | ALB | NLB |
|---|---|---|
| Content routing (path/host/header) | Yes | No |
| Static IP | No | Yes |
| Preserve source IP | No | Yes |
| TCP/UDP | No | Yes |
| Lambda target / WAF | Yes | No |
| Cross-zone | On by default (free) | Off by default (cost) |

- CLB = legacy, avoid in new designs
- GWLB = L3, third-party appliances, GENEVE — "bump in the wire"
- Connection draining / deregistration delay: default 300s, tune to connection length

## Gotchas to remember

1. ALB stateful; NLB can be stateless (raw forwarding)
2. NLB doesn't support WAF → use ALB
3. Health checks aren't free — each is an HTTP request
4. ALB routes to Lambda for serverless
5. Security group on instances controls what the LB can reach