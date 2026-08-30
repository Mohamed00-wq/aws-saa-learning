# EC2 (Elastic Compute Cloud)

## What it is
The core compute service in AWS — provides resizable virtual servers (instances) in the cloud. It solves the problem of needing on-demand, scalable computing power without buying or maintaining physical hardware. It sits at the center of most AWS architectures as the "server" building block that other services (load balancers, auto scaling, storage) attach to.

## Key concepts
- **Instance** — a running virtual server launched from an AMI
- **Instance Type** — defines the hardware profile (CPU, memory, network, storage) e.g. `t3.micro`, `m5.large`
- **AMI (Amazon Machine Image)** — the template an instance is launched from (OS + pre-installed software + config)
- **Security Group** — a virtual firewall controlling inbound/outbound traffic to the instance
- **Key Pair** — used for secure SSH/RDP access to the instance
- **EBS Volume** — persistent block storage attached to an instance
- **Instance Lifecycle** — pending → running → stopping → stopped → terminated
- **Pricing Models** — On-Demand, Reserved, Spot, Savings Plans, Dedicated Hosts

## Key commands (AWS CLI)
```bash
# List all running instances
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"

# Launch a new instance from an AMI
aws ec2 run-instances \
    --image-id ami-0abcd1234efgh5678 \
    --instance-type t3.micro \
    --key-name my-key-pair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --count 1

# Stop an instance
aws ec2 stop-instances --instance-ids i-0abcd1234efgh5678

# Start a stopped instance
aws ec2 start-instances --instance-ids i-0abcd1234efgh5678

# Terminate an instance
aws ec2 terminate-instances --instance-ids i-0abcd1234efgh5678

# Create an AMI from a running/stopped instance
aws ec2 create-image \
    --instance-id i-0abcd1234efgh5678 \
    --name "MyCustomAMI" \
    --no-reboot

# Describe available AMIs owned by you
aws ec2 describe-images --owners self
```

## How it works
EC2 instances are always launched **from** an AMI — you cannot launch an instance from nothing. When you run `run-instances`, AWS copies the AMI's OS and pre-installed software onto a new EBS volume attached to the new instance, then boots it. From that point on, the instance is independent: changes made inside it do not affect the original AMI or any other instance launched from the same AMI.

EC2 connects to many other services: **EBS** for persistent storage, **Security Groups** and **NACLs** for network access control, **VPC** for networking placement, **Elastic Load Balancer** for distributing traffic across instances, and **Auto Scaling Groups** for automatically launching/terminating instances based on demand — always using an AMI as the launch template.

Limits/quotas worth remembering: default vCPU-based limits per instance family per region (soft limits, can request increases), and instance store data is lost on stop/terminate unless the volume is EBS-backed.

## Relationship between AMI and EC2

- **AMI is the template, EC2 Instance is the running copy.** An AMI packages an OS + software + configuration; an EC2 instance is the live, billable, running server created from that package.
- **You must select an AMI to launch an instance** — it's a required parameter in `run-instances`, not optional.
- **The relationship is bidirectional**: you launch an instance *from* an AMI, and you can also *create* a new AMI *from* an existing (ideally stopped) instance — closing the loop: AMI → EC2 → (configure) → new AMI → more EC2 instances.
- **Consistency at scale**: Auto Scaling Groups rely on a single AMI (via a Launch Template) so that every new instance spun up during a scaling event is identical — same OS, same software, same config.
- **Independence after launch**: once an instance is running, it's fully decoupled from its source AMI — modifying the instance never modifies the AMI, and modifying/deleting the AMI never affects instances already launched from it (though you'd lose the ability to launch *new* identical ones).
- **Region scope carries over**: since an AMI is region-locked, the instances launched from it are also confined to that same region unless the AMI is copied elsewhere first.

## Exam domain(s)
- [ ] Design Secure Architectures (30%)
- [x] Design Resilient Architectures (26%)
- [x] Design High-Performing Architectures (24%)
- [x] Design Cost-Optimized Architectures (20%)

## Lab notes
What I actually did hands-on (screenshots, steps, gotchas).

## Exam-style questions
**Q1.** A Solutions Architect needs to launch 20 identical EC2 instances with the same OS, software, and configuration, as quickly as possible. What should they do?
- A) Manually launch 20 blank instances and configure each one individually
- B) Create a custom AMI from a fully configured instance and launch all 20 instances from that AMI
- C) Use 20 different AMIs, one per instance
- D) Launch the instances first, then install software afterward via SSH

<details>
<summary>Answer</summary>
B is correct — a single custom AMI guarantees all 20 instances are identical and launch fast since the software is already baked in. Manual configuration (A, D) is slow and error-prone; using different AMIs (C) breaks consistency.
</details>

**Q2.** An EC2 instance was launched from a public AMI. The engineer then installs additional software and makes configuration changes directly on the running instance. What happens to the original AMI?
- A) The AMI is automatically updated with the new changes
- B) The AMI remains unchanged; the instance is now independent of it
- C) The instance loses its connection to the AMI and stops working
- D) AWS creates a new AMI version automatically every hour

<details>
<summary>Answer</summary>
B is correct — once launched, an instance is a fully independent copy. Changes made on the running instance have no effect on the source AMI, and there's no automatic syncing or versioning between the two.
</details>

## Related services
- [[AMI]] — the template EC2 instances are launched from
- [[EBS]] — provides persistent storage attached to EC2 instances
- [[Auto-scaling]] — automatically launches/terminates EC2 instances based on demand, using an AMI as the source
- [[ELB]] — distributes incoming traffic across multiple EC2 instances
- [[VPC]] — defines the network (subnets, routing) EC2 instances run inside