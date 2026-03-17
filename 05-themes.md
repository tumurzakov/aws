# Part 5: Cross-Cutting Themes

---

## Serverless Patterns and Anti-Patterns

### When to Go Serverless

Serverless (Lambda, DynamoDB, S3, API Gateway, Step Functions, EventBridge) is ideal when:

- Traffic is unpredictable or spiky — you do not want to pay for idle capacity
- You want to minimize operational overhead — no patching, no capacity planning, no server management
- Your workload is event-driven — respond to S3 uploads, DynamoDB changes, API requests, schedules
- You are building new applications and want to move fast without infrastructure concerns
- Your processing fits within Lambda's constraints (15-min timeout, 10 GB memory, stateless)

### When NOT to Go Serverless

- Sustained, predictable high throughput — EC2 or Fargate with reserved capacity is cheaper
- Ultra-low-latency requirements — Lambda cold starts add 100ms-10s of unpredictable latency
- Long-running processes — anything exceeding 15 minutes needs Fargate, ECS, or EC2
- Stateful workloads — WebSocket servers, game servers, or applications that need persistent connections
- GPU workloads — Lambda does not offer GPU instances
- Large monolithic applications — lifting and shifting a monolith to Lambda is not useful; containerize with Fargate instead

### Anti-Patterns

**Lambda monolith**: Putting an entire Express/FastAPI application inside a single Lambda function. This creates large deployment packages, long cold starts, and wastes the scaling granularity of Lambda. Instead, use one Lambda function per route or per logical group of routes.

**Lambda-to-Lambda (synchronous)**: One Lambda function directly invoking another and waiting for the response. This is an anti-pattern because: the calling Lambda pays for idle wait time, it creates tight coupling, it is difficult to debug, and timeouts cascade. Instead, use Step Functions for orchestration or SQS/EventBridge for asynchronous decoupling.

**Lambda-to-Lambda (asynchronous via direct invoke)**: Slightly better than synchronous, but still creates tight coupling and makes error handling difficult. Prefer SQS or EventBridge between functions.

**Orchestrating from Lambda**: Using a Lambda function to coordinate multiple downstream services (call A, then B, then C). This is fragile — any failure requires complex retry logic in your code. Use Step Functions instead, which provides built-in retry, catch, parallel execution, and visual monitoring.

**Lambda for everything**: Not every service needs to be a Lambda function. Batch processing might be better with Fargate or AWS Batch. Stream processing might be better with Kinesis + KCL on ECS. APIs with sustained traffic might be cheaper on Fargate.

### Service Selection Shortcuts

Use these when the architecture question is underspecified and you need to narrow the service set quickly:

- Need durable task buffering with simple worker consumption: SQS
- Need fan-out notifications to multiple consumers: SNS + SQS
- Need event routing based on event content or SaaS/AWS-native producers: EventBridge
- Need ordered high-throughput stream replay with multiple consumers: Kinesis
- Need simple request/response code with bursty traffic and short execution: Lambda
- Need long-running processes, custom runtimes, sidecars, or tighter OS/container control: ECS or EKS
- Need key-value access at scale with predictable low latency: DynamoDB
- Need relational joins, SQL transactions, and PostgreSQL compatibility: Aurora PostgreSQL
- Need workflow orchestration, retries, and compensating logic: Step Functions

If two services both work, prefer the simpler one operationally unless the simpler one violates a hard requirement like ordering, retention, latency, or protocol support.

---

## Event-Driven Architecture

### Core Concepts

Event-driven architecture (EDA) is a design pattern where components communicate through events rather than direct API calls. An event is a record of something that happened — "order placed," "payment received," "file uploaded." Producers emit events without knowing or caring who consumes them. This creates loose coupling, independent scalability, and system resilience.

### Event Sourcing

Instead of storing the current state of an entity, you store the sequence of events that led to the current state. The current state is derived by replaying events.

Benefits:
- Complete audit trail — you know exactly how the system arrived at its current state
- Time travel — reconstruct the state at any point in time
- Event replay — reprocess events when business logic changes
- Natural fit for event-driven systems

