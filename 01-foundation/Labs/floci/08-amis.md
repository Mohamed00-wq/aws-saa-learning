# Lab 08 — AMIs with Floci

> The blueprint for every EC2 instance. Discover images, launch from one, create a custom AMI from a running instance, copy/share, and see what deregistering really deletes.

**Reference notes:** `services/AMIs.md`

## 1. Discover available AMIs

```bash
aws ec2 describe-images --owners self --query 'Images[*].[ImageId,Name,State,RootDeviceType]'
```

Filter by state — only `available` AMIs can launch:

```bash
aws ec2 describe-images --filters "Name=state,Values=available" --query 'Images[*].ImageId'
```

> In floci, the built-in AMI is typically `ami-00000000`. In real AWS you'd filter by owner (`self`/`amazon`), virtualization (`hvm`), and trust.

## 2. Launch an instance from the AMI

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ami-00000000 \
  --instance-type t2.micro \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 describe-instances --instance-ids $INSTANCE_ID \
  --query 'Reservations[0].Instances[0].[InstanceId,ImageId,RootDeviceName]'
```

Note: the instance ID (`i-...`) is **immutable** and independent of the AMI after launch.

## 3. Create a custom AMI from the running instance

```bash
AMI_ID=$(aws ec2 create-image \
  --instance-id $INSTANCE_ID \
  --name "lab-custom-ami-v1" \
  --description "my first golden image" \
  --query 'ImageId' --output text)
echo $AMI_ID
```

**AMI lifecycle:** it goes `pending` → `available`:

```bash
aws ec2 wait image-available --image-ids $AMI_ID
aws ec2 describe-images --image-ids $AMI_ID --query 'Images[0].[State,Name]'
```

> **Best practice:** stop the instance first for data consistency (real AWS). The AMI is a snapshot of the attached EBS volumes + block device mapping.

## 4. Launch from YOUR new AMI

```bash
NEW_INSTANCE=$(aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 describe-instances --instance-ids $NEW_INSTANCE \
  --query 'Reservations[0].Instances[0].ImageId'
```

This is the golden-image pattern: pre-baked AMI → every instance launches identical (fast launch, used by ASGs).

## 5. Block device mapping — override size/type at launch

```bash
aws ec2 run-instances \
  --image-id $AMI_ID \
  --instance-type t2.micro \
  --block-device-mappings '[
    {"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":30,"VolumeType":"gp3"}}
  ]' \
  --query 'Instances[0].InstanceId' --output text
```

> You can override volume size/type at launch **without modifying the AMI** — a key exam point.

## 6. Copy the AMI (region-locked)

```bash
COPIED_AMI=$(aws ec2 copy-image \
  --source-image-id $AMI_ID \
  --source-region us-east-1 \
  --name "lab-custom-ami-copy")
echo $COPIED_AMI
```

> **AMI IDs are region-unique.** Copying creates a new AMI (and copies the underlying snapshots). In real AWS, encrypted AMIs can be copied with a different KMS key.

## 7. Share the AMI

```bash
# AMIs are private by default
aws ec2 describe-image-attribute --image-id $AMI_ID --attribute launchPermissions

# Share with a specific account
aws ec2 modify-image-attribute \
  --image-id $AMI_ID \
  --launch-permission '{"Add":[{"UserId":"123456789012"}]}'

# Or make it public
aws ec2 modify-image-attribute --image-id $AMI_ID --launch-permission '{"Add":[{"Group":"all"}]}'
```

> Sharing an AMI does **NOT** share its underlying snapshots — they must be shared separately (classic exam gotcha).

## 8. Deregister — what does it delete?

```bash
# Get the snapshot created by the AMI
SNAP_ID=$(aws ec2 describe-images --image-ids $AMI_ID --query 'Images[0].BlockDeviceMappings[0].Ebs.SnapshotId' --output text)
echo "snapshot: $SNAP_ID"

# Deregister the AMI
aws ec2 deregister-image --image-id $AMI_ID

# Snapshot still exists!
aws ec2 describe-snapshots --snapshot-ids $SNAP_ID --query 'Snapshots[0].State'
```

> **Deregistering removes the AMI, NOT the snapshots.** Delete them manually or keep paying storage. Running instances keep working.

```bash
aws ec2 delete-snapshot --snapshot-id $SNAP_ID
```

## Cleanup

```bash
aws ec2 terminate-instances --instance-ids $NEW_INSTANCE
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
```

## Exam hooks

- Deregistering AMI ≠ deleting snapshots — separate resources
- Stop instance before creating AMI (data consistency)
- Copy AMI = new ID per region
- Sharing AMI doesn't share snapshots — share both
- **EBS-backed** AMIs persist across stop/start; **instance store-backed** can only be terminated
- Marketplace AMIs may have per-instance licensing fees

## Gotchas to remember

1. AMI IDs unique per region
2. `DeleteOnTermination` affects attached volumes at instance terminate
3. EC2 Image Builder = automated AMI build/test/distribute, schedule periodic rebuilds
4. Community AMIs — verify trust before using