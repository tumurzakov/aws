# Part 3: AI and Machine Learning Services

AWS provides AI/ML services at three levels: pre-trained AI services (no ML knowledge needed), Bedrock (foundation models with managed infrastructure), and SageMaker (full ML platform for custom models). This section covers when to use each and how they work.

---

## Amazon Bedrock

### Overview

Bedrock is a managed service for accessing foundation models (FMs) from Amazon (Titan), Anthropic (Claude), Meta (Llama), Cohere, AI21, Stability AI, and others. It provides a unified API for model invocation, fine-tuning, and RAG (Retrieval-Augmented Generation) without managing infrastructure. Bedrock is the default starting point for most GenAI workloads on AWS.

### How It Works

You call the Bedrock InvokeModel API with a model ID, input prompt, and inference parameters. Bedrock routes your request to the appropriate model and returns the response. The Converse API provides a unified chat interface across models — same request format regardless of the underlying model, which simplifies model switching.

For streaming responses, use InvokeModelWithResponseStream or Converse with streaming enabled. This is critical for chat applications where time-to-first-token matters more than total latency.

### On-Demand vs Provisioned Throughput vs Cross-Region Inference

**On-demand**: Pay per input/output token. No commitment, no guaranteed throughput. Suitable for variable or low-volume workloads. Subject to account-level rate limits that can be restrictive for production.

**Provisioned Throughput**: Reserve a fixed number of model units for guaranteed capacity. Sold in 1-month or 6-month terms. More cost-effective for sustained, predictable workloads. Required for custom/fine-tuned models. You commit to a fixed throughput (measured in model units) regardless of utilization.

**Cross-Region Inference**: Routes requests across multiple regions automatically to find available capacity. Reduces throttling for on-demand workloads by spreading load across regions. You opt in via an inference profile, and Bedrock handles routing transparently. Tradeoff: data leaves your primary region, which may have compliance implications.

### Knowledge Bases (Managed RAG)

Knowledge Bases let you connect Bedrock to your data sources (S3, web crawlers, Confluence, SharePoint, etc.) for RAG. Bedrock automatically chunks documents, generates embeddings, and stores them in a vector database (OpenSearch Serverless, Aurora PostgreSQL, Pinecone, or Redis Enterprise Cloud).

At query time, Bedrock retrieves relevant chunks from the vector store, augments the prompt with context, and generates a grounded response. This is fully managed — no need to build your own RAG pipeline.

**Chunking strategies**: Fixed-size (simple, predictable), hierarchical (parent-child chunks for context), semantic (using embedding similarity to find natural boundaries), or custom (you provide pre-chunked data). The chunking strategy dramatically affects retrieval quality — test different approaches.

**Metadata filtering**: You can attach metadata to documents (department, date, document type) and filter at query time so that only relevant chunks are retrieved. This is essential for multi-tenant systems or when query context narrows the relevant document set.

**Hybrid search**: Knowledge Bases support combining vector similarity search with keyword-based search for better retrieval accuracy than either approach alone.

### Guardrails

Guardrails let you apply safety policies to model interactions. You can configure:
- Content filters (hate, insults, sexual, violence) with configurable thresholds
- Denied topics (specific subjects the model should not discuss)
- Word filters (profanity, custom blocked words)
- Sensitive information filters (PII detection and redaction)
- Contextual grounding checks (hallucination detection against source documents)

Guardrails are applied independently of the model and work with any model available in Bedrock. They can be applied to both input (prompt) and output (response). Guardrails add approximately 100-200ms of latency per invocation.

Important: Guardrails can be used with non-Bedrock models too — you can apply them via the ApplyGuardrail API to any text, regardless of which model generated it. This enables consistent safety policies across your entire AI stack.

### Agents

Bedrock Agents enable models to take actions by calling APIs and accessing knowledge bases. You define actions (as OpenAPI schemas or Lambda functions), and the agent autonomously plans and executes multi-step tasks using ReAct-style reasoning.

Agent architecture:
- **Instructions**: Natural language description of the agent's role and behavior
- **Action groups**: Sets of APIs the agent can call, defined as OpenAPI schemas with Lambda function backends
- **Knowledge bases**: One or more Knowledge Bases the agent can query for information
- **Guardrails**: Safety policies applied to the agent's inputs and outputs