Challenges:
- Complexity — reading current state requires replaying events (mitigate with snapshots)
- Schema evolution — events are immutable, so handling schema changes requires careful versioning
- Storage — event logs grow indefinitely (mitigate with compaction or archival)

### CQRS (Command Query Responsibility Segregation)

CQRS separates the write model (commands) from the read model (queries). Commands update the write store (often an event log or normalized database), and a separate process projects the data into a read-optimized store (often a denormalized view or search index).

Benefits:
- Independently optimize read and write stores
- Read store can be denormalized for fast queries
- Multiple read models for different query patterns
- Scales reads and writes independently

CQRS is often paired with event sourcing: commands produce events, events update the read model through projections.

### Saga Pattern

A saga is a sequence of local transactions across multiple services, where each step has a compensating action that undoes its effect if a later step fails. Sagas replace distributed transactions (which are impractical in microservices) with eventual consistency.

Two styles:

**Choreography**: Each service listens for events and decides what to do next. No central coordinator. Services are loosely coupled but the overall flow is implicit and hard to trace.

**Orchestration**: A central coordinator (Step Functions is perfect for this) tells each service what to do and handles compensation on failure. The flow is explicit and visible but the coordinator is a potential bottleneck.

When to use which:
- Choreography: Simple sagas with 2-4 steps, services are independently owned by different teams
- Orchestration: Complex sagas with many steps, conditional logic, or error handling. Step Functions makes this straightforward.

### At-Least-Once and Idempotency

Almost all AWS messaging services (SQS standard, SNS, EventBridge, Kinesis, DynamoDB Streams) provide at-least-once delivery. This means your consumer may receive the same event more than once. You must design consumers to be idempotent — processing the same event multiple times should produce the same result as processing it once.

Idempotency strategies:
- **Idempotency key**: Store a unique event ID in DynamoDB (with a condition expression) before processing. If the ID already exists, skip processing.
- **Conditional writes**: Use DynamoDB condition expressions or database constraints to prevent duplicate state changes.
- **Naturally idempotent operations**: Some operations are inherently idempotent (set a value, delete a record). Prefer these over increment/append operations.
- **Lambda Powertools idempotency utility**: Automatically handles idempotency for Lambda functions using DynamoDB to store processed event hashes.

### Ordering Guarantees

- SQS Standard: No ordering guarantee
- SQS FIFO: Ordered within message group
- Kinesis: Ordered within shard (partition key)
- DynamoDB Streams: Ordered within partition key
- EventBridge: No ordering guarantee
- SNS: No ordering guarantee (FIFO topics preserve order to SQS FIFO subscribers)

If you need global ordering across all events, you are limited to a single SQS FIFO message group (300 TPS) or a single Kinesis shard (1 MB/s). Global ordering is extremely expensive at scale — design your system to only require ordering within a partition (customer ID, order ID, etc.).

### Event Contract Hygiene

Event-driven systems usually fail at the contract boundary, not at the transport layer. Minimum rules:

- Put immutable business facts in the event and avoid embedding transient transport concerns
- Include an event ID, event type, producer, timestamp, and schema version
- Prefer additive schema changes; treat field deletion or semantic changes as breaking changes
- Keep payloads small and store large blobs in S3 with a reference pointer
- Make consumers tolerant of unknown fields and duplicate deliveries
- Define whether the event is a command, notification, or state-change fact; mixing these semantics causes coupling

---

## Security and Compliance

### Shared Responsibility Model

AWS is responsible for security "of" the cloud — physical infrastructure, hypervisors, managed service internals. You are responsible for security "in" the cloud — IAM configuration, data encryption, network configuration, application security, patching (for EC2), and access management.

For managed services (Lambda, DynamoDB, S3), AWS handles more of the stack, but you are still responsible for IAM, data classification, encryption configuration, and network access controls.

### Encryption Everywhere

