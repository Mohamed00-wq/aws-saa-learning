# AWS Elemental MediaConvert  File-Based Video Transcoding (VOD)

## Purpose

AWS Elemental MediaConvert is a **file-based video transcoding service** for **Video-on-Demand (VOD)**: you give it a source media file (in S3) and job settings it produces broadcast-grade, multi-format, multi-bitrate outputs ready for delivery or archiving  **without running transcoding infrastructure**. It's the **successor to Amazon Elastic Transcoder** (which AWS **discontinued Nov 13, 2025**). For SAA: "transcode/convert **on-demand** video files (S3 in → outputs to S3) with **jobs**" → MediaConvert. Pay per minute of output (Basic tier from ~ $0.0075/min Professional tier for broadcast features).

## Main use cases

- **VOD transcoding**  convert camera/mezzanine masters into **HLS/DASH adaptive-bitrate streams** (streams to many devices)
- **Broadcast/file deliverables**  prep content for **broadcast TV, streaming apps, or archives** (youtube-quality outputs at scale)
- **Multiscreen delivery**  generate multiple renditions/resolutions (SD/HD/4K/**8K**) in parallel per job
- **Content prep for VOD libraries** (Spotify/Netflix-style back catalogs, media houses, e-learning)
- **Post-production**  frame-accurate slices, graphic overlays, watermarking, content protection (DRM), creation of thumbnails/captions

## Key features

- **Transcode via jobs**  input (S3) → job (settings/presets) → outputs (to S3) console/API/SDK **S3 event notifications** + EventBridge orchestration parallelism scales elastically for peak loads
- **Formats/codecs**  broad input support **HLS, CMAF/DASH, MP4, WebM, MXF** ffmpeg-relatives codecs AVC, **HEVC (H.265)**, **VP8/VP9**, AV1 **QVBR** (quality-defined variable bitrate) for smaller high-quality files
- **Profiles/tiers**  **Basic** (web delivery, AVC/VP8/VP9) vs **Professional** (broadcast: 4K/8K, **HDR incl. Dolby Vision**, immersive audio, captions/overlays, DRM-compatible outputs)
- **Leverages scale**  no servers, autoscaling compute for VOD loads, predictable **pay-as-you-go** billing, CloudWatch metrics
- Integrates with the **Media Services** family  MediaLive (live streams), MediaPackage (packaging), MediaStore/Iceberg S3, MediaTailor (ad insertion)  for end-to-end media pipelines

## When to use

- File-based (non-live) content you want to encode/repurpose: any **video files that need conversion, compression, or reformatting for playback**
- Streaming-ready **HLS/DASH renditions** on demand (vs configuring persistent live encoders)
- Automatic, high-scale batch conversion driven by **S3/EventBridge events**  ingest S3, transcode, distribute output S3 + CloudFront
- Migration from **Elastic Transcoder** (which is retired)  replacement/presets conversion is the standard path

## Important limitation

- **Not a live-streaming encoder**  that's **AWS Elemental MediaLive** (MediaConvert is file/VOD). **Job-based**  you submit a job, wait for completion not interactive/low-latency editing. **Cost is per output minute** (aggregated across renditions) and **S3 storage/data-transfer for ingress/egress are extra**. Some advanced features are **Professional-tier only** region availability varies. Output **quality** depends on inputs & settings (it won't "fix" tiny/broken source files).

## SAA relevance

- "**Transcode video files / VOD** (S3 in → encoded outputs)" → **AWS Elemental MediaConvert**
- "**Adaptive bitrate (HLS/DASH), broadcast streaming, content for many devices**" → MediaConvert (jobs + S3/EventBridge)
- "**Live stream encoding**" → **MediaLive** "**live&VOD packaging/serving**" → MediaPackage/MediaStore + CloudFront
- "**Legacy Elastic Transcoder**" → **retired Nov 13, 2025 → migrate to MediaConvert**
- Exam traps: MediaConvert = **file-based, on-demand (jobs)** MediaLive = **live, streaming (channels)** don't confuse HLS packaging with encoding remember it reads/writes **S3** and is **evently** scalable/pay-per-output-minute. Mnemonic: **“Convert files = MediaConvert · Convert live = MediaLive.”**