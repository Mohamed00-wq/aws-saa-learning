# AMI — Amazon Machine Image

## What it is

An AMI is the immutable blueprint from which every EC2 instance is launched. It packages everything needed to boot a fully functional server: the operating system, pre-installed software, configuration files, and a block device mapping that defines which EBS volumes (and how big) are attached at launch.

AMIs solve the problem of consistency and speed. Instead of launching a blank instance and manually configuring it every time, you launch 100 instances from the same AMI and they all come up identical. This is the foundation of everything from simple deployments to auto-scaled architectures to disaster recovery strategies.

---

## Key concepts

### AMI types (by origin)

| Type | Description | Use case |
|---|---|---|
| **AWS-provided** | Official AMIs maintained by AWS (Amazon Linux 2, Amazon Linux 2023, Windows Server, Ubuntu, etc.) | General-purpose workloads where you want a well-maintained, patched base |
| **AWS Marketplace** | Third-party AMIs sold through the AWS Marketplace (e.g. pre-configured SAP, Fortinet, Bitnami) | Commercial software that needs licensing |
| **Community** | AMIs shared publicly by other AWS users | Niche configurations, but **verify trust** — community AMIs can contain anything |
| **Custom** | AMIs you create from an existing instance or via EC2 Image Builder | Production workloads where you need specific software, configs, or hardening baked in |

### EBS-backed vs Instance Store-backed

- **EBS-backed AMI** (dominant today): the root volume is an EBS snapshot. When you launch an instance, AWS creates a new EBS volume from the snapshot and attaches it. Data persists across stop/start cycles.
- **Instance Store-backed AMI**: the root volume is on physically attached local storage. Data is lost when the instance stops or terminates. These AMIs are rare today and mainly used for specific high-IOPS workloads.

**Exam note:** instance store-backed AMIs cannot be stopped — they can only be running or terminated. If a question says "an instance cannot be stopped," think instance store.

### AMI lifecycle

When you create an AMI from an existing instance:
1. AWS snapshots all attached EBS volumes (the root volume and any additional volumes specified in the block device mapping).
2. The AMI goes through a `pending` state while the snapshot is being created.
3. The AMI becomes `available` and is assigned an AMI ID (e.g. `ami-0abc1234def56789`).
4. The AMI is registered with a block device mapping that tells future launches which snapshots to use and how to map them to device names.

**Best practice:** stop the instance before creating an AMI to ensure data consistency. If the instance is running, there may be in-flight writes that haven't been flushed to disk, leading to an inconsistent snapshot (similar to pulling a USB drive without ejecting).

### AMI copying and sharing

**Copy AMI:**
- AMIs are **region-locked** — they exist in a single region. To use an AMI in another region, you must use Copy AMI.
- Copying creates a new AMI in the target region backed by new EBS snapshots in that region.
- Copying across regions also works for migrating workloads or setting up disaster recovery.
- Encrypted AMIs can be copied, and you can change the encryption key during copy (e.g. copy from an AMI encrypted with a source-account KMS key to one encrypted with a destination-account KMS key).

**Sharing AMIs:**
- By default, AMIs are private (only visible to your account).
- You can share an AMI with specific AWS accounts (by account ID) or make it public.
- Sharing an AMI does **not** share the underlying snapshots — you must explicitly share those too if you want the recipient to be able to launch instances from the AMI.
- Shared AMIs appear in the recipient's account under "Private AMIs" and can be used exactly like any other AMI.

### Launch permissions

Launch permissions control who can use your AMI:
- **Private** — only your account.
- **Explicit** — shared with specific AWS account IDs.
- **Public** — available to anyone.

Launch permissions are set on the AMI, not on the snapshots. The recipient still needs snapshot access to actually launch instances.

### Block device mapping

The block device mapping defines:
- Which EBS volumes to create from which snapshots.
- The device names (e.g. `/dev/xvda`, `/dev/sdf`).
- Volume size (can be larger than the snapshot size, allowing expansion).
- Volume type (gp3, io2, etc.) — you can choose a different type than the original snapshot.
- Whether to delete the volume on instance termination (`DeleteOnTermination`).
- Whether the volume is encrypted.

**Key exam point:** when launching from an AMI, you can override the block device mapping to customize volume sizes, types, or encryption — without modifying the AMI itself.

### Deregistering and cleanup

When you deregister an AMI:
- The AMI is removed from your account and cannot be used to launch new instances.
- The **associated EBS snapshots are NOT deleted automatically** — you must delete them manually to reclaim storage and avoid charges. This is a classic exam/cost trap.
- Instances already running from the deregistered AMI are unaffected.

---

## Architecture deep dive

### How AMIs work under the hood

When AWS launches an instance from an EBS-backed AMI:
1. The AMI's block device mapping is consulted to find the source snapshot IDs.
2. For each volume in the mapping, AWS creates a new EBS volume in the target AZ using the specified snapshot.
3. If the volume size in the block device mapping is larger than the snapshot size, the volume is created at the larger size and the extra space is available (but needs to be extended by the OS).
4. The new EBS volumes are attached to the instance at the specified device names.
5. The instance boots from the root volume.

This means:
- Launching from an AMI is essentially "copy-on-write" — the new EBS volume starts as a clone of the snapshot.
- Multiple instances launched from the same AMI share no data after launch — they are fully independent copies.
- The speed of launch depends on snapshot size and how much data needs to be loaded.

### AMI and snapshots relationship

