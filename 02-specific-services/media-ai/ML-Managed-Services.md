# AWS Managed ML Services — SageMaker & Bedrock for Building/Training AI

## Purpose

The **ML-platform layer** of the AWS AI/ML stack — for when you need to **build, train, tune, and deploy machine-learning models and generative-AI applications** (not just call pre-trained APIs). The two pillars are **Amazon SageMaker AI** (full ML lifecycle: notebooks, data prep, training, tuning, deployment/endpoints, monitoring, and MLOps **for ANY model framing incl. FMs**) and **Amazon Bedrock** (managed **foundation models** + generative AI apps/agents). Around them sit **specialized ML services** (Personalize, Forecast, Fraud Detector, Kendra, DevOps Guru, A2I) that automate specific ML outcomes. For SAA: "managed ML platform / build & train models / serverless inference" → SageMaker; "use pre-trained foundation models for GenAI" → Bedrock.

## Main use cases

- **SageMaker** — Build/Train/Deploy custom ML: **Studio** notebooks(studio), data/feature store, **processing**, built-in algorithms, **Autopilot** (AutoML), **JumpStart** (pre-built models/FMs), **Canvas** (no-code ML for analysts), **Model Registry/Pipelines** (MLOps), **multi-AZ/Serverless inference** endpoints, **Inference Recommender**, **Ground Truth** (labeling) + Augmented AI (human review)
- **Bedrock** — **Generative AI**: pick/swap **FMs** (Anthropic Claude, Amazon Nova, Meta Llama, Cohere, Mistral, etc.) via one `InvokeModel` API; **Knowledge Bases/RAG**, **Agents**, **Guardrails**, fine-tuning/continued pre-training, evaluation, model marketplace
- **Specialized ML services** — **Personalize** (recommendations), **Forecast** (time-series), **Fraud Detector** (fraud risk), **Kendra** (intelligent search), **DevOps Guru** (ML anomaly detection on ops), **Lookout*** (industrial/equipment), **Amazon Q** assistants
- **Data/ML pipeline** — Glue (ETL) → SageMaker data wrangler/feature store → training → SageMaker endpoints; Bedrock apps integrated with **Lambda, API Gateway, S3, DynamoDB**
- **Edge/industrial ML** — SageMaker Edge Manager, AWS Panorama, DeepLens

## Key features

- **Amazon SageMaker AI** — managed notebooks (SageMaker Studio), data preparation, algorithm training + tuning, **distributed training**, **inference (endpoints, serverless, multi-model, model registry)**, **SageMaker Canvas** (no-code) & **Studio Lab**, **Ground Truth/A2I** (labeling + human review), **Pipelines** (CI/CD), model monitoring, **Inference Recommender**; supports TensorFlow/PyTorch/Scikit etc; armed solution char: works with **S3, ECR (custom containers), and AWS Trainium/Inferentia chips**
- **Amazon Bedrock** — Choice of FMs (Anthropic, Amazon **Nova/Titan**, Meta, Cohere, Mistral), single API; data stays **private/enterprise-grade**; **Knowledge Bases (RAG)**, **Agents** (tool use/Lambda), **Guardrails** (safety), **Model evaluation**, serverless; pay per token/hour
- **Specialized services** — Personalize (recs), Forecast (forecasting), Fraud Detector, Kendra (search), DevOps Guru, Lookout (anomalies), health-AI (Comprehend Medical / Transcribe Medical / HealthLake)
- Governance & pricing — CloudWatch, IAM, **SageMaker notebooks/endpoints have instance types**, **Managed services bill per use** (tokens, DPUs, instance-hours)

## When to use

- You have the ML skills/data and need **full model lifecycle control** (train, tune, deploy) → **SageMaker**
- Need **generative AI with pre-built foundation models** without building/training → **Bedrock** (swap models easily)
- AutoML (no-code) — SageMaker **Autopilot/Canvas**; special outcome without writing ML — **Personalize/Forecast/Fraud Detector/Kendra**
- Building RAG apps over your documents, agents that use tools → **Bedrock** (Knowledge Bases, Agents)
- Always choose **managed platforms over DIY EC2** for SAA answers around ML operations

## Important limitation

- **You take on ML/DevOps overhead** — these are *platforms* (you provision/tune training jobs, endpoints cost **by instance hour**, monitor drift); choose **purpose-built AI services** instead when a pre-trained API fits (Textract/Rekognition/Translate...). **Bedrock** gives FMs only — you **don't control infrastructure**, and **both require real usage discipline** for cost; **availability/latency** depends on region + endpoint type (SageMaker Serverless vs multi-AZ). Fine-tuning FMs on Bedrock and big SageMaker training still **consume significant compute/cost**. Don't pick SageMaker/Bedrock for simple OCR/TTS — that's the AI-service layer (see AI-Dev-Tools).

## SAA relevance

- "**Build/train/deploy custom ML models, MLOps, serverless inference**" → **Amazon SageMaker** (incl. Autopilot, Canvas, Ground Truth, Model Registry, Studio)
- "**Generative AI with foundation models / agents / RAG / choose model providers**" → **Amazon Bedrock** (vs running your own)
- "**No-code AutoML / business-user ML**" → SageMaker Canvas/Autopilot; "**recommendations**" → **Personalize**; "**forecasting**" → **Forecast**; "**fraud**" → **Fraud Detector**; "**intelligent search**" → **Kendra**; "**ops anomalies**" → **DevOps Guru**
- "**Label data + human review**" → SageMaker Ground Truth & A2I
- Exam traps: **Bedrock = managed third-party FMs/GenAI; SageMaker = build & train your own** — don't swap; AI-dev services (Rekognition/Textract/Polly...) are **API calls, not SageMaker training**; "air-gapped specialized ML service" ≠ SageMaker — match the specialized service to the use case.