**Encryption at rest**: All data stores should use encryption at rest. Most AWS services support this natively with KMS: S3 (SSE-S3, SSE-KMS, SSE-C), DynamoDB (AWS owned or customer managed KMS key), Aurora (KMS-encrypted storage volumes), SQS (SSE-KMS or SSE-SQS), and Lambda (environment variables encrypted with KMS).

**Encryption in transit**: All communication should use TLS. AWS service endpoints support TLS by default. For VPC-internal traffic, use TLS between services. API Gateway enforces HTTPS by default. CloudFront supports custom SSL certificates.

### Least Privilege

Never grant more permissions than necessary. Use specific IAM actions, specific resource ARNs, and condition keys. Avoid `*` in actions and resources. Use IAM Access Analyzer to identify unused permissions and generate least-privilege policies. Review and rotate credentials regularly.

### SOC 2, HIPAA, GxP

- **SOC 2**: AWS services are SOC 2 compliant. Your responsibility: configure services securely, enable logging, enforce access controls.
- **HIPAA**: AWS offers a HIPAA-eligible services list. You must sign a Business Associate Agreement (BAA) with AWS. Your responsibility: encrypt PHI, restrict access, log all access, configure services per HIPAA requirements.
- **GxP** (pharmaceutical/biotech): AWS provides GxP guidance and a qualification workbook. Your responsibility: validate systems, maintain audit trails, control change management, ensure data integrity.

For regulated environments, focus on: encryption everywhere, comprehensive audit logging (CloudTrail + CloudWatch), least-privilege access, MFA enforcement, and automated compliance checking (AWS Config rules).

### Practical Security Baseline

For a new AWS workload, the minimum sane baseline is:

- Separate accounts for production and non-production using AWS Organizations
- SSO or federated access for humans; avoid long-lived IAM users
- MFA enforced for privileged access
- CloudTrail enabled organization-wide and delivered to a protected central account
- Default encryption at rest using KMS where customer-managed keys are required
- Secrets stored in Secrets Manager or Parameter Store, never in code or plain environment files
- Security groups restricted to explicit ports and peers; no broad inbound rules without justification
- Public internet exposure only through a deliberate edge layer such as CloudFront, ALB, or API Gateway
- Log retention configured explicitly for every CloudWatch log group

---

## High Availability and Disaster Recovery

### Availability Zones and Regions

**Availability Zone (AZ)**: One or more discrete data centers with independent power, cooling, and networking. AZs within a region are connected by low-latency (<2ms) fiber links.

**Region**: A geographic area with 3+ AZs. Regions are completely independent and isolated from each other.

For high availability: deploy across multiple AZs within a region. Most managed services (DynamoDB, S3, SQS, Lambda) do this automatically. For RDS/Aurora, use Multi-AZ deployments.

For disaster recovery: replicate across regions. This adds cost and complexity but protects against region-level failures.

### RPO and RTO

**RPO (Recovery Point Objective)**: How much data loss is acceptable. "We can afford to lose up to 1 hour of data."

**RTO (Recovery Time Objective)**: How quickly must the system be restored. "We need to be back online within 4 hours."

### DR Strategies (from cheapest/slowest to most expensive/fastest)

**Backup and Restore** (RPO: hours, RTO: hours):
- Regular backups to S3 (cross-region replication)
- Restore from backups in the DR region when needed
- Cheapest option but slowest recovery
- Suitable when RPO/RTO of hours is acceptable

**Pilot Light** (RPO: minutes, RTO: tens of minutes):
- Core infrastructure running in DR region (database replicas)
- Compute and other resources are pre-configured but stopped
- On failover, start the stopped resources and scale up
- Database replication keeps data near-current

**Warm Standby** (RPO: seconds, RTO: minutes):
- Scaled-down but fully functional copy of the production environment in the DR region
- Handles read traffic or a portion of production traffic
- On failover, scale up to full production capacity
- More expensive than pilot light but faster recovery