| Action | What happens |
|---|---|
| Create AMI from instance | Snapshots all EBS volumes, registers the AMI |
| Create volume from snapshot | Creates a new EBS volume in the same region/AZ as the snapshot |
| Copy AMI cross-region | Copies all underlying snapshots to the target region, registers a new AMI |
| Deregister AMI | Removes the AMI registration; snapshots remain |
| Delete snapshot | Does NOT deregister the AMI — but new launches from the AMI will fail if the snapshot is needed |

**Incremental snapshots:** the first snapshot is a full copy. Subsequent snapshots are incremental — only the blocks that changed since the last snapshot are stored. This means creating snapshots of a 1 TB volume is fast if only a few GB changed.

### EC2 Image Builder

A fully managed AWS service for building, testing, and publishing AMIs:
- You define a **recipe** (base AMI + components to install + validation tests).
- Image Builder runs the recipe on an EC2 instance, creates an AMI, runs your tests, and optionally distributes the result to multiple regions.
- You can set a **schedule** to rebuild AMIs periodically (e.g. weekly to pick up security patches).
- Distribution settings let you share the resulting AMI with specific accounts and regions automatically.

This is the AWS-recommended way to keep AMIs up to date, rather than manually creating and copying them.

---

## Exam domain(s)

- [ ] Design Secure Architectures (30%)
- [x] **Design Resilient Architectures (26%)** — AMI as the basis for disaster recovery (copy AMI cross-region), self-healing with ASG
- [x] **Design High-Performing Architectures (24%)** — pre-baked AMIs for fast launch, EC2 Image Builder for automated maintenance
- [x] **Design Cost-Optimized Architectures (20%)** — deleting snapshots after deregistering, choosing the right base AMI

---

## Advanced gotchas & edge cases

1. **Deregistering an AMI does NOT delete its snapshots.** You must delete snapshots manually to avoid storage costs. This is tested frequently.

2. **Creating an AMI from a running instance can lead to data inconsistency.** Stop the instance first for consistent snapshots, unless you're using file-system-aware tools.

3. **AMI IDs are unique per region.** The same AMI copied to another region gets a different ID.

4. **Sharing an AMI does not share its snapshots.** You must share snapshots explicitly, or the recipient will not be able to launch instances.

5. **You can change EBS volume type when launching from an AMI.** Even if the AMI was created with gp2, you can launch with io2 by overriding the block device mapping.

6. **Amazon Linux AMIs are region-specific and patched regularly.** An AMI from 6 months ago may not have the latest security patches — use Image Builder or build fresh AMIs periodically.

7. **Marketplace AMIs may have licensing costs.** Be aware of per-instance or per-hour licensing fees associated with Marketplace AMIs.

8. **Instance store-backed AMIs cannot be stopped, only terminated.** This is a key differentiator for exam scenarios.

---

## Exam-style questions

**Q1.** A company runs an Auto Scaling Group that launches new instances during peak traffic. Each new instance takes several minutes to boot and install required software before it can serve traffic. What should a Solutions Architect do to reduce this launch time?
- A) Increase the instance type size
- B) Create a custom AMI with software pre-installed and use it in the launch template
- C) Use a larger EBS volume
- D) Enable detailed monitoring on the ASG

<details><summary>Answer</summary>
**B** — a custom AMI with pre-installed software eliminates the installation step at launch, drastically reducing time to serve. Instance type (A) and EBS size (C) do not affect software installation time. Monitoring (D) has no impact on launch speed.
</details>

**Q2.** A Solutions Architect created an AMI in us-east-1 and needs to launch identical instances in eu-west-1. What must they do?
- A) The AMI is automatically available in all regions
- B) Copy the AMI to eu-west-1
- C) Create a new snapshot in eu-west-1 manually
- D) Change the AMI's region setting

<details><summary>Answer</summary>
**B** — AMIs are region-locked. Copy AMI handles the snapshot copying and AMI registration in the target region automatically.
</details>

**Q3.** An engineer deregisters a custom AMI but forgets to delete the associated snapshots. What is the impact?
- A) Instances already launched from the AMI stop working
- B) New instances cannot be launched from the AMI
- C) The snapshots continue to incur storage charges
- D) Both B and C

<details><summary>Answer</summary>
**D** — deregistering removes the AMI (so no new launches are possible), but snapshots are separate resources and remain, incurring S3 storage charges until explicitly deleted.
</details>

**Q4.** A company needs to distribute a custom AMI to 12 different AWS accounts within the same region. What is the most efficient approach?
- A) Make the AMI public
- B) Share the AMI with each account ID and share the underlying snapshots
- C) Create a copy of the AMI for each account
- D) Export the AMI to S3 and have each account import it

<details><summary>Answer</summary>
**B** — sharing an AMI with specific accounts (plus the underlying snapshots) is the controlled, secure approach. Making it public (A) exposes it to everyone. Copying for each account (C) is unnecessary and wastes storage. Import/export (D) is for converting VM images, not sharing.
</details>

---

## Related services

- [[EC2]] — the service that launches instances from AMIs
- [[EBS]] — EBS-backed AMIs rely on snapshots; launching from an AMI creates new EBS volumes
- [[Auto-scaling]] — ASGs use a Launch Template that references an AMI to launch identical instances at scale
- [[EC2 Image Builder]] — automates AMI creation, testing, and distribution on a schedule
- [[S3]] — AMI export/import uses S3 as an intermediary for VM files
- [[KMS]] — AMIs can be encrypted with KMS keys; cross-account copy may require key sharing