Agents handle the full reasoning loop: understand the user's request, decide which tools to call, call them, interpret results, and decide next steps. They maintain conversation history within a session.

**Multi-agent collaboration**: You can create supervisor agents that delegate to specialized sub-agents. This enables complex workflows where a routing agent decides which specialist agent should handle the request.

### Flows

Bedrock Flows (previously called Prompt Flows) provide a visual, low-code way to build multi-step AI workflows. You chain together nodes — prompts, knowledge base queries, Lambda functions, conditions, and iterators — into a DAG (directed acyclic graph).

Use Flows when you want deterministic orchestration of AI steps (not autonomous agent behavior). For example: retrieve context from a knowledge base, run it through a summarization prompt, then classify the result. Flows give you explicit control over each step, while Agents give the model autonomy to decide its own steps.

### Model Customization

**Fine-tuning**: Train a model on your data to improve performance on specific tasks. You provide training data (prompt-completion pairs), and Bedrock fine-tunes the base model. The fine-tuned model is private to your account. Requires provisioned throughput to serve.

**Continued pre-training**: Feed unlabeled domain-specific data (industry jargon, internal documents) to extend the model's knowledge. Less targeted than fine-tuning but useful for domain adaptation.

**Model evaluation**: Bedrock provides tools to evaluate model quality against your use cases. You can run automatic evaluations (using reference answers) or human evaluations (using a workforce to rate outputs). This helps you compare models and fine-tuned variants objectively.

### Batch Inference

For non-real-time workloads, Bedrock supports batch inference. You provide a JSONL file of prompts in S3, and Bedrock processes them all, writing results back to S3. Batch inference is cheaper than real-time invocation (typically 50% discount) and is ideal for bulk document processing, classification, or data enrichment where latency does not matter.

### Model Invocation Logging

You can enable logging of all Bedrock API calls to S3 and/or CloudWatch Logs. This captures the full input prompts and output responses, which is essential for auditing, compliance, and debugging. Note that this can generate significant data volume and cost.

### Limits and Quotas

- Model availability varies by region — check before architecting
- On-demand rate limits vary by model (requests per minute and tokens per minute, both adjustable)
- Maximum input/output tokens depend on the model (e.g., Claude: 200K input context)
- Knowledge Base: up to 10 data sources per knowledge base
- Guardrails: applied per invocation, add 100-200ms latency
- Batch inference: maximum 50,000 records per job
- Agents: maximum 5 action groups, 5 knowledge bases per agent

### Pricing Model

Bedrock charges per token (input and output separately). Prices vary significantly by model — Claude Haiku is ~10x cheaper than Claude Opus. Provisioned throughput charges per model unit per hour. Knowledge Bases charge for embedding generation, vector storage (through the underlying vector DB), and retrieval queries. Guardrails charge per text unit assessed. Batch inference is typically 50% of on-demand pricing.

### Edge Cases and Gotchas

- Not all models are available in all regions. Some newer models launch in us-east-1 and us-west-2 first.
- On-demand rate limits are per-account and can be restrictive for production workloads. Request increases weeks before launch.
- Knowledge Base retrieval quality depends heavily on chunking strategy. Default fixed-size chunking is often suboptimal. Test semantic or hierarchical chunking.
- Knowledge Base embeddings must be regenerated if you change the embedding model or chunking strategy.
- Model invocation logging logs the full prompt and response — ensure this complies with your data handling policies. For regulated industries, this can be a compliance requirement and a data privacy risk simultaneously.
- Fine-tuned models require provisioned throughput (no on-demand) — this means a minimum ongoing cost.
- Cross-region inference sends data to other regions. Verify this is acceptable for your compliance requirements before enabling.
- Agent reasoning can be unpredictable. Always set Guardrails on agents and test edge cases thoroughly.
- Bedrock does not support streaming for batch inference — batch is for throughput, not latency.
- Converse API is the recommended interface for new applications. InvokeModel requires model-specific request formats.

### Common Interview Questions

**Q: What is RAG and how does Bedrock Knowledge Bases implement it?**
A: RAG (Retrieval-Augmented Generation) enhances model responses with relevant external knowledge. Bedrock Knowledge Bases automate the RAG pipeline: ingest documents from S3 or other sources, chunk them, generate embeddings, store them in a vector database, and at query time retrieve relevant chunks to augment the prompt. This grounds the model's response in actual data, reducing hallucination.

