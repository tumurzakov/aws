# AWS Service and Theme Index

This reference covers AWS services and architectural themes at the general-knowledge level — how things work internally, limits, gotchas, and common interview questions. It is not tied to any specific project.

## How to Use This Set

- Start with `01-messaging.md` if the discussion is about decoupling, fan-out, queues, streams, or event buses
- Start with `02-compute-storage.md` if the design question is about runtime choice, persistence, orchestration, or IaC
- Start with `03-ai-ml.md` if the topic is GenAI, ML, foundation models, RAG, or document/image/speech AI
- Start with `04-supporting.md` if the problem is about security, networking, observability, delivery, identity, or access
- Use `05-themes.md` for architecture trade-offs and decision framing across services
- Use `06-questions.md` for interview prep and final review

## Reading Paths

### Path 1: Serverless Backend

`01-messaging.md` -> `02-compute-storage.md` -> `04-supporting.md` sections on IAM, VPC, API Gateway, CloudWatch -> `05-themes.md`

### Path 2: Platform / Cloud Architecture

`04-supporting.md` -> `02-compute-storage.md` sections on ECS/EKS, Aurora, CDK -> `05-themes.md` -> `06-questions.md`

### Path 3: AI / GenAI Focus

`03-ai-ml.md` -> `02-compute-storage.md` sections on Lambda, Step Functions, S3 -> `04-supporting.md` sections on IAM, VPC -> `06-questions.md`

### Path 4: Interview Refresh

`00-index.md` -> skim service comparisons in `01-messaging.md` and `04-supporting.md` -> `06-questions.md`

## Services Covered

- **SQS** — Fully managed message queue service with standard and FIFO variants
- **SNS** — Pub/sub notification service for fan-out messaging
- **EventBridge** — Serverless event bus for routing events between services using content-based rules
- **Kinesis** — Real-time streaming data ingestion and processing at scale
- **Lambda** — Serverless compute that runs code in response to events without managing servers
- **ECS (Elastic Container Service)** — Managed container orchestration service (Fargate or EC2)
- **EKS (Elastic Kubernetes Service)** — Managed Kubernetes service
- **DynamoDB** — Serverless NoSQL key-value and document database with single-digit millisecond latency
- **ElastiCache** — Managed Redis/Memcached for in-memory caching
- **DAX** — DynamoDB-specific in-memory cache with microsecond reads
- **MemoryDB** — Durable Redis-compatible database for primary data store use cases
- **S3** — Object storage with virtually unlimited capacity and strong consistency
- **Aurora PostgreSQL** — MySQL/PostgreSQL-compatible relational database with shared distributed storage
- **DocumentDB** — Managed document database with MongoDB wire protocol compatibility
- **Step Functions** — Serverless workflow orchestration for coordinating distributed services
- **CDK** — Infrastructure as code framework using familiar programming languages
- **Bedrock** — Managed service for foundation models (GenAI) with RAG, guardrails, and agents
- **SageMaker** — Full ML platform for building, training, and deploying custom models
- **Rekognition** — Computer vision for image and video analysis
- **Textract** — Document text and structure extraction (OCR+)
- **Comprehend** — Natural language processing (entities, sentiment, PII, classification)
- **Translate** — Neural machine translation for 75+ languages
- **Transcribe** — Speech-to-text with speaker diarization and custom vocabulary
- **Polly** — Text-to-speech with neural voices
- **Kendra** — Intelligent enterprise search powered by ML
- **Q Business** — GenAI-powered enterprise assistant grounded in your data
- **ELB (Elastic Load Balancing)** — ALB for Layer 7 and NLB for Layer 4 load balancing
- **Route 53** — Highly available DNS with health checks and various routing policies
- **IAM** — Identity and access management for authentication and authorization
- **Organizations** — Centralized management and billing for multiple AWS accounts
- **Resource Access Manager (RAM)** — Sharing AWS resources across accounts
- **Secrets Manager** — Secret storage with automatic rotation
- **Systems Manager Parameter Store** — Configuration and secret storage
- **CloudWatch** — Monitoring, logging, and alarming for AWS resources and applications
- **X-Ray** — Distributed tracing for debugging and analyzing microservice architectures
- **CloudTrail** — Audit logging for API calls across your AWS account
- **KMS** — Key management and envelope encryption for data protection
- **VPC** — Virtual private cloud networking, subnets, endpoints, Transit Gateway, and PrivateLink
- **API Gateway** — Managed API proxy for REST, HTTP, and WebSocket APIs
- **AppSync** — Managed GraphQL service for real-time applications
- **Cognito** — User authentication and federated identity management
- **CloudFront** — Global CDN for caching and delivering content at edge locations
- **Athena** — Serverless interactive SQL queries over data in S3
- **CodeBuild** — Fully managed build service for compiling, testing, and producing artifacts
- **CodeDeploy** — Automated deployments with blue/green, canary, and linear strategies
- **CodePipeline** — Continuous delivery orchestration across build, test, and deploy stages
- **CodeArtifact** — Managed artifact repository for npm, PyPI, Maven packages
- **ECR** — Managed Docker container registry with scanning and lifecycle policies
- **Managed Grafana** — Fully managed Grafana for multi-source dashboards
- **Managed Prometheus** — Fully managed Prometheus for container metrics

