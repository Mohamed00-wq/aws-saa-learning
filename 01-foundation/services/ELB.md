# ELB (Elastic Load Balancing)

## What it is
Elastic Load Balancing (ELB) automatically distributes incoming 
application traffic across multiple targets (EC2 instances, 
containers, IP addresses, Lambda functions) in one or more 
Availability Zones.

## Key concepts

### Types of Load Balancers
1. **Application Load Balancer (ALB)** - Layer 7 (HTTP/HTTPS)
   - Content-based routing (path, host, headers)
   - Best for microservices, container-based apps
   - Supports WebSocket, HTTP/2

2. **Network Load Balancer (NLB)** - Layer 4 (TCP/UDP/TLS)
   - Ultra-high performance, millions of requests/sec
   - Static IP support (Elastic IP per AZ)
   - Best for extreme performance, low latency

3. **Gateway Load Balancer (GWLB)** - Layer 3 (Network layer)
   - For deploying/scaling third-party virtual appliances
   - (firewalls, IDS/IPS)

4. **Classic Load Balancer (CLB)** - Legacy, avoid in new designs

### Core components
- **Listener**: Checks for connection requests on a protocol/port
- **Target Group**: Set of resources ELB routes traffic to
- **Health Checks**: Periodic checks to determine target status
  (healthy/unhealthy) - unhealthy targets removed from rotation
- **Cross-Zone Load Balancing**: Distributes traffic evenly across
  ALL registered targets in ALL enabled AZs (not just local AZ)

### Security
- ELB itself needs a Security Group (for ALB/CLB, not NLB)
- SSL/TLS termination can happen at the ELB level
- Supports AWS Certificate Manager (ACM) integration

## Key commands (AWS CLI)

# Create an ALB
aws elbv2 create-load-balancer \
  --name my-alb \
  --subnets subnet-xxx subnet-yyy \
  --security-groups sg-xxx \
  --type application

# Create a target group
aws elbv2 create-target-group \
  --name my-targets \
  --protocol HTTP \
  --port 80 \
  --vpc-id vpc-xxx \
  --health-check-path /health

# Register targets
aws elbv2 register-targets \
  --target-group-arn arn:aws:... \
  --targets Id=i-xxxxxx

# Create a listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:... \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:...

# Describe target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:...

## How it works
1. Client sends request → hits the ELB's DNS name (ELB has no
   fixed public IP by default, except NLB with EIP)
2. Listener checks the request against configured rules
3. ELB forwards request to a healthy target in the target group,
   based on routing algorithm (round robin for ALB, flow hash
   for NLB)
4. Health checks continuously monitor targets; unhealthy ones
   are pulled out automatically
5. Works hand-in-hand with ASG: as ASG launches/terminates
   instances, they auto-register/deregister with the target group

## Exam domain(s) checklist
- [ ] Design Resilient Architectures (26%) — ELB across multiple
      AZs = core HA pattern
- [ ] Design High-Performing Architectures (24%) — choosing 
      ALB vs NLB vs GWLB based on use case
- [ ] Design Secure Architectures (30%) — SSL termination, SG 
      rules on ELB

## Lab notes
(يتوضاف بعد ما تكمل الـ hands-on)

## Exam-style questions

**Q1:** A company needs to route traffic based on URL path 
(/api/* to one set of servers, /images/* to another). Which 
load balancer type should they use?
<details><summary>Answer</summary>Application Load Balancer (ALB) - 
supports content-based/path-based routing at Layer 7.</details>

**Q2:** An application requires extremely low latency and needs 
to handle millions of requests per second with a static IP. 
Which load balancer fits best?
<details><summary>Answer</summary>Network Load Balancer (NLB) - 
Layer 4, designed for extreme performance and supports static/
Elastic IPs.</details>

**Q3:** True or False: Cross-Zone Load Balancing means the ELB 
only distributes traffic within the AZ where the request 
originated.
<details><summary>Answer</summary>False - Cross-Zone Load Balancing 
distributes traffic evenly across ALL targets in ALL enabled AZs, 
regardless of which AZ the request came from.</details>

## Related services
[[EC2]] [[Auto-scaling]] [[VPC]] [[ACM]] [[Route53]]