**Q: How do you ensure responsible AI use with Bedrock?**
A: Use Guardrails to apply content filters, denied topics, word filters, PII redaction, and grounding checks. Enable model invocation logging for audit trails. Use IAM to control which users/roles can access which models. For sensitive workloads, use VPC endpoints to keep traffic off the public internet.

**Q: When would you use Bedrock Agents vs Bedrock Flows?**
A: Use Agents when the task requires autonomous reasoning — the model decides which tools to call and in what order based on the user's request. Use Flows when you want deterministic, step-by-step orchestration where you control the exact sequence of AI operations. Agents are flexible but less predictable; Flows are rigid but fully controllable.

**Q: What is the first thing to check when RAG quality is poor?**
A: Check retrieval quality before blaming the model. Verify that the right chunks are being retrieved — poor chunking, weak embeddings, missing metadata filters, or wrong similarity thresholds usually cause more quality problems than model choice. Use the Knowledge Base retrieve-only API (without generation) to inspect what chunks are returned.

### Interactions with Other Services

- **Lambda**: Custom actions for Bedrock Agents are implemented as Lambda functions.
- **S3**: Data source for Knowledge Bases, storage for model invocation logs, input/output for batch inference.
- **OpenSearch Serverless**: Vector database for Knowledge Bases.
- **Aurora PostgreSQL**: Alternative vector database for Knowledge Bases (using pgvector).
- **Step Functions**: Orchestrate multi-step AI workflows with Bedrock InvokeModel steps.
- **CloudWatch**: Metrics and logs for monitoring Bedrock usage (invocations, latency, throttles).
- **EventBridge**: React to Bedrock model invocation events.
- **API Gateway**: Expose Bedrock-powered APIs to external consumers.

---

## Amazon SageMaker

### Overview

SageMaker is AWS's full-featured ML platform for building, training, and deploying custom machine learning models. Unlike Bedrock (which provides pre-built foundation models), SageMaker gives you control over the entire ML lifecycle: data preparation, model training, hyperparameter tuning, model hosting, and monitoring.

Use SageMaker when you need custom models trained on your data, when you need to control the training process, or when your use case requires specialized model architectures not available as foundation models.

### Key Components

**SageMaker Studio**: Web-based IDE for ML development. Includes notebooks (JupyterLab), experiment tracking, model registry, and pipeline visualization. Studio is the central workspace for data scientists.

**SageMaker Notebooks**: Managed Jupyter notebook instances with pre-installed ML frameworks (PyTorch, TensorFlow, scikit-learn, XGBoost). Available as Studio notebooks (persistent) or notebook instances (standalone).

**SageMaker Processing**: Run data processing and feature engineering jobs on managed infrastructure. You provide a processing script and a container, SageMaker provisions instances, runs the job, and terminates them.

**SageMaker Training**: Managed model training with automatic infrastructure provisioning. You specify the training algorithm (built-in or custom container), input data (from S3), instance type, and hyperparameters. SageMaker provisions the instances, runs training, saves the model to S3, and terminates the instances. Supports distributed training across multiple instances and GPUs.

**SageMaker Endpoints**: Managed real-time inference endpoints. You deploy a model to an endpoint, and SageMaker provisions instances, load-balances across them, and handles auto-scaling. Supports multiple model variants on the same endpoint for A/B testing.

### Training Options

**Built-in algorithms**: SageMaker provides optimized implementations of common algorithms (XGBoost, Linear Learner, K-Means, Object Detection, etc.). These are pre-optimized for SageMaker's distributed infrastructure and are the fastest way to train standard models.

**Bring your own container (BYOC)**: Package your custom training code in a Docker container and run it on SageMaker. Use when you need a custom algorithm, framework, or training loop.

**Script mode**: Provide a training script (Python) and let SageMaker use a pre-built framework container (PyTorch, TensorFlow, etc.). Simpler than BYOC — you write the training code, SageMaker handles the container and infrastructure.

