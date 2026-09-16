# Lab 09 — Auto Scaling with Floci

> Auto Scaling Groups keep the right number of instances running and make EC2 self-healing. Launch templates, min/max/desired, health checks, scaling policies, and instance refresh.

**Reference notes:** `services/Auto-scaling.md`

## 1. Launch Template (versioned — `$Latest` / `$Default`)

```bash
AMI_ID=ami-00000000

aws ec2 create-launch-template \
  --launch-template-name lab-lt \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId":"'$AMI_ID'",
    "InstanceType":"t2.micro",
    "KeyName":"lab-key"
  }'
```

**Verify & update — versioning:**

```bash
aws ec2 describe-launch-template-versions --launch-template-name lab-lt --query 'LaunchTemplateVersions[*].[VersionNumber,VersionDescription]'

# Update → v2 becomes $Default
aws ec2 create-launch-template-version \
  --launch-template-name lab-lt \
  --version-description "v2 - bigger" \
  --launch-template-data '{"ImageId":"'$AMI_ID'","InstanceType":"t3.micro"}'

aws ec2 describe-launch-templates --launch-template-names lab-lt --query 'LaunchTemplates[0].[DefaultVersionNumber,LatestVersionNumber]'
```

## 2. Create the Auto Scaling Group

```bash
SUBNETS=$(aws ec2 describe-subnets --query 'Subnets[].SubnetId' --output text | tr '\t' ' ')
SUBNET_LIST=$(echo $SUBNETS | tr ' ' ',')

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-asg \
  --launch-template LaunchTemplateName=lab-lt \
  --min-size 1 \
  --max-size 4 \
  --desired-capacity 2 \
  --vpc-zone-identifier "$SUBNET_LIST"
```

**Verify:**

```bash
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-asg \
  --query 'AutoScalingGroups[0].[MinSize,MaxSize,DesiredCapacity,Instances]'
```

## 3. Desired capacity is bounded by min/max

```bash
# Try to want MORE than max
aws autoscaling set-desired-capacity --auto-scaling-group-name lab-asg --desired-capacity 10
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-asg \
  --query 'AutoScalingGroups[0].DesiredCapacity'
# → clamps to max (4)
```

## 4. Self-healing — terminate an instance

```bash
INSTANCE_ID=$(aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names lab-asg \
  --query 'AutoScalingGroups[0].Instances[0].InstanceId' --output text)

# Kill the underlying EC2 instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# ASG replaces the failed instance automatically
sleep 5
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names lab-asg \
  --query 'AutoScalingGroups[0].Instances[*].{ID:InstanceId,Health:HealthStatus,Lifecycle:LifecycleState}'
```

> The ASG senses the lost instance (EC2 health check / standby) and launches a replacement to restore desired capacity. This is the **self-healing** exam point.

## 5. Scaling policies

**Target tracking** (simplest — set a CPU target, ASG does the math):

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name lab-asg \
  --policy-name cpu-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{ "TargetValue": 50.0, "PredefinedMetricSpecification": { "PredefinedMetricType": "ASGAverageCPUUtilization" } }'
```

**Scheduled scaling** (known times — e.g. daily spike at 9am):

```bash
aws autoscaling put-scheduled-update-group-action \
  --auto-scaling-group-name lab-asg \
  --scheduled-action-name daily-spike \
  --recurrence "0 9 * * *" \
  --desired-capacity 4

aws autoscaling describe-scheduled-actions --auto-scaling-group-name lab-asg
```

## 6. Mixed instances — On-Demand base + Spot burst

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name lab-mixed-asg \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": { "LaunchTemplateName": "lab-lt" },
      "Overrides": [
        { "InstanceType": "t2.micro" },
        { "InstanceType": "t3.micro" }
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 1,
      "OnDemandPercentageAboveBaseCapacity": 50,
      "SpotAllocationStrategy": "capacity-optimized"
    }
  }' \
  --min-size 1 --max-size 6 --desired-capacity 3 \
  --vpc-zone-identifier "$SUBNET_LIST"
```

> Cost pattern: Reserved/Savings Plans (baseline) + On-Demand (moderate scale) + Spot (burst). `capacity-optimized` = fewest interruptions (recommended).

## 7. Instance Refresh (zero-downtime deployment)

```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name lab-asg \
  --preferences '{"MinHealthyPercentage": 100}'

aws autoscaling describe-instance-refreshes --auto-scaling-group-name lab-asg
```

> Instance Refresh rolls out a new launch template version while keeping min healthy %, replaces instances one by one — zero downtime.

## 8. Scale-in protection

```bash
aws autoscaling set-instance-protection \
  --auto-scaling-group-name lab-asg \
  --instance-ids $INSTANCE_ID \
  --protected-from-scale-in
```

Protects critical instances from being terminated by scale-in events.

## Cleanup

```bash
aws autoscaling set-desired-capacity --auto-scaling-group-name lab-asg --desired-capacity 0
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-asg --force-delete
aws autoscaling delete-auto-scaling-group --auto-scaling-group-name lab-mixed-asg --force-delete
aws ec2 delete-launch-template --launch-template-name lab-lt
```

## Exam hooks

- ASGs ≠ just scaling — they're the **self-healing + HA** mechanism (paired with ELB + Launch Template)
- **Horizontal** scaling (more instances, zero downtime) vs **vertical** (bigger instance, needs stop/start, downtime)
- Desired bounded by min/max
- Health check grace period (300s) protects new instances
- Warm Pool = pre-initialized stopped instances (count toward max!)
- Predictive scaling needs ~14 days of history
- Termination default: closest to next billing hour (cost-efficient)

## Gotchas to remember

1. Launch Configurations are **immutable**; Launch Templates are **versioned**
2. Degraded-but-serving instances replaced only if ELB marks unhealthy
3. Cooldown (default 300s) prevents flapping — per-policy
4. Prefer horizontal for stateless/HA; vertical for quick fixes on stateful