**Active-Active / Multi-Region Active** (RPO: ~0, RTO: ~0):
- Full production environments in multiple regions, all handling traffic simultaneously
- DNS-based routing (Route 53) distributes traffic across regions
- DynamoDB Global Tables, Aurora Global Database, S3 Cross-Region Replication for data
- Most expensive but near-zero RPO/RTO
- Complexity: conflict resolution for concurrent writes (last-writer-wins)

### DR Selection Heuristic

- Choose Backup and Restore if the business can tolerate hours of recovery and this is primarily a cost-sensitive workload
- Choose Pilot Light if data loss tolerance is low but traffic can tolerate recovery operations during failover
- Choose Warm Standby if recovery must be operationally boring and fast, but full active-active cost is not justified
- Choose Active-Active only when downtime is materially more expensive than permanent duplicated infrastructure and operational complexity

### Per-Service DR Capabilities

Understanding which services have built-in multi-region support vs which require manual patterns is critical for DR design.

**Services with built-in multi-region:**
- DynamoDB Global Tables — multi-active, last-writer-wins, <1s replication
- Aurora Global Database — single-writer primary, read replicas in secondary regions, <1s replication, managed failover
- DocumentDB Global Clusters — single-writer, read replicas in up to 5 secondary regions
- S3 Cross-Region Replication — async object replication, configurable per bucket/prefix
- ElastiCache Global Datastore — cross-region Redis replication, <1s lag, promote secondary on failover
- EventBridge Global Endpoints — automatic failover for event ingestion
- Route 53 — DNS-based failover routing with health checks
- CloudFront — global by design, serves from edge locations

**Regional services requiring manual DR patterns:**
- Lambda — deploy the same function in both regions, Route 53 routes to the healthy region's API Gateway
- SQS/SNS — regional, no cross-region replication. Use EventBridge cross-region rules or application-level publishing to both regions
- API Gateway — regional. Deploy in both regions, use Route 53 failover or CloudFront with origin failover
- Step Functions — regional, stateful. Long-running workflows in the failed region are lost. Design for restart/resume in the DR region
- Cognito — regional, no native replication. Users must re-authenticate in the DR region (tokens are region-specific). Can mitigate with a shared identity provider upstream of Cognito
- ECS/EKS — deploy in both regions, use Route 53 or Global Accelerator for traffic routing
- Secrets Manager — supports cross-region replication of secrets. Enable for DR
- KMS — regional keys. Use multi-region keys for seamless failover, or re-encrypt data with DR region keys

### Stateless vs Stateful DR

The key insight: stateless components (Lambda, ECS tasks, API Gateway) are easy to redeploy in a DR region — just deploy the same code/config. The hard part is always the data layer.

DR complexity by tier:
1. **Stateless compute**: Redeploy (minutes). Use IaC to deploy identical stacks in both regions.
2. **Configuration/secrets**: Replicate with Secrets Manager cross-region, SSM Parameter Store (manual or automation), or IaC.
3. **Databases**: The hardest part. Use Global Tables, Global Database, or CRR depending on the service. Test failover regularly.
4. **DNS cutover**: Route 53 failover routing. TTL determines how quickly clients discover the new region. Set TTL low (60s) for fast failover, but this increases DNS query costs.

### DR Interview Questions

**Q: Design a multi-region DR strategy for a serverless application using DynamoDB, Lambda, and API Gateway.**
A: Deploy the same CDK stack in two regions. Use DynamoDB Global Tables for data replication (active-active). Deploy Lambda functions and API Gateway in both regions. Use Route 53 failover routing with health checks on both API Gateway endpoints. Set DNS TTL to 60 seconds. If the primary region fails, Route 53 routes traffic to the secondary region. DynamoDB Global Tables keep data synchronized. RPO is near-zero (async replication <1s), RTO is 60-120 seconds (DNS propagation).

**Q: What is the most common mistake in multi-region DR?**
A: Not testing failover. Teams build the DR infrastructure but never simulate a failure. When a real outage occurs, they discover that: database replication is broken, IAM roles are missing in the DR region, secrets were not replicated, or the DNS failover takes longer than expected. Regular DR drills (at least quarterly) are essential. Automate the failover process and test it in a staging environment.