**SageMaker JumpStart**: A model hub with hundreds of pre-trained models (foundation models, computer vision, NLP, tabular) that you can fine-tune or deploy directly. This is the middle ground between Bedrock (fully managed FM) and custom training — you get pre-trained models but can fine-tune and host them yourself.

### Hosting and Inference

**Real-time inference**: Persistent endpoints for low-latency predictions. You pay for the instances whether they are handling requests or not. Supports auto-scaling based on invocations or custom CloudWatch metrics.

**Serverless inference**: SageMaker provisions compute on demand and scales to zero when not in use. Higher cold-start latency (seconds) but no cost when idle. Use for intermittent traffic.

**Batch transform**: Process large datasets offline. SageMaker reads input from S3, runs predictions, and writes results to S3. No persistent endpoint needed. Use for bulk scoring, periodic predictions, or data enrichment.

**Asynchronous inference**: For requests that take longer to process (up to 15 minutes). SageMaker queues requests, processes them, and stores results in S3. Supports auto-scaling to zero. Use for large payloads, model ensembles, or complex post-processing.

### SageMaker Pipelines

ML CI/CD: define, automate, and manage end-to-end ML workflows as DAGs. Steps include processing, training, evaluation, condition checking, model registration, and deployment. Pipelines integrate with the model registry for versioning and approval workflows.

### Feature Store

A centralized repository for ML features. Ingest feature values once, reuse them across training and inference. Supports both online store (low-latency lookup for real-time inference) and offline store (S3-based for batch training). Feature Store solves the training-serving skew problem — features used during training are exactly the same as those used during inference.

### Model Registry

A central catalog for versioning, tracking, and managing model artifacts. Models go through approval statuses (PendingApproval, Approved, Rejected). Combined with Pipelines, this enables a full CI/CD workflow: train, evaluate, register, approve, deploy.

### Model Monitor

Continuously monitors deployed models for data drift (input data distribution changes), model quality drift (accuracy degradation), bias drift (fairness metrics change), and feature attribution drift. Generates alerts via CloudWatch when drift is detected.

### SageMaker Canvas

A no-code ML tool for business analysts. Upload data (CSV, from S3, or from databases), select a target column, and Canvas trains a model automatically. Supports classification, regression, forecasting, and NLP. No ML expertise required.

### Limits and Quotas

- Training: up to 28 days per job, instance types up to p5.48xlarge (8 GPUs)
- Real-time endpoints: up to 500 per account (soft limit)
- Serverless inference: max 6 MB payload, 60-second timeout, 20,480 MB memory
- Batch transform: up to 100 MB per record (with splitting), instance types up to ml.p3.16xlarge
- Pipelines: up to 500 steps per pipeline
- Feature Store: up to 2,500 feature groups per account

### Pricing Model

SageMaker charges per instance-hour for training and hosting. Different instance families for different needs: ml.m (general purpose), ml.c (compute-optimized), ml.p/ml.g (GPU). Training costs stop when the job completes. Hosting costs continue as long as the endpoint is running. Serverless inference charges per millisecond of compute time and per GB of data processed. Notebooks charge per instance-hour.

### Edge Cases and Gotchas

- Real-time endpoints cost money 24/7 even with no traffic. Use serverless inference or auto-scaling with scale-to-zero for intermittent workloads.
- Training jobs can fail after hours due to data issues, code bugs, or infrastructure problems. Use SageMaker Debugger and checkpointing to avoid losing progress.
- Model deployment involves creating a model, an endpoint configuration, and an endpoint — three separate resources. CDK and the SageMaker SDK simplify this but the underlying complexity matters for debugging.
- Serverless inference has cold starts of several seconds. Not suitable for low-latency applications.
- Feature Store online store has a per-record limit of 350 features and 100 KB per record.
- GPU instances (p3, p4, g5) have limited availability. Request capacity in advance for large training jobs.
- Model Monitor requires a baseline dataset to detect drift. Without it, monitoring is not meaningful.
- Data in SageMaker Training jobs is copied from S3 to local instance storage before training starts. For very large datasets, use File mode (copy) or Pipe mode (stream) depending on access patterns.

### Common Interview Questions

