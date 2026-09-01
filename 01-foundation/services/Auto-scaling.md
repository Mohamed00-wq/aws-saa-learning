# (ASG) Auto Scaling Groups

## What it is
Auto Scaling Groups (ASG) automatically adjust the number of EC2 
instances in response to demand, ensuring the right number of 
healthy instances are running at all times — scaling out under 
load, scaling in to save cost when demand drops.

## Key concepts

### Core components
- **Launch Template** (preferred over legacy Launch Configuration)
  - Defines: AMI, instance type, key pair, security groups,
    user data script — the "blueprint" for new instances
- **ASG settings**:
  - **Min size**: never go below this number of instances
  - **Max size**: never go above this number
  - **Desired capacity**: target number ASG tries to maintain
- **VPC/Subnets**: ASG spreads instances across the subnets/AZs
  you specify → built-in high availability
- **Target Group attachment**: ASG registers/deregisters instances
  with an ELB target group automatically as it scales

### Scaling Policies
1. **Target Tracking** - simplest, most common
   - "Keep average CPU at 50%" → ASG adjusts automatically
2. **Step Scaling** - scale by defined increments based on
   CloudWatch alarm thresholds (e.g. +2 instances if CPU > 70%)
3. **Simple Scaling** - single scaling action per alarm (legacy)
4. **Scheduled Scaling** - scale based on known time patterns
   (e.g. scale up every weekday at 8 AM)
5. **Predictive Scaling** - uses ML to forecast traffic and
   scale ahead of demand

### Health Checks
- ASG can use **EC2 status checks** and/or **ELB health checks**
- If ELB health check enabled: ASG terminates instances the ELB
  marks unhealthy and launches replacements automatically
- **Cooldown period**: time after a scaling activity before ASG
  will trigger another one (prevents rapid flapping)

### Termination Policy
- Controls WHICH instance ASG kills first when scaling in
  (default: oldest launch template + closest to next billing hour)

## Key commands (AWS CLI)

# Create a launch template
aws ec2 create-launch-template \
  --launch-template-name my-template \
  --version-description v1 \
  --launch-template-data '{"ImageId":"ami-xxxxxx","InstanceType":"t2.micro"}'

# Create an Auto Scaling Group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name my-asg \
  --launch-template LaunchTemplateName=my-template,Version='$Latest' \
  --min-size 2 --max-size 6 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-xxx,subnet-yyy" \
  --target-group-arns arn:aws:elasticloadbalancing:...

# Attach a target group to existing ASG
aws autoscaling attach-load-balancer-target-groups \
  --auto-scaling-group-name my-asg \
  --target-group-arns arn:aws:...

# Create a target tracking scaling policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name my-asg \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"},
    "TargetValue": 50.0}'

# Describe ASG status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names my-asg

## How it works
1. You define a Launch Template (what an instance should look like)
2. You create an ASG referencing that template + min/max/desired
   + which subnets (AZs) to spread across
3. ASG launches `desired capacity` instances immediately
4. CloudWatch metrics (CPU, requests, custom metrics) feed scaling
   policies → ASG adds/removes instances to match demand
5. New instances auto-register with attached ELB target group;
   terminated instances auto-deregister
6. Unhealthy instances (per EC2 or ELB health checks) are replaced
   automatically — this is what gives ASG its "self-healing" property

## Exam domain(s) checklist
- [ ] Design Resilient Architectures (26%) — self-healing, 
      multi-AZ distribution, min/max/desired
- [ ] Design High-Performing Architectures (24%) — target tracking
      vs step scaling, predictive scaling
- [ ] Design Cost-Optimized Architectures (20%) — scheduled 
      scaling, right-sizing desired capacity, mixing Spot/On-Demand

## Lab notes
(يتوضاف بعد ما تكمل الـ hands-on)

## Exam-style questions

**Q1:** An ASG has min=2, max=10, desired=4, and one instance 
fails its ELB health check. What happens?
<details><summary>Answer</summary>ASG terminates the unhealthy 
instance and launches a replacement to bring the count back to 
the desired capacity (4) — this is automatic self-healing.</details>

**Q2:** Which scaling policy type would best handle a predictable 
daily traffic spike every morning at 9 AM?
<details><summary>Answer</summary>Scheduled Scaling - lets you 
define scaling actions at specific known times, rather than 
reacting to metrics after the spike already starts.</details>

**Q3:** True or False: An ASG can only launch instances into a 
single Availability Zone.
<details><summary>Answer</summary>False - ASG is designed to 
spread instances across multiple subnets/AZs (as specified in 
vpc-zone-identifier) for high availability.</details>

## Related services
[[EC2]] [[ELB]] [[CloudWatch]] [[Launch-Template]]