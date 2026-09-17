# Snow — AWS Snow Family

## What it is

AWS Snow Family = **rugged physical devices** that bring AWS storage/compute to you: move data into/out of AWS **offline** (ship it) and run **edge computing** in disconnected environments. Use them when bandwidth makes online transfer too slow — e.g. 1 PB over a 500 Mbps link takes ~8 months, but ships in days/weeks.

## The devices

| Device | Capacity | Compute | Best for |
|---|---|---|---|
| **Snowcone** | 8–14 TB | 2 vCPU / 4 GB | Smallest, portable; battery-powered; edge IoT; can transfer data **online via DataSync** or ship it back |
| **Snowball Edge Storage Optimized** | 80 TB HDD + 1 TB SSD | 40 vCPU / 80 GB | Dozens of TB→PB data transfer + local storage |
| **Snowball Edge Compute Optimized** | 28–42 TB NVMe SSD | up to ~104 vCPU, optional GPU | Compute-heavy edge: video processing, IoT analytics, **ML inference** |
| **Snowmobile** | up to **100 PB** | — | Exabyte-scale migrations — a 45-ft truck literally hauling a data center |

- **Cluster**: up to 3–16 Snowball Edge devices can be clustered for **durability + more capacity** (~PB-scale local storage).
- **Rule of thumb**: <10 TB → online (DataSync/direct upload); 10 TB–10 PB → Snowball Edge; >10 PB → Snowmobile.

## Edge computing on Snow

Devices run select AWS services locally for environments with **no/internittent/low-bandwidth connectivity** (ships, oil rigs, wind farms, factories, remote bases):

- **EC2-compatible instances** (AMIs bundled before shipment) — `sbe1`/`sbe-c`/`sbe-g` types.
- **S3-compatible storage** — object storage locally, subset of S3 API.
- **Lambda via AWS IoT Greengrass** — event-driven processing at the edge.
- **EKS Anywhere** — run Kubernetes on the device.

## How a data transfer job works

1. Create a job in the Snow console: choose device, specify S3 buckets and AMIs/Lambda (pre-loaded before shipping), set notification (SNS).
2. AWS ships a ruggedized, tamper-evident device (KMS-encrypted, 256-bit; GPS/tamper-tracking).
3. Connect to local network, **unlock with manifest + unlock code** (via Snowball client or **AWS OpsHub** UI).
4. Copy data (Snowball client, S3 adapter/SDK, or NFS + `aws s3 cp`).
5. Ship back (import job) — AWS transfers data into your **S3 bucket**. Export jobs go the other way.

## Snowcone online mode

Snowcone ships with a **DataSync agent pre-installed** — connect it to the network and push data online to AWS over DataSync for recurring/mobile workloads, instead of shipping the device back each time.

## Exam domains

- [x] **Secure (30%)** — KMS encryption, tamper-evident hardware, manifest + unlock code
- [x] **Resilient (26%)** — clustered devices for durability; offline transfer independent of network
- [x] **High-Performing (24%)** — high transfer speeds (~1–2+ Gbps per device), parallel devices for PB-scale
- [x] **Cost-Optimized (20%)** — physical transfer cheaper than months of bandwidth; run cost in days, ~10% per day over 10 days

## Key gotchas

1. **Choose Snow when bandwidth-limited or fully disconnected** — not when a fast link exists (use DataSync)
2. **Snowcone = smallest, portable, DataSync-enabled**; **Snowball Edge = the workhorse**; **Snowmobile = only for >10 PB**
3. **Devices can edge-compute**: EC2-compatible, Lambda (Greengrass), S3-compatible storage, EKS Anywhere
4. **Manifest + unlock code are required to use a device** — separate security channels (S3 + console)
5. **Unlock/configure via Snowball client or AWS OpsHub**
6. **Import job = data to AWS; Export job = data from AWS** — buckets/AMIs specified at job creation (pre-configured before shipment)
7. **Cluster multiple Snowball Edges (3–16) for durability and bigger local capacity**
8. **Inter-Region transfers are not the point** — use S3 CRR for region-to-region movement

## Related services

- **S3** — final destination of every import job (Snowball devices hold S3-compatible buckets)
- **DataSync** — the online alternative; also how Snowcone pushes data over the network
- **Storage Gateway** — Tape Gateway on Snowball Edge Storage Optimized for tape migrations
- **EC2 / Lambda / EKS Anywhere** — compute run locally on devices at the edge
- **KMS** — encrypts data on the devices