**Q: When would you use SageMaker vs Bedrock?**
A: Use Bedrock when you want to use pre-built foundation models (text generation, summarization, classification) without training your own model. Use SageMaker when you need custom models trained on your data, specialized architectures (computer vision, time series, recommendation), or full control over the training and hosting process. Many projects use both: Bedrock for GenAI features and SageMaker for custom ML models.

**Q: How does SageMaker handle model deployment and updates?**
A: SageMaker endpoints support multiple model variants with configurable traffic splitting for A/B testing and canary deployments. You create a new endpoint configuration with the updated model, then update the endpoint. SageMaker performs a rolling deployment with no downtime. For blue/green deployments, use deployment guardrails that automatically roll back if CloudWatch alarms fire.

**Q: What is training-serving skew and how does SageMaker address it?**
A: Training-serving skew occurs when the features used during training differ from those used during inference (due to different code paths, data processing pipelines, or timing). SageMaker Feature Store solves this by providing a single source of truth for feature values — the same features used for training (offline store) are served to the model in production (online store).

### Interactions with Other Services

- **S3**: Primary storage for training data, model artifacts, and batch transform results.
- **ECR**: Custom training and inference containers are stored in ECR.
- **Lambda**: Invoke SageMaker endpoints from Lambda for serverless AI applications.
- **Step Functions**: Orchestrate ML pipelines as Step Functions workflows (alternative to SageMaker Pipelines).
- **CloudWatch**: Metrics, logs, and alarms for training jobs, endpoints, and Model Monitor.
- **VPC**: Training jobs and endpoints can run inside your VPC for network isolation.
- **IAM**: Fine-grained access control for notebooks, training jobs, endpoints, and model artifacts.
- **Glue**: Data preparation and ETL before training.

---

## AWS AI Services (Pre-Built)

AWS provides pre-trained AI services for common tasks. These require no ML knowledge — you call an API and get results. They are the fastest way to add AI capabilities to an application.

### Amazon Rekognition

Computer vision service for image and video analysis.

- **Image analysis**: Object/scene detection, facial analysis (age, emotion, gender), face comparison and search, text detection (OCR on images), content moderation (unsafe content detection), celebrity recognition, custom labels (train your own image classifier with as few as 10 images)
- **Video analysis**: Same capabilities as image plus activity detection, person tracking, and segment detection (shot/technical cue detection)
- **Use cases**: Identity verification (face comparison), content moderation, automated media tagging, visual search, workplace safety (PPE detection)

Pricing: per image/video analyzed. Image analysis: $0.001 per image (first 1M), decreasing with volume. Video: per minute analyzed.

Gotchas: Facial recognition has accuracy variability across demographics — test thoroughly and understand the limitations before using for high-stakes decisions. Rekognition Custom Labels requires at least 10 labeled images per class but works best with hundreds.

### Amazon Textract

Document analysis and extraction service. Goes beyond basic OCR.