---

## Cost Optimization

### Right-Sizing

- Monitor actual usage (CloudWatch metrics) and compare to provisioned capacity
- For DynamoDB: switch to on-demand if utilization is consistently below 18% of provisioned capacity. Switch to provisioned if utilization is stable and predictable.
- For Lambda: use Power Tuning to find the optimal memory/cost balance
- For EC2/Fargate: use Compute Optimizer recommendations

### Storage Tiering

- S3: Use Intelligent-Tiering for unpredictable access patterns. Use lifecycle policies to transition to IA/Glacier for known access patterns.
- DynamoDB: Use TTL to automatically delete old data. Use DynamoDB export to S3 for archival.
- CloudWatch Logs: Set retention policies. Default "never expire" is a cost trap.

### Capacity Modes

- DynamoDB provisioned with auto-scaling is cheaper than on-demand for predictable workloads
- Aurora Serverless v2 avoids paying for idle capacity but costs more per ACU than provisioned instances at sustained utilization
- Lambda is cheaper than Fargate/EC2 for sporadic, event-driven workloads. Fargate/EC2 is cheaper for sustained, predictable compute.

### Cost Allocation Tags

Tag all resources with cost allocation tags (team, project, environment). Enable them in the Billing console. This enables per-project and per-team cost tracking in Cost Explorer. Without tags, you cannot attribute costs to specific workloads.

### Common Cost Traps

- NAT Gateway data processing fees ($0.045/GB) — use VPC endpoints for AWS service traffic
- CloudWatch Logs with no retention policy — grows forever
- Cross-AZ data transfer ($0.01/GB each way) — adds up for high-throughput services
- DynamoDB GSI over-provisioning or under-provisioning (throttling vs waste)
- Forgotten resources — dev/test environments left running, old snapshots, unused EIPs ($0.005/hr when unattached)
- S3 incomplete multipart uploads — create lifecycle policies to abort them
- Reserved capacity purchased but unused

### Cost Review Checklist

Before calling a design "cost-optimized," check these:

- Is traffic bursty enough that serverless beats always-on compute?
- Are NAT Gateways carrying AWS-to-AWS traffic that should use VPC endpoints instead?
- Are logs, snapshots, and S3 objects governed by retention and lifecycle rules?
- Are DynamoDB GSIs, Aurora replicas, or Fargate tasks sized for real usage instead of peak guesswork?
- Is cross-AZ or cross-region data transfer an intentional design choice rather than an accidental default?
- Are teams using tags that let finance attribute the spend to the owning service?

---

## Well-Architected Framework

The AWS Well-Architected Framework provides guidance for building secure, high-performing, resilient, and efficient cloud architectures. It has six pillars. Each pillar has design principles, key focus areas, and common anti-patterns.

### Well-Architected Review and Tool

A Well-Architected Review is a structured assessment of your workload against the six pillars. AWS provides the Well-Architected Tool in the console — you answer questions about your architecture, and the tool identifies high-risk issues with recommended improvements. Reviews should be done at major milestones (initial design, before production launch, annually).

### Lenses

Lenses are specialized extensions of the framework for specific domains:
- **Serverless Lens**: Lambda, API Gateway, DynamoDB, Step Functions best practices
- **SaaS Lens**: Multi-tenancy, tenant isolation, onboarding, billing
- **Machine Learning Lens**: Data preparation, model training, deployment, monitoring
- **Data Analytics Lens**: Data lakes, ETL, analytics workloads
- **Financial Services Lens**: Regulatory compliance, resilience requirements

Apply the general framework first, then the relevant lens for domain-specific guidance.

### 1. Operational Excellence

Focus: Run and monitor systems to deliver business value and continually improve processes.

Design principles:
- Perform operations as code (IaC with CDK/CloudFormation)
- Make frequent, small, reversible changes
- Refine operations procedures frequently
- Anticipate failure (chaos engineering, game days)
- Learn from all operational failures

