 # ![AMI](/images/icons/arch/Arch_Amazon-EC2-Image-Builder_64.svg) AMI — Amazon Machine Image 

 


## What it is

An AMI is the immutable blueprint from which every EC2 instance is launched. It packages everything needed to boot a fully functional server: the operating system, pre-installed software, configuration files, and a block device mapping that defines which EBS volumes (and how big) are attached at launch.

AMIs solve the problem of consistency and speed. Instead of launching a blank instance and manually configuring it every time, you launch 100 instances from the same AMI and they all come up identical. This is the foundation of everything from simple deployments to auto-scaled architectures to disaster recovery strategies.

## AMI types

| Type | Notes |
|---|---|
| **AWS-provided** | Amazon Linux, Ubuntu, Windows — well-maintained base |
| **Marketplace** | Third-party AMIs (SAP, Fortinet) — watch for licensing costs |
| **Community** | Shared by other users — verify trust |
| **Custom** | Built from an instance or via EC2 Image Builder |

**EBS-backed vs Instance Store-backed:**
- **EBS-backed** (most common): root volume from snapshot, persists across stop/start.
- **Instance Store-backed**: root on local storage, **cannot be stopped** — only terminated.

## Lifecycle

1. Snapshot all attached EBS volumes → AMI goes `pending`.
2. Becomes `available` with an AMI ID → registered with a block device mapping.
3. **Stop the instance first** for data consistency.

## Copy & share

- **Region-locked**: use Copy AMI to use in another region (copies snapshots too).
- **Sharing**: AMI is private by default. Share with specific accounts or make public.
- **Key**: sharing an AMI does **not** share its underlying snapshots — share them separately.
- Encrypted AMIs can be copied with a different KMS key.

## Block device mapping

Defines snapshot source, device name, volume size/type, `DeleteOnTermination`, and encryption.
**Exam point:** you can override volume size/type at launch without modifying the AMI.

## Deregistering

Deregistering removes the AMI but **does NOT delete snapshots** — delete them manually or pay storage costs. Running instances are unaffected.

## EC2 Image Builder

Managed service for automated AMI creation: recipe (base + components + tests) → build → distribute to multiple regions. Set a schedule for periodic rebuilds.

## Exam domains

- [ ] Secure Architectures (30%)
- [x] **Resilient Architectures (26%)** — copy AMI cross-region for DR, ASG with AMI for self-healing
- [x] **High-Performing (24%)** — pre-baked AMIs for fast launch, Image Builder
- [x] **Cost-Optimized (20%)** — clean up snapshots after deregistering

## Key gotchas

1. Deregistering AMI ≠ deleting snapshots — both are separate resources
2. Stop instance before creating AMI to avoid inconsistent snapshots
3. AMI IDs are unique per region — copy = new ID
4. Sharing AMI doesn't share snapshots — must share both
5. Instance store-backed AMIs can only be terminated, not stopped
6. Marketplace AMIs may have per-instance licensing fees

## Related services

- [[EC2]] — launches instances from AMIs
- [[EBS]] — AMI snapshots create new EBS volumes at launch
- [[Auto-scaling]] — ASGs use Launch Template referencing an AMI
- [[EC2 Image Builder]] — automated AMI build/test/distribute
- [[KMS]] — AMI encryption; cross-account copy may need key sharing