- **DetectDocumentText**: Basic OCR — extract text and layout from documents
- **AnalyzeDocument**: Extract structured data from forms (key-value pairs), tables, signatures, and queries (ask questions about the document)
- **AnalyzeExpense**: Specialized extraction from receipts and invoices (vendor, line items, totals)
- **AnalyzeLending**: Specialized extraction from mortgage and lending documents
- **AnalyzeID**: Extract data from identity documents (passports, driver's licenses)

Pricing: per page. DetectText: $0.0015/page. AnalyzeDocument with forms: $0.050/page. Queries: $0.015/page.

Gotchas: Textract works with scanned documents, PDFs, and images (JPEG, PNG, TIFF). Handwritten text support is available but less accurate than printed text. For complex multi-page documents, use asynchronous (batch) APIs — synchronous APIs have a one-page limit.

### Amazon Comprehend

Natural language processing (NLP) service.

- **Entity recognition**: Extract entities (people, places, organizations, dates, quantities) from text
- **Sentiment analysis**: Determine if text is positive, negative, neutral, or mixed
- **Key phrase extraction**: Identify important phrases
- **Language detection**: Identify the language of text (supports 100+ languages)
- **PII detection and redaction**: Find and optionally redact personally identifiable information
- **Topic modeling**: Discover topics across a collection of documents (batch only)
- **Custom classification**: Train a text classifier on your labeled data
- **Custom entity recognition**: Train a custom entity recognizer for domain-specific entities (drug names, product codes, etc.)

Pricing: per 100 characters (minimum 3 units = 300 characters per request). Entity/sentiment/key phrase: $0.0001 per unit. Custom models: training cost + per-unit inference cost.

Gotchas: Real-time analysis has a 100 KB per-document limit. For larger documents, split them. Custom classifiers need at least 50 examples per class, but 1,000+ is recommended for production quality. Topic modeling is batch-only and requires at least 1,000 documents.

### Amazon Comprehend Medical

Specialized NLP for medical text. Extracts medical entities (conditions, medications, dosages, procedures, anatomy), detects PHI (protected health information), and links entities to standard medical ontologies (ICD-10-CM, RxNorm, SNOMED CT).

Critical for healthcare/life sciences applications where standard Comprehend lacks medical domain knowledge. HIPAA-eligible service.

### Amazon Translate

Neural machine translation supporting 75+ languages. Supports real-time translation (TranslateText API) and batch translation (files in S3). Custom terminology lets you define domain-specific translations (brand names, technical terms) that override the default translation.

Active Custom Translation lets you adapt the translation model to your domain using parallel data (source-target sentence pairs) without ML expertise.

Pricing: $15 per million characters (real-time), $7.50 per million characters (batch). Free tier: 2M characters/month for 12 months.

### Amazon Transcribe

Speech-to-text service. Converts audio to text with punctuation, speaker identification (diarization), and custom vocabulary.

- **Real-time transcription**: Stream audio and receive text in real time (WebSocket API)
- **Batch transcription**: Process audio files from S3
- **Call Analytics**: Specialized for contact center calls — sentiment analysis, issue detection, call summarization
- **Medical**: HIPAA-eligible transcription with medical vocabulary

Supports custom vocabularies (domain-specific terms) and custom language models (for domain-specific speech patterns). Automatic language identification detects the spoken language from 37 supported languages.

Pricing: per second of audio. Standard: $0.024/min. Medical: $0.0375/min.

### Amazon Polly

Text-to-speech service. Converts text to lifelike speech in 30+ languages with dozens of voices.

- **Standard voices**: Concatenative synthesis — clear but slightly robotic
- **Neural voices**: Deep learning-based synthesis — more natural and expressive
- **Long-form voices**: Optimized for reading long content (articles, books)
- **SSML support**: Control pronunciation, emphasis, speed, pauses, and other speech characteristics

Pricing: per character. Standard: $4/million characters. Neural: $16/million characters.

### Amazon Kendra

Intelligent enterprise search powered by ML. Kendra understands natural language questions and returns specific answers (not just document links). It indexes documents from multiple sources (S3, SharePoint, Salesforce, databases, web pages, etc.) and uses ML to rank results by relevance.

Key difference from traditional search: Kendra understands the intent behind a question and can extract the specific answer from a document, not just return matching documents. For example, "What is the parental leave policy?" returns the specific policy paragraph, not just a link to the employee handbook.

Pricing: expensive. Kendra Developer Edition starts at $810/month. Enterprise Edition starts at $1,008/month plus per-document and per-query charges. This makes it a significant commitment.

### Amazon Q Business

The evolution of Kendra for GenAI-powered enterprise search and assistance. Q Business connects to your organization's data sources, indexes content, and provides a conversational AI assistant that answers questions grounded in your data.

Key differences from Kendra: Q Business uses generative AI to synthesize answers (not just extract snippets), supports multi-turn conversations, integrates with 40+ data sources natively, and provides a ready-made web application. Q Business is built on Bedrock under the hood.

Q Business also supports plugins (actions the assistant can take — create a Jira ticket, file a ServiceNow request) and guardrails (topic-level access controls — different users see different content based on their existing permissions in the source systems).

Pricing: per user per month. Q Business Lite: $3/user/month. Q Business Pro: $20/user/month.

---

## Comparison: When to Use What

### Decision Framework

**Pre-built AI Services** (Rekognition, Textract, Comprehend, Translate, Transcribe, Polly):
- Best for: Specific, well-defined AI tasks (OCR, translation, sentiment, speech)
- ML expertise needed: None
- Customization: Limited (custom vocabularies, custom classifiers)
- Cost: Per-API-call pricing, cheap for low volume
- Time to value: Minutes (just call an API)

**Amazon Bedrock**:
- Best for: GenAI applications (chatbots, summarization, content generation, reasoning, RAG)
- ML expertise needed: None for basic use, some for RAG optimization and fine-tuning
- Customization: RAG with your data, fine-tuning, prompt engineering
- Cost: Per-token pricing, varies widely by model
- Time to value: Hours to days

**Amazon SageMaker**:
- Best for: Custom ML models (recommendation engines, fraud detection, time series forecasting, custom computer vision)
- ML expertise needed: Significant (data science, ML engineering)
- Customization: Full control over data, algorithms, training, and hosting
- Cost: Per-instance-hour for training and hosting, can be expensive
- Time to value: Weeks to months

### Common Combinations

Many production systems use multiple tiers:

- **Document processing pipeline**: Textract (extract text) -> Comprehend (extract entities and PII) -> Bedrock (summarize and classify)
- **Customer service bot**: Transcribe (speech-to-text) -> Bedrock with Knowledge Bases (answer questions) -> Polly (text-to-speech)
- **Content platform**: Rekognition (moderate images) -> Translate (localize content) -> Bedrock (generate descriptions)
- **ML-powered product**: SageMaker (custom recommendation model) -> Bedrock (natural language explanations of recommendations) -> Comprehend (sentiment analysis on user feedback)

### Bedrock vs SageMaker JumpStart

Both let you use pre-trained foundation models. The key difference:

**Bedrock**: Fully managed inference. You do not see or manage the infrastructure. Model is served by AWS. Best for applications that just need to call a model API.

**SageMaker JumpStart**: You deploy the model to your own SageMaker endpoint. You manage the instance type, scaling, and infrastructure. Best when you need to customize the serving infrastructure, deploy models not available in Bedrock, or run models in your own VPC with full network control.

### When to Build Custom vs Use Pre-Built

Build custom (SageMaker) when:
- Your task is domain-specific and no pre-built service handles it well
- You have proprietary training data that gives competitive advantage
- You need full control over the model architecture and training process
- You need to run models on-premises or at the edge (SageMaker Neo, SageMaker Edge)

Use pre-built when:
- A standard AI capability (OCR, translation, sentiment) meets your needs
- You want to minimize operational overhead and ML expertise requirements
- Time to market is more important than model customization
- The pre-built service quality meets your accuracy requirements

### Common Interview Questions

**Q: How would you architect an AI-powered document processing system on AWS?**
A: For structured document extraction, use Textract to extract text, forms, and tables. For entity extraction and PII detection, pipe the text through Comprehend. For summarization or classification requiring reasoning, use Bedrock with a foundation model. Store extracted data in DynamoDB or Aurora. Use S3 for document storage and Step Functions to orchestrate the pipeline. For human review of low-confidence results, use Amazon Augmented AI (A2I).

**Q: What is the difference between Bedrock Knowledge Bases and Kendra?**
A: Kendra is a traditional enterprise search engine — it indexes documents and returns ranked results or extracted answers. Knowledge Bases is specifically designed for RAG — it retrieves relevant chunks and feeds them as context to a generative model. Kendra returns answers from existing text; Knowledge Bases + Bedrock generates new text grounded in your data. Q Business combines both: search + generative AI in a ready-made assistant.

**Q: How do you evaluate and compare AI models for a production use case?**
A: Define evaluation criteria specific to your use case (accuracy, latency, cost, safety). Create a representative test dataset with ground-truth answers. Use Bedrock Model Evaluation for automated metrics (ROUGE, BERTScore, human evaluation). Compare multiple models on the same dataset. Consider cost-performance tradeoffs — a cheaper model that is 95% as good may be the right choice. Monitor production metrics with Model Monitor (SageMaker) or custom CloudWatch metrics after deployment.

**Q: How do you handle AI costs at scale?**
A: Choose the cheapest model that meets quality requirements (Claude Haiku vs Opus is ~10x cost difference). Use batch inference for non-real-time workloads (50% discount). Cache frequent responses (ElastiCache or DynamoDB). Use Bedrock Provisioned Throughput for sustained workloads with predictable pricing. Implement request-level cost tracking with tags and CloudWatch. For SageMaker endpoints, use auto-scaling and serverless inference to avoid paying for idle compute.