Key questions:
- How do you determine what your priorities are? (Business objectives drive operational priorities)
- How do you structure your organization to support your business outcomes? (Team structure, ownership)
- How do you manage workload and operations events? (Runbooks, incident response, escalation)
- How do you evolve operations? (Feedback loops, continuous improvement)

Anti-patterns: Manual deployments, no runbooks, blame culture, changes require weeks of approvals, no post-incident reviews.

AWS services: CloudFormation/CDK, CloudWatch, X-Ray, Systems Manager (runbooks, automation), EventBridge (operational events), AWS Health Dashboard.

### 2. Security

Focus: Protect information, systems, and assets while delivering business value through risk assessments and mitigation.

Design principles:
- Implement a strong identity foundation (least privilege, no long-lived credentials)
- Enable traceability (log everything, audit everything)
- Apply security at all layers (network, compute, application, data)
- Automate security best practices (detect and remediate automatically)
- Protect data in transit and at rest
- Keep people away from data (automate data processing, minimize direct access)
- Prepare for security events (incident response plans, tested regularly)

Key questions:
- How do you manage identities for people and machines? (IAM Identity Center, IRSA, no shared accounts)
- How do you manage permissions? (Least privilege, ABAC, permission boundaries)
- How do you detect and investigate security events? (GuardDuty, Security Hub, CloudTrail)
- How do you protect your network resources? (VPC, security groups, WAF, Shield)
- How do you protect your compute resources? (Patching, vulnerability scanning, runtime protection)
- How do you classify your data? (Data classification, encryption policies, access controls)

Anti-patterns: IAM users with access keys, overly permissive `*/*` policies, no encryption, no logging, manual security reviews, shared credentials.

AWS services: IAM (Identity Center, Access Analyzer, ABAC), KMS, CloudTrail, GuardDuty, Security Hub, Config, WAF, Shield, Macie.

### 3. Reliability

Focus: Ensure a workload performs its intended function correctly and consistently when it is expected to.

Design principles:
- Automatically recover from failure (no manual intervention needed)
- Test recovery procedures regularly (game days, chaos engineering)
- Scale horizontally to increase aggregate workload availability
- Stop guessing capacity (auto-scaling, on-demand, serverless)
- Manage change in automation (IaC, CI/CD with rollback)

Key questions:
- How do you manage service quotas and constraints? (Monitor limits, request increases proactively)
- How do you plan your network topology? (Multi-AZ, redundant connectivity)
- How do you design your workload service architecture? (Loosely coupled, degradation over failure)
- How does your workload withstand component failures? (Retry, circuit breaker, fallback)
- How do you test reliability? (Failure injection, load testing, DR drills)
- How do you plan for disaster recovery? (RPO/RTO defined, strategy implemented, tested)

Anti-patterns: Single point of failure, single AZ deployment, no retries, no health checks, untested DR, manual scaling, no circuit breakers, monolithic architecture where one component failure takes down everything.

AWS services: Route 53 (health checks, failover), ELB (health checks), Auto Scaling, multi-AZ deployments, DynamoDB Global Tables, Aurora Global Database, S3 CRR, CloudWatch (alarms, monitoring).

### 4. Performance Efficiency

Focus: Use computing resources efficiently to meet system requirements and maintain that efficiency as demand changes and technologies evolve.

Design principles:
- Democratize advanced technologies (use managed services)
- Go global in minutes (multi-region, CloudFront)
- Use serverless architectures
- Experiment more often (easy to try different configurations)
- Consider mechanical sympathy (understand how services work internally)

Key questions:
- How do you select the best performing architecture? (Benchmarking, prototyping)
- How do you select your compute solution? (Lambda vs Fargate vs EC2, right-sizing)
- How do you select your storage solution? (EBS vs EFS vs S3, right type and class)
- How do you select your database solution? (Purpose-built databases, access pattern matching)
- How do you configure your networking solution? (VPC endpoints, placement groups, Global Accelerator)
- How do you monitor your resources to ensure they are performing? (CloudWatch, load testing)

