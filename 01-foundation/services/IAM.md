# AMI (Amazon Machine Image)

## What it is
A template that contains everything needed to launch an EC2 instance: the operating system, pre-installed software, configuration, and block device mapping. Used to launch identical, ready-to-go server copies instead of installing everything from scratch each time.

## Key concepts
- **EBS-backed vs Instance Store-backed** — EBS-backed (most common today) relies on EBS Snapshots; Instance Store-backed stores data on temporary local disk that is lost when the instance stops
- **Region-locked** — an AMI belongs to a single Region; to use it elsewhere you must Copy AMI to the target region
- **Launch Permissions** — control who can use the AMI: private, shared with specific accounts, or public
- **Block Device Mapping** — maps the AMI to its attached EBS volumes and their sizes
- **AMI types** — AWS-provided, AWS Marketplace, Community, Custom

## How it works
When you create an AMI from an existing instance, AWS takes a Snapshot of all attached EBS volumes in the background, records the Block Device Mapping, and registers a new AMI (an AMI ID starting with `ami-`). The AMI goes through a `pending` state before becoming `available`.

It's best practice to stop the instance before creating an AMI to ensure data consistency, especially if there are active disk writes happening during the snapshot.

AMIs are commonly used with **Auto Scaling Groups** to launch identical instances quickly during scaling events, and with **EC2 Image Builder** to automate building, testing, and publishing AMIs on a recurring schedule.

When an AMI is deregistered, its associated snapshots are **not deleted automatically** — they must be removed manually to avoid extra storage costs.

## Exam domain(s)
- [ ] Design Secure Architectures (30%)
- [x] Design Resilient Architectures (26%)
- [x] Design High-Performing Architectures (24%)
- [x] Design Cost-Optimized Architectures (20%)

## Lab notes
What I actually did hands-on (screenshots, steps, gotchas).

## Exam-style questions
**Q1.** A company runs an Auto Scaling Group that launches new EC2 instances during peak traffic. Each new instance takes several minutes to boot and install required software before it can serve traffic. What should a Solutions Architect do to reduce this launch time?
- A) Increase the instance type size
- B) Create a custom AMI with the software pre-installed and use it in the launch template
- C) Use a larger EBS volume
- D) Enable detailed monitoring on the Auto Scaling Group

<details>
<summary>Answer</summary>
B is correct — a custom AMI with pre-installed software removes the need to reinstall it every time a new instance launches, drastically cutting boot time. The other options don't address install time.
</details>

**Q2.** A Solutions Architect created an AMI from a running EC2 instance in the us-east-1 region. The company now needs to launch instances from this AMI in the eu-west-1 region. What should be done?
- A) The AMI is automatically available in all regions
- B) Copy the AMI to the eu-west-1 region using the Copy AMI action
- C) Create a new Snapshot in eu-west-1 manually
- D) Change the AMI's region setting

<details>
<summary>Answer</summary>
B is correct — an AMI belongs to a single region, so using it elsewhere requires an explicit Copy AMI action. There's no setting to change an AMI's region, and manually creating a snapshot isn't necessary since Copy AMI handles that.
</details>

## Related services
- [[EC2]] — the AMI is the template used to launch EC2 instances
- [[EBS]] — EBS-backed AMIs rely on EBS Snapshots under the hood
- [[Auto-scaling]] — uses a standardized AMI to launch identical instances when scaling