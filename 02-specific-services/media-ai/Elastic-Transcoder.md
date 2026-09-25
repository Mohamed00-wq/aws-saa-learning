# Amazon Elastic Transcoder  Legacy Video Transcoding (Retired)

## Purpose

Amazon Elastic Transcoder was the **first-generation file-based video transcoding service**: transcode video files (S3 input) into formats/bitrates for playback across **devices** using **pipelines, jobs, and presets**. **IMPORTANT (SAA current state): AWS discontinued support on November 13, 2025**  the console and resources are no longer accessible AWS's answer is to **migrate to AWS Elemental MediaConvert** (its successor, with more features and better pricing). For the exam, Elastic Transcoder matters as a legacy/EOL answer: essentially **"transcode files → use MediaConvert"**.

## Main use cases

- **Batch/file transcoding** of VOD content into multiple device-friendly renditions (e.g., MP4/H.264, WebM/VP8/VP9, adaptive HLS)
- Simpler **on-demand workflows** with **pipelines** (input bucket → output bucket + notification/lambda per job) for small/medium media apps
- **Presets** out-of-the-box for common resolutions/bitrates (720p, 480p, HD), auto bitrate optimization, thumbnails, audio-only outputs
- **Media workflows** triggered via **SNS notifications** when jobs complete (historic pattern)

## Key features

- **Pipelines**  declarative config (input S3 bucket, output bucket/storage class, IAM role) reused by many jobs
- **Jobs**  one task = input file + output files + predefined/custom **presets** (video/audio/captions/thumbnails), **auto-bitrate** for quality/size balance
- **Scale**  elastic parallelism pay per output-minute (SD/HD/audio-only rates, free tier 20/20/10 min), no min fee
- **Notifications**  SNS on completion job status queried/canceled
- **Formats**  MP4/H.264/AAC, FLV, WebM/VP8+Vorbis, MPEG-2, OGG/Vorbis-FLAC, WAV/FLAC, MP3 audio outputs, etc.

## When to use

- **Only in legacy considerations**  existing apps using its API/pipelines must **migrate to AWS Elemental MediaConvert** (AWS has EOL'd it)
- SAA review: recognize it for **historic confusion**  "transcode video file" on today's exam ⇒ **MediaConvert** (not Elastic Transcoder)
- Reference for presets-based transcoding concepts that carry into MediaConvert templates

## Important limitation

- **Retired (EOL)  November 13, 2025**  console/resources no longer accessible **not for new designs**. Limited feature set vs MediaConvert (no HDR/4K-8K broadcast workflows, no DASH/CMAF richness), **60-second minimum billing** per output, and **no live / no streaming-encoder** capability. All of this reinforces choosing **MediaConvert** for any current VOD need.

## SAA relevance

- "Transcode video file now" → **AWS Elemental MediaConvert** (Elastic Transcoder = **EOL/retired Nov 13, 2025**)
- "Live stream encoding" → **MediaLive** "packaging/live VOD delivery" → MediaPackage/MediaStore/CloudFront
- Exam traps: any answer choosing/to-migrate-to **Elastic Transcoder** is outdated  **prefer MediaConvert** don't treat "[Transcoder+HLS]" as a live tool remember it's **file/job/pipeline/preset** based and **S3-driven**.
- Mnemonic: **"ET retired, MediaConvert is the VOD transcode tool."**