Anti-patterns: Over-provisioned instances running at 10% utilization, wrong database type for the access pattern, no caching, synchronous calls where async would work, not using CDN for static content.

AWS services: Lambda, Fargate, Auto Scaling, CloudFront, ElastiCache/DAX, purpose-built databases, S3 Intelligent-Tiering.

### 5. Cost Optimization

Focus: Run systems to deliver business value at the lowest price point.

Design principles:
- Implement cloud financial management (dedicated team or function)
- Adopt a consumption model (pay for what you use)
- Measure overall efficiency (business value per dollar)
- Stop spending money on undifferentiated heavy lifting (managed services)
- Analyze and attribute expenditure (tags, Cost Explorer)

Key questions:
- How do you implement cloud financial management? (Budgets, alerts, regular reviews)
- How do you govern usage? (SCPs, quotas, tagging enforcement)
- How do you monitor usage and cost? (Cost Explorer, Cost Anomaly Detection, Budgets)
- How do you decommission resources? (Lifecycle policies, cleanup automation)
- How do you evaluate cost when you select services? (TCO analysis, not just unit price)

Anti-patterns: No tagging strategy, no log retention policies, NAT Gateway for AWS service traffic, paying on-demand for predictable workloads, unused reserved capacity, forgotten dev/test environments.

AWS services: Cost Explorer, AWS Budgets, Cost Anomaly Detection, Trusted Advisor, Compute Optimizer, S3 Intelligent-Tiering, Savings Plans.

### 6. Sustainability

Focus: Minimize environmental impact of running cloud workloads.

Design principles:
- Understand your impact (Carbon Footprint Tool)
- Establish sustainability goals
- Maximize utilization (right-size, scale down during low usage)
- Adopt more efficient hardware (Graviton processors — up to 60% less energy per computation)
- Use managed services (shared, optimized infrastructure)
- Reduce downstream impact (minimize data transfer, optimize client-side code)

### Pillar Trade-Offs

Pillars sometimes conflict:
- **Reliability vs Cost**: Multi-region active-active is the most reliable but most expensive. Choose the DR strategy that matches your actual RPO/RTO requirements, not the most impressive one.
- **Performance vs Cost**: Over-provisioning ensures performance but wastes money. Right-size with monitoring data, not guesswork.
- **Security vs Performance**: Encryption adds latency. VPC endpoints add cost. Guardrails add deployment friction. Accept these as the cost of security, but avoid security theater (controls that add friction without reducing risk).
- **Operational Excellence vs Speed**: Automation, testing, and runbooks take time to build but pay off long-term. Cutting corners on operations creates tech debt that compounds.

The right architecture is the one that makes the right trade-offs for your specific business context.

### Well-Architected Interview Questions

**Q: Walk me through how you would apply Well-Architected to a new serverless application.**
A: Start with Reliability: deploy across multiple AZs (Lambda and DynamoDB do this automatically), configure DLQs on all async flows, set up CloudWatch alarms. Security: use least-privilege IAM roles per function, encrypt all data (SSE-S3, DynamoDB encryption), enable CloudTrail and GuardDuty. Performance: right-size Lambda memory using Power Tuning, add caching (DAX or ElastiCache) for hot paths, use SQS to decouple components. Cost: use on-demand for DynamoDB during early growth, set CloudWatch Logs retention, use Lambda ARM64 (Graviton) for 20% cost savings. Operational Excellence: deploy with CDK, monitor with CloudWatch Logs Insights, create runbooks for common incidents. Review with the Serverless Lens for domain-specific guidance.

**Q: What are the six pillars and which is most important?**
A: Operational Excellence, Security, Reliability, Performance Efficiency, Cost Optimization, and Sustainability. None is universally most important — the priority depends on business context. A financial services application prioritizes Security and Reliability. A startup MVP prioritizes Cost Optimization and Performance. A regulated healthcare application prioritizes Security and Compliance (a Security concern). The framework is a decision aid, not a checklist — you make trade-offs based on your constraints.
