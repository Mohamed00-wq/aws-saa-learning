# Amazon AI Developer Tools  Purpose-Built AI APIs & Amazon Bedrock

## Purpose

Amazon's **AI developer services** let you add intelligence to applications with **simple API calls**  no ML models to build or train. Two layers: **purpose-built AI services** (pre-trained, specialized APIs for vision, speech, language, search, etc.) and **Amazon Bedrock** (Fully-managed access to **foundation models (FMs) from leading AI providers** + **Amazon Nova** for building **generative AI** apps/agents). For SAA the win is recognizing "which API service". This is the "no-ML-skills-needed" layer of the AWS AI/ML stack  the lower ML platforms are SageMaker etc.

## Main use cases

- **Vision / computer vision**  **Amazon Rekognition**: detect/recognize objects, faces (even in video), moderation, Celebrity/Content/Text detection
- **Speech & audio**  **Polly** (text→lifelike speech, SSML), **Transcribe** (speech→text, streaming + real-time, subtitles/call analytics)
- **Conversational AI**  **Lex** (build voice/text **chatbots** on Alexa's ASR+NLU) → calls **Lambda** backends
- **Language / NLP**  **Translate** (neural MT), **Comprehend** (entity/sentiment/key-phrase/language detection custom classification+entities Comprehend Medical)
- **Document extraction**  **Textract** (OCR of text, **tables, forms/key-value pairs, handwriting**, for processing invoices/forms)
- **Search**  **Kendra** (ML-powered enterprise **search over unstructured text** with natural-language queries)
- **Generative AI**  **Amazon Bedrock**: pick FMs (Anthropic Claude, Amazon Nova, etc.) via one API customizing/agents/RAG **Amazon Q** (AI assistant) for business/code use cases

## Key features

- **Purpose-built, fully managed AI services**  you only send/BB data via **API/SDK** no infrastructure, pay per call/usage
- **Amazon Rekognition**  object/face/scene detection, **image & video moderation**, face search/indexing, Custom Labels
- **Amazon Polly**  dozens of voices/languages, **SSML**, neural TTS (incl. Newscaster) **Transcribe**  batch + streaming speech-to-text, speaker diarization, custom vocab **Lex**  bots with Lambda + client libraries **Translate/Comprehend**  MT & NLP insights **Textract**  structured extraction (tables, key-value, handwriting) **Kendra**  natural-language enterprise search
- **Amazon Bedrock**  Serverless FM platform (Anthropic, Meta, Cohere, Mistral, Amazon Nova/Titan), **one API**, `InvokeModel`, customizations (fine-tuning, continued pre-training), **Agents**, **RAG/Knowledge Bases**, guardrails, **Model evaluation** model provenance/hallucination-checking value prop
- **Amazon Q**  AI assistants (Business for enterprise/WorkDocs, Developer for AWS code/IDE)
- All integrate with **IAM, CloudWatch (usage), EventBridge**, and work over **VPC endpoints** (many core AI services support private endpoints/containers in-VPC)

## When to use

- Want instant AI features without ML expertise  "just call an API" for OCR, translation, transcription, chatbots, face/content moderation, search
- Processing documents/pdfs (Textract), audio/video (Transcribe, Rekognition), conversations (Lex), multilingual content (Translate/Comprehend)
- Need quick **generative AI** (chat, text/image gen, summarization, agents) with managed FMs  Bedrock (vs self-training)
- A dozen well-known exam scenarios: HR/forms automation, compliance document scanning, call-center transcription/analytics, product photo moderation, video subtitling

## Important limitation

- **You don't tune/train these**  they work out-of-the-box, and you're bound by their **pre-trained scope**, **per-call pricing**, and **latency/region availability**. For **custom model training** or full ML control, you must graduate to **Amazon SageMaker** (or fine-tune FMs via **Bedrock**). They process data you send  **sensitive data** needs customer-managed keys/VPC-endpoint/PrivateLink consideration, and some services have **input-size limits** (docs, video duration) and **regional availability** constraints. Kendra/Personalize are **search/recs specific**, not general AI.

## SAA relevance

- "Have **AI without training ML** → **API services** (Rekognition/Polly/Transcribe/Lex/Translate/Comprehend/Textract/Kendra)"
- "Transcribe audio, **Textract** forms/OCR, **Rekognition** faces/moderation, **Lex** chatbots, **Polly** TTS, **Translate** content, **Comprehend** sentiment/entities, **Kendra** enterprise search"  know each one's job
- "Generative AI / foundation models / agents **without owning infrastructure/Training**" → **Amazon Bedrock** (IS vs SageMaker)
- "Build/train own models, notebooks, full MLOps" → **Amazon SageMaker**
- Exam traps: don't confuse **Bedrock (managed FMs, no training of infra)** with **SageMaker (build/train your own)** **Rekognition** = vision + moderation **Transcribe** = audio-to-text **Polly** = text-to-speech  know the direction of each call.