## Themes Covered

- Serverless patterns and anti-patterns
- Event-driven architecture (event sourcing, CQRS, sagas, idempotency)
- Security and compliance (shared responsibility, encryption, least privilege)
- High availability and disaster recovery (AZs, regions, RPO/RTO strategies, Multi-region)
- Cost optimization (right-sizing, tiering, capacity modes, cost allocation)
- Multi-account strategy (AWS Organizations, consolidated billing, VPC sharing)
- Well-Architected Framework (six pillars)

## File Layout

- `00-index.md` — This file
- `01-messaging.md` — SQS, SNS, EventBridge, Kinesis, Amazon MQ, Amazon MSK, and comparison
- `02-compute-storage.md` — Lambda, ECS/EKS, DynamoDB, S3, Aurora, DocumentDB, Step Functions, CDK
- `03-ai-ml.md` — Bedrock, SageMaker, Rekognition, Textract, Comprehend, Translate, Transcribe, Polly, Kendra, Q Business
- `04-supporting.md` — IAM, CloudWatch, X-Ray, CloudTrail, KMS, VPC, API Gateway, Cognito, CloudFront, Athena, Security Services, Observability, CI/CD
- `05-themes.md` — Cross-cutting architectural themes
- `06-questions.md` — Interview question bank

## Service-to-File Map

| Topic | Primary file |
| --- | --- |
| SQS, SNS, EventBridge, Kinesis, MQ, MSK | `01-messaging.md` |
| Lambda, ECS, EKS, DynamoDB, S3, Aurora, DocumentDB, Step Functions, CDK | `02-compute-storage.md` |
| Bedrock, SageMaker, Rekognition, Textract, Comprehend, Translate, Transcribe, Polly, Kendra, Q Business | `03-ai-ml.md` |
| IAM, Organizations, RAM, CloudWatch, X-Ray, CloudTrail, KMS | `04-supporting.md` |
| Secrets Manager, Parameter Store, VPC, ELB, Route 53 | `04-supporting.md` |
| API Gateway, AppSync, Cognito, CloudFront, Athena | `04-supporting.md` |
| WAF, Shield, GuardDuty, Security Hub, Config | `04-supporting.md` |
| CodeBuild, CodeDeploy, CodePipeline, CodeArtifact, ECR | `04-supporting.md` |
| Managed Grafana, Managed Prometheus, CloudWatch Synthetics | `04-supporting.md` |
| Serverless, EDA, security, HA/DR, cost, Well-Architected | `05-themes.md` |
| Interview drills and gotchas | `06-questions.md` |

## Coverage Notes

- The service docs go deeper on mechanics, quotas, and failure modes than on step-by-step setup
- The theme doc is where trade-offs live; use it when multiple AWS services could solve the same problem
- The question bank is intentionally answer-first and optimized for rapid repetition, not full explanations
