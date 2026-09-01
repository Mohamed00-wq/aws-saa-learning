# (EBS) Elastic Block Store

## What it is
Elastic Block Store (EBS) provides persistent block-level storage 
volumes for use with EC2 instances. Unlike instance store, EBS 
data persists independently of the instance lifecycle (survives 
stop/start, and can be detached/reattached to another instance).

## Key concepts

### Volume Types
1. **gp3** (General Purpose SSD) - Latest gen, default choice
   - Baseline 3,000 IOPS, 125 MB/s throughput (independent of size)
   - Can provision extra IOPS/throughput separately from size
   - Cheaper than gp2 for same performance

2. **gp2** (General Purpose SSD) - Previous gen
   - IOPS tied to volume size (3 IOPS/GB, burstable to 3,000)

3. **io1/io2** (Provisioned IOPS SSD) - High-performance
   - For latency-sensitive, high-IOPS workloads (databases)
   - io2 Block Express: up to 256,000 IOPS

4. **st1** (Throughput Optimized HDD) - Big data, log processing
   - Cannot be a boot volume

5. **sc1** (Cold HDD) - Infrequently accessed data, lowest cost
   - Cannot be a boot volume

### Key characteristics
- **AZ-locked**: An EBS volume lives in a single Availability Zone,
  must attach to an instance in the SAME AZ
- **Snapshots**: Point-in-time backups stored in S3, incremental
  (only changed blocks after the first snapshot), can be copied
  across regions/AZs to create a new volume elsewhere
- **Elastic Volumes**: Can increase size, change type, or adjust
  IOPS/throughput ON THE FLY without detaching
- **Encryption**: EBS encryption uses KMS, encrypts data at rest,
  in transit between instance and volume, and all snapshots

### EBS vs Instance Store
- Instance Store: physically attached, ephemeral (data lost on
  stop/terminate), very high IOPS, free
- EBS: network-attached, persistent, billed separately

## Key commands (AWS CLI)

# Create a volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --size 20 --volume-type gp3

# Attach volume to instance
aws ec2 attach-volume \
  --volume-id vol-xxxxxx \
  --instance-id i-xxxxxx \
  --device /dev/sdf

# Create a snapshot
aws ec2 create-snapshot \
  --volume-id vol-xxxxxx \
  --description "backup before upgrade"

# Create volume from snapshot (e.g. in another AZ)
aws ec2 create-volume \
  --availability-zone us-east-1b \
  --snapshot-id snap-xxxxxx

# Modify volume (resize/change type live)
aws ec2 modify-volume \
  --volume-id vol-xxxxxx \
  --size 50 --volume-type gp3

## How it works
1. EBS volume created in a specific AZ
2. Attached to an EC2 instance in that same AZ via the 
   hypervisor's network layer (not physically local)
3. OS sees it as a block device (like /dev/xvdf), needs to be
   formatted/mounted to be usable
4. To move data to another AZ/region: take a snapshot → copy 
   snapshot if needed → create new volume from snapshot in 
   target AZ
5. Root volume (boot volume) is deleted by default on instance
   termination unless "DeleteOnTermination" is set to false

## Exam domain(s) checklist
- [ ] Design Resilient Architectures (26%) — snapshots for backup/
      DR, cross-AZ/region volume recovery
- [ ] Design Cost-Optimized Architectures (20%) — choosing right
      volume type (sc1/st1 vs gp3 vs io2) for the workload
- [ ] Design High-Performing Architectures (24%) — IOPS/throughput
      tuning per workload type

## Lab notes
(يتوضاف بعد ما تكمل الـ hands-on)

## Exam-style questions

**Q1:** A company wants to migrate an EBS volume's data from 
us-east-1a to us-east-1b. What's the correct approach?
<details><summary>Answer</summary>Create a snapshot of the volume, 
then create a new volume from that snapshot in us-east-1b. EBS 
volumes cannot move between AZs directly.</details>

**Q2:** True or False: You must stop an EC2 instance to increase 
the size of its attached gp3 EBS volume.
<details><summary>Answer</summary>False - Elastic Volumes lets you 
modify size, type, and IOPS on a live, in-use volume without 
detaching or stopping the instance.</details>

**Q3:** Which EBS volume type would you choose for a NoSQL 
database requiring consistent sub-millisecond latency and very 
high IOPS?
<details><summary>Answer</summary>io2 (or io2 Block Express) - 
Provisioned IOPS SSD, designed for the most demanding low-latency, 
high-IOPS workloads.</details>

## Related services
[[EC2]] [[AMI]] [[KMS]] [[S3]]