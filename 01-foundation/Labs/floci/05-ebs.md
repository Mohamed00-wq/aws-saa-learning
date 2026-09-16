# Lab 05 — EBS with Floci

> Persistent block storage for EC2. Create volumes, attach them, snapshot them, and see AZ scoping in action.

**Reference notes:** `services/EBS.md`

## 1. Find your AZ (EBS is AZ-scoped)

```bash
AVAILABILITY_ZONE=us-east-1a
echo $AVAILABILITY_ZONE
```

> **EBS is AZ-scoped** — a volume attaches only to an instance in the same AZ. This is exam constraint #1.

## 2. Create a volume

```bash
VOL_ID=$(aws ec2 create-volume \
  --availability-zone $AVAILABILITY_ZONE \
  --size 10 \
  --volume-type gp3 \
  --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=lab-ebs}]' \
  --query 'Volume.VolumeId' --output text)
echo $VOL_ID
```

**Verify:**

```bash
aws ec2 describe-volumes --volume-ids $VOL_ID --query 'Volumes[0].[VolumeId,Size,VolumeType,State,AvailabilityZone]'
```

## 3. gp3 sizing — IOPS decoupled from size

```bash
# gp3: baseline IOPS is 3000 regardless of size
aws ec2 describe-volumes --volume-ids $VOL_ID --query 'Volumes[0].Iops'
# → 3000   (gp3: IOPS independent of GB)
```

> Unlike gp2 (3 IOPS/GB), gp3 gives 3000 IOPS baseline no matter the disk size. That's why gp3 replaces gp2.

## 4. Launch an instance in the SAME AZ and attach

```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=availability-zone,Values=$AVAILABILITY_ZONE" --query 'Subnets[0].SubnetId' --output text)

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-00000000 \
  --instance-type t2.micro \
  --subnet-id $SUBNET_ID \
  --count 1 \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 attach-volume --volume-id $VOL_ID --instance-id $INSTANCE_ID --device /dev/sdf
```

**Verify:**

```bash
aws ec2 describe-volumes --volume-ids $VOL_ID --query 'Volumes[0].Attachments'
aws ec2 describe-instance-attribute --instance-id $INSTANCE_ID --attribute blockDeviceMapping
```

## 5. Try cross-AZ attach (fails — prove it)

```bash
aws ec2 attach-volume --volume-id $VOL_ID --instance-id $INSTANCE_ID --device /dev/sdg --availability-zone us-east-1b 2>&1 || echo "FAILED — volume is AZ-locked"
```

> EBS volumes cannot attach across AZs. To move between AZs/regions: snapshot → copy → create volume.

## 6. Snapshots (incremental, S3-backed, region-scoped)

```bash
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id $VOL_ID \
  --description "lab snapshot" \
  --query 'Snapshot.SnapshotId' --output text)

aws ec2 describe-snapshots --snapshot-ids $SNAP_ID --query 'Snapshots[0].[SnapshotId,State,VolumeSize]'
```

Wait for `completed`:

```bash
aws ec2 wait snapshot-completed --snapshot-ids $SNAP_ID
echo "snapshot ready"
```

## 7. Create a volume from the snapshot (in another AZ)

```bash
VOL2_ID=$(aws ec2 create-volume \
  --availability-zone us-east-1b \
  --snapshot-id $SNAP_ID \
  --volume-type gp3 \
  --query 'Volume.VolumeId' --output text)

aws ec2 describe-volumes --volume-ids $VOL2_ID --query 'Volumes[0].[AvailabilityZone,SnapshotId]'
```

You've now moved data from AZ-a to AZ-b via the snapshot — the classic DR pattern.

## 8. Volume types compared

```bash
# gp2 (legacy, IOPS = 3 × GB)
aws ec2 create-volume --availability-zone us-east-1a --size 20 --volume-type gp2 \
  --query 'Volume.Iops' --output text   # roughly 60ish (3×20, bursts to 3000)

# io2 — provisioned IOPS, mission-critical
aws ec2 create-volume --availability-zone us-east-1a --size 10 --volume-type io2 --iops 2000

# st1 / sc1 — HDD, cannot be boot volumes
aws ec2 create-volume --availability-zone us-east-1a --size 500 --volume-type st1
aws ec2 create-volume --availability-zone us-east-1a --size 500 --volume-type sc1
```

## 9. Elastic volumes — resize live

```bash
aws ec2 modify-volume --volume-id $VOL_ID --size 20
# Elastic Volumes: resize/type/IOPS change while in use, no downtime
```

## Cleanup

```bash
aws ec2 detach-volume --volume-id $VOL_ID
aws ec2 delete-volume --volume-id $VOL_ID
aws ec2 delete-volume --volume-id $VOL2_ID
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

> Snapshots are separate from AMIs/volumes — delete snapshots explicitly if you want them gone.

## Exam hooks

- EBS is **AZ-scoped** — snapshots are the only way to cross AZ/region
- gp3 = decoupled IOPS vs gp2 = 3 IOPS/GB
- Snapshots incremental, point-in-time, S3-backed
- HDD (st1/sc1) cannot boot
- Root volume deleted on termination; non-root preserved (check `DeleteOnTermination`)
- Encryption can't be added retroactively — use encrypted snapshot

## Gotchas to remember

1. Multi-Attach (io1/io2) requires a **clustered filesystem** (GFS2)
2. Snapshots of encrypted volumes are always encrypted
3. No automatic backups — snapshots are opt-in
4. `DeleteOnTermination` true by default on root volume