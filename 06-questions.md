# Part 6: Interview Question Bank

Rapid-fire questions and answers covering gotchas across all services. Organized by difficulty: Fundamentals, Intermediate, and Advanced.

---

## Fundamentals

**Q1: What consistency model does S3 use?**
A: Strong read-after-write consistency for all operations since December 2020. PUT, GET, LIST, and DELETE operations are all strongly consistent. The old eventual consistency model for overwrites and deletes is gone.

**Q2: What is the maximum size of an SQS message?**
A: 256 KB. For larger payloads, use the SQS Extended Client Library which stores the actual payload in S3 and sends a reference pointer (up to 2 GB).

**Q3: What is the default visibility timeout for SQS?**
A: 30 seconds. Maximum is 12 hours. Set it to at least 6x your average processing time.

**Q4: What is the difference between SQS and SNS?**
A: SQS is a queue (point-to-point, messages consumed and deleted). SNS is pub/sub (one-to-many fan-out, fire-and-forget). They are commonly used together: SNS fans out to multiple SQS queues.

**Q5: What is the maximum Lambda execution time?**
A: 15 minutes (900 seconds). For longer processes, use Step Functions, Fargate, or ECS.

**Q6: Does SNS persist messages?**
A: No. SNS is fire-and-forget. If a subscriber is unavailable, the message may be lost after retry exhaustion. Always pair SNS with SQS for durable delivery.

**Q7: What is the maximum item size in DynamoDB?**
A: 400 KB, including attribute names. Use short attribute names for large items and store large objects in S3.

**Q8: What is a cold start in Lambda?**
A: The initialization time when Lambda creates a new execution environment — downloading code, starting the runtime, and running initialization code. Adds 100ms-10s depending on language and package size. Eliminated by provisioned concurrency.

**Q9: What are the two types of VPC endpoints?**
A: Gateway endpoints (free, only for S3 and DynamoDB) and Interface endpoints (paid, for most other AWS services via PrivateLink).

**Q10: What is the maximum API Gateway integration timeout?**
A: 29 seconds. This is a hard limit. Use async patterns for longer operations.

---

## Intermediate

**Q11: How does DynamoDB handle hot partitions?**
A: Adaptive capacity redistributes throughput from underutilized partitions to hot ones (within 5-30 minutes). However, a single partition cannot exceed 3,000 RCU or 1,000 WCU. To truly fix hot partitions: use high-cardinality partition keys, implement write sharding, or add DAX for read caching.

**Q12: What happens when a Lambda function processing SQS messages fails?**
A: The message becomes visible again after the visibility timeout expires and is retried. After maxReceiveCount failures, it moves to the DLQ (if configured). With ReportBatchItemFailures enabled, only failed messages (not the entire batch) return to the queue.

**Q13: Explain the difference between EventBridge and SNS for pub/sub.**
A: EventBridge supports content-based filtering on any field in the event body, archive and replay, schema registry, SaaS integrations, and 200+ target types. SNS supports simple attribute-based filtering, delivers to email/SMS/push, and has higher raw throughput. EventBridge is better for complex event-driven architectures; SNS is better for simple notification fan-out.

**Q14: What is the difference between Standard and Express Step Functions?**
A: Standard: 1-year max duration, exactly-once, priced per state transition, 25K history event limit. Express: 5-minute max duration, at-least-once, priced per execution+duration, no history event limit. Use Standard for long-running auditable workflows; Express for high-volume short-duration processing.

**Q15: How does Aurora's storage work?**
A: Aurora uses a shared distributed storage layer across 3 AZs (6 copies of data). It uses quorum writes (4/6) and reads (3/6). Only redo logs are replicated (not full data pages), reducing I/O by 6x. Storage auto-grows in 10 GB increments up to 128 TB but never shrinks.

**Q16: What is envelope encryption in KMS?**
A: Generate a data key from KMS. Encrypt your data locally with the data key. Store the encrypted data key alongside the encrypted data. To decrypt, send the encrypted data key to KMS to get the plaintext data key, then decrypt locally. This avoids sending large data to KMS (which has a 4 KB Encrypt/Decrypt limit).

**Q17: What is the SQS FIFO deduplication window?**
A: 5 minutes. If two messages with the same deduplication ID are sent within 5 minutes, the second one is accepted but not delivered. After 5 minutes, a message with the same deduplication ID is treated as a new message.

**Q18: How do you handle the 256 KB payload limit in Step Functions?**
A: Do not pass large data through the state machine. Store large payloads in S3 and pass only the S3 reference (bucket + key) between states. Use the ResultSelector and ResultPath to keep payloads minimal.

**Q19: What is GSI back-pressure throttling in DynamoDB?**
A: When a GSI does not have enough write capacity to keep up with base table writes, DynamoDB throttles writes on the base table. The throttling appears on the base table, not the GSI, which makes it confusing to diagnose. Always provision GSI write capacity to match or exceed the base table's write throughput.

**Q20: What are CloudWatch Logs retention defaults and why does it matter?**
A: Default retention is "never expire." Logs accumulate forever with increasing storage costs. This is one of the most common cost traps in AWS. Always set explicit retention policies on every log group.

**Q21: How does Kinesis differ from SQS for streaming?**
A: Kinesis is a log (records retained, multiple consumers, ordered per shard, capacity managed by shards). SQS is a queue (messages consumed and deleted, no ordering guarantee in standard, virtually unlimited throughput). Use Kinesis for multiple consumers on ordered streams; use SQS for task processing where each message is done once.

**Q22: What is the maximum number of DynamoDB Streams consumers?**
A: Recommended maximum is 2 Lambda functions per stream. You can have more using KCL, but exceeding 2 consumers increases the risk of read throttling on the stream.

**Q23: Can you guarantee exactly-once processing in SQS?**
A: FIFO queues provide exactly-once delivery (within the 5-minute deduplication window). But true exactly-once processing also requires your consumer to be idempotent, because the delete-after-processing step can fail, leading to reprocessing.

**Q24: What happens to presigned S3 URLs when the signing credentials are revoked?**
A: Presigned URLs created with long-lived IAM user credentials remain valid until their expiration time, even after the credentials are revoked. Presigned URLs created with temporary credentials (STS/roles) expire when the temporary credentials expire, regardless of the URL's expiration setting.

**Q25: What are the data transfer costs for cross-AZ traffic?**
A: $0.01/GB in each direction ($0.02/GB round trip). This adds up for high-throughput services like RDS read replicas or Kinesis consumers in a different AZ.

---

## Advanced

**Q26: What happens when a DynamoDB partition splits?**
A: When a partition exceeds 10 GB or its throughput limits, DynamoDB splits it into two partitions. The partition key space is divided, and data is redistributed. Splits are permanent — partitions are never merged back. During the split, there may be brief periods of increased latency, but the table remains available. Adaptive capacity continues to work across the new partitions.

**Q27: Explain Lambda's recursive loop detection.**
A: Lambda detects circular invocation patterns (Lambda -> SQS -> Lambda or Lambda -> SNS -> Lambda) after approximately 16 iterations. It stops the loop, sends a notification via AWS Health Dashboard, and temporarily blocks the recursive pattern. This only detects SQS and SNS loops — other recursive patterns (Lambda -> EventBridge -> Lambda) are not detected.

**Q28: How does DynamoDB handle transactions internally?**
A: DynamoDB transactions use a two-phase commit: (1) Prepare phase — lock all items involved, validate conditions. (2) Commit phase — apply all changes atomically. This is why transactions cost 2x — each item is read twice (once for prepare, once for commit). If two transactions conflict on the same item, one will fail with TransactionConflictException.

**Q29: What is the difference between Lambda Destinations and DLQ?**
A: Destinations receive the full invocation result (input + output or error) and support 4 target types (SQS, SNS, EventBridge, Lambda) for both success and failure. DLQs receive only the failed event payload and support only SQS and SNS, only for failures. Destinations are newer and more flexible. Important: Destinations work only for async invocations, not for event source mapping failures (those use the event source mapping's on-failure destination, which is a different configuration).

**Q30: How does EventBridge archive replay interact with current rules?**
A: Replayed events are sent to the event bus and evaluated against current rules, not the rules that existed when the event was originally published. If you have added new rules since the event was archived, replayed events will be delivered to the new targets. If you have deleted rules, those targets will not receive the replayed events. This can be useful (backfill a new consumer) or dangerous (unexpected side effects from rule changes).

**Q31: What is the DynamoDB on-demand "ramp-up" behavior?**
A: On-demand tables can instantly handle up to 2x the previous peak throughput. Beyond that, DynamoDB takes up to 30 minutes to fully scale. A completely cold table (no recent traffic) can handle approximately 4,000 WCU and 12,000 RCU initially. If you expect a sudden traffic spike on a cold on-demand table, pre-warm it by temporarily switching to provisioned mode with your expected capacity, then switching back to on-demand.

**Q32: How does S3 handle the prefix throughput limit internally?**
A: S3 automatically partitions data by key prefix. Each partition supports 3,500 writes/s and 5,500 reads/s. When S3 detects a hot prefix, it automatically repartitions to distribute load. However, this repartitioning is reactive and takes time — sudden traffic spikes on a new prefix can cause 503 Slow Down errors before repartitioning completes. Historical advice about using random prefixes is less critical now due to automatic repartitioning, but it still helps for extreme throughput.

**Q33: How does Aurora's failover work mechanically?**
A: When the primary fails: (1) Aurora detects the failure via heartbeat (seconds). (2) The cluster endpoint DNS is updated to point to the promoted replica (typically the replica with the highest priority). (3) The promoted replica begins accepting writes. Total time is approximately 30 seconds. Applications using the cluster endpoint reconnect automatically, but they must handle the brief connection drop. Read replicas continue serving reads from the shared storage during failover.

**Q34: What is the Kinesis "shard blocking" problem with Lambda?**
A: When Lambda reads from Kinesis and the function fails on a batch, Lambda retries the same batch from the same position. Because Kinesis is ordered per shard, Lambda cannot skip ahead — it blocks the entire shard until the batch succeeds or the data expires. This means one bad record can block processing for all records in that shard. Mitigations: set maximum retry age and maximum retry attempts, enable bisect-on-error (Lambda splits the failing batch in half to isolate the bad record), configure an on-failure destination, and handle errors within the function itself.

**Q35: How does DynamoDB's conditional write achieve optimistic locking?**
A: Read the item and note a version attribute. Update the item with a condition expression: `attribute_exists(pk) AND version = :expected_version`. If another writer modified the item between your read and write, the condition fails and the update is rejected with ConditionalCheckFailedException. Your application retries with a fresh read. This is cheaper and more scalable than pessimistic locking because no locks are held during processing.

**Q36: What is the difference between KMS symmetric and asymmetric key operations?**
A: For symmetric keys, plaintext key material never leaves KMS — all Encrypt, Decrypt, and GenerateDataKey operations happen within the KMS HSM. For asymmetric keys, the public key can be downloaded and used outside AWS (e.g., for client-side encryption or signature verification). Private key operations (Decrypt, Sign) still happen within KMS. Symmetric keys are used for 99% of AWS service integrations.

**Q37: How does SQS FIFO high-throughput mode work?**
A: High-throughput FIFO mode increases throughput from 300 to up to 70,000 messages per second per queue by allowing SQS to process messages across multiple internal partitions. The trade-off: within a 5-minute window, messages with the same deduplication ID may be delivered more than once (at-least-once instead of exactly-once). Ordering within a message group is still guaranteed.

**Q38: What happens when CloudFormation detects a construct ID change in CDK?**
A: CDK generates CloudFormation logical IDs from construct IDs. When the construct ID changes, the logical ID changes. CloudFormation sees a new resource (with the new logical ID) and a missing resource (the old logical ID). It creates the new resource and deletes the old one. For stateful resources (databases, S3 buckets), this means data loss. The only protection is setting RemovalPolicy.RETAIN, which prevents the old resource from being deleted (you then manually import it or update references).

**Q39: How do EventBridge Pipes differ from EventBridge event bus rules?**
A: Pipes are point-to-point integrations (one source to one target) with built-in filtering, enrichment, and transformation. They read from poll-based sources (SQS, Kinesis, DynamoDB Streams, Kafka). Event bus rules are many-to-many: events from any source are evaluated against all rules and delivered to matching targets. Use Pipes when you have a single source that needs to be connected to a single target with optional processing. Use event bus rules when you need routing, fan-out, or content-based delivery to multiple targets.

**Q40: What is the difference between S3 Object Lock governance and compliance modes?**
A: Governance mode allows users with the s3:BypassGovernanceRetention permission to delete or overwrite locked objects. Compliance mode cannot be overridden by anyone, including the root account — the object cannot be deleted until the retention period expires. Compliance mode is for regulatory requirements (SEC Rule 17a-4, HIPAA). Choose compliance only when you are certain about the retention period, because it cannot be shortened.

**Q41: How does Lambda SnapStart work?**
A: SnapStart (Java only currently) takes a Firecracker snapshot of the initialized execution environment after the init phase completes. On cold start, instead of re-running the full init, Lambda restores from the snapshot. This reduces Java cold starts from 5-10s to under 200ms. Gotcha: any uniqueness assumptions made during init (random seeds, unique IDs, network connections) must be refreshed at restore time. Use the `beforeCheckpoint` and `afterRestore` hooks to handle this.

**Q42: How do you prevent runaway costs from DynamoDB scans?**
A: First, avoid scans in production — use Queries with partition key conditions. If you must scan: set a Limit parameter (number of items to evaluate per call), use FilterExpression to reduce result size (but note that the scan still reads and charges for all items), use Parallel Scan with multiple segments to finish faster, and schedule scans during off-peak hours. For analytics on DynamoDB data, export to S3 and query with Athena instead.

**Q43: What is the "sticky consumer" problem with Kinesis?**
A: KCL assigns shards to workers via leases. If a worker fails but its leases are not promptly released, the shard is stuck until the lease expires (default 10 seconds in KCL 2.x). During this time, no other worker processes the shard. Additionally, when shards are redistributed after scaling or failure, some workers may temporarily process more shards than others, causing uneven load. The KCL handles this automatically but there is a convergence delay.

**Q44: How does DynamoDB Global Tables handle conflicts?**
A: Last-writer-wins based on item-level timestamps. When the same item is written concurrently in two regions, the write with the later timestamp wins. There is no conflict detection or resolution mechanism — the "losing" write is silently overwritten. This means your application must be designed to tolerate lost updates or you must implement application-level conflict detection (e.g., version vectors).

**Q45: What is the Aurora storage "never shrinks" problem?**
A: Aurora storage grows automatically as you add data, but the allocated volume never shrinks when you delete data. Deleted space is reused internally by future inserts, but the billed storage remains at the high-water mark. The only way to reclaim storage is to create a new cluster from a snapshot (logical dump/restore). This is important for workloads with large temporary data sets.

**Q46: How does S3 cross-region replication handle deletes?**
A: By default, CRR does not replicate delete operations. If you delete an object in the source bucket, it remains in the destination bucket. You can enable delete marker replication (for versioned buckets), which replicates delete markers but not permanent version deletes. This is intentional — it prevents accidental or malicious mass deletions from propagating to the backup region.

**Q47: What are the limits of DynamoDB transactions?**
A: Maximum 100 items per transaction, maximum 4 MB total size. Transactions cost 2x the normal RCU/WCU. Two transactions writing to the same item will conflict — one will fail. Transactions cannot span multiple accounts or regions. All items in a transaction must be in the same region. Transactions time out after 10 seconds.

**Q48: How does IAM policy evaluation differ for cross-account access via resource-based policies vs IAM roles?**
A: With resource-based policies (e.g., S3 bucket policy granting access to Account B): the effective permissions are the union of the identity policy and the resource policy. Account B's principal can access the resource even without an identity policy allowing it, as long as the resource policy grants access. With IAM roles (AssumeRole): the effective permissions are the intersection of the role's policy and any session policy. The calling account needs permission to call sts:AssumeRole, and the role's trust policy must allow the caller. The key difference: resource-based policies can grant access unilaterally.

**Q49: What is the "Lambda inside VPC" networking trap?**
A: Lambda in a VPC loses internet access. It cannot call AWS APIs, external APIs, or download packages without a NAT Gateway or VPC endpoints. Common failure: Lambda tries to call DynamoDB or S3 but times out because there is no route. Solution: use gateway endpoints (free) for S3 and DynamoDB, interface endpoints (paid) for other services, or a NAT Gateway (expensive at $0.045/hr + $0.045/GB) for internet access. Many teams do not realize this until their first VPC-attached Lambda deployment fails.

**Q50: How does EventBridge achieve at-least-once delivery and what are the implications?**
A: EventBridge stores events durably and delivers them to targets with retries (up to 24 hours, 185 attempts). The at-least-once guarantee means a target may receive the same event more than once (especially during retries after transient failures). Combined with the lack of ordering guarantees, this means every EventBridge consumer must be idempotent. For Lambda targets, use the built-in idempotency features (PowerTools) or implement your own using DynamoDB conditional writes with the event ID.

**Q51: When would you choose ECS over EKS?**
A: Choose ECS when you want AWS-native container orchestration with lower operational overhead, simpler IAM integration, and no need for raw Kubernetes APIs. Choose EKS when you specifically need Kubernetes portability, CRDs/operators, Helm-based platform patterns, or an existing Kubernetes skill set. If the requirement is just "run containers on AWS," ECS is usually the simpler default.

**Q52: What is the practical difference between Secrets Manager and Parameter Store?**
A: Secrets Manager is optimized for secrets lifecycle management: native rotation workflows, version staging labels, and tighter secret-specific integrations. Parameter Store is better for hierarchical configuration and lower-cost simple secret storage, but advanced parameters and rotation patterns are less ergonomic. Use Secrets Manager for database credentials, API tokens, and rotating secrets; use Parameter Store for configuration trees and non-rotating values.

**Q53: Why do many teams put CloudFront in front of S3 or API Gateway even when the origin already works?**
A: CloudFront reduces latency through edge caching, absorbs traffic spikes, offloads TLS and compression, supports WAF integration, and gives better control over CDN behavior like signed URLs/cookies and cache policies. The origin may work without it, but CloudFront often improves both performance and protection at the edge.

**Q54: What is the key operational difference between ALB and NLB?**
A: ALB is Layer 7: it understands HTTP/HTTPS, supports host/path routing, WebSockets, redirects, and header-based behavior. NLB is Layer 4: it forwards TCP/UDP/TLS with very high throughput and low latency, preserves the client IP, and is better for non-HTTP protocols or extreme connection scale. If you need HTTP-aware routing, use ALB. If you need raw transport performance, static IPs, or non-HTTP traffic, use NLB.

**Q55: What is the main trade-off between API Gateway and AppSync?**
A: API Gateway is the better default for REST-style APIs, simple proxying, and broad protocol support. AppSync is purpose-built for GraphQL, real-time subscriptions, client-driven field selection, and resolver-based aggregation across data sources. AppSync reduces backend over-fetching for GraphQL clients, but it adds GraphQL-specific operational and schema design complexity.

**Q56: How does Cognito User Pools differ from Identity Pools?**
A: User Pools handle user authentication: sign-up, sign-in, tokens, password policies, MFA, and federation. Identity Pools handle AWS credential vending: they exchange identities from User Pools or external IdPs for temporary AWS credentials so clients can call AWS services directly. Authentication and authorization to AWS resources are separate concerns, and Cognito splits them that way.

**Q57: What is the biggest Route 53 misconception in HA design?**
A: Many teams think Route 53 health checks and failover routing give instantaneous failover. They do not. DNS caching, TTLs, resolver behavior, and client retry logic all affect failover time. Route 53 is powerful, but it is not a connection-level failover mechanism; applications still need retry logic and tolerance for stale DNS answers.

**Q58: Why is Athena often the right answer for one-off analysis but the wrong answer for a hot production path?**
A: Athena is excellent for ad hoc SQL over S3 with no infrastructure to manage, especially for logs, exports, and periodic reporting. It is usually a poor fit for latency-sensitive transactional workloads because query startup time, scan-based pricing, and file layout sensitivity make it unpredictable and expensive for frequent small queries. It is an analytics tool, not an OLTP database.

**Q59: What is the most common multi-account mistake with AWS Organizations and RAM?**
A: Teams centralize networking or shared services but forget the ownership model. With RAM and VPC sharing, the network account owns the VPC constructs while participant accounts own the resources placed into shared subnets. That split affects permissions, troubleshooting, quotas, and who can modify what. If you do not make that responsibility boundary explicit, operations become confusing fast.

**Q60: What is the first thing to check when Bedrock output quality is poor in a RAG workflow?**
A: Check retrieval quality before blaming the model. Weak chunking, low-signal embeddings, poor metadata filtering, or retrieving the wrong documents usually hurt output more than model choice. In most RAG failures, the prompt and model are downstream of bad context selection.

**Q61: Is DocumentDB actually MongoDB?**
A: No. DocumentDB implements the MongoDB wire protocol (API compatibility) but uses a completely different storage engine — the Aurora-style distributed storage with 6-way replication across 3 AZs. Most MongoDB applications work unchanged, but features relying on MongoDB internals (oplog tailing, server-side JavaScript with $where/$function, map-reduce, capped collections) do not work. Always test before migrating.

**Q62: When would you choose DocumentDB over DynamoDB?**
A: Choose DocumentDB when your data is naturally document-shaped with nested structures, when you need ad-hoc queries and complex aggregation pipelines, when you are migrating from MongoDB, or when query patterns are not fully known at design time. Choose DynamoDB for single-digit millisecond latency at any scale, well-defined access patterns modeled with partition/sort keys, serverless on-demand scaling, or multi-region active-active replication.

**Q63: How does DocumentDB handle change data capture?**
A: DocumentDB supports change streams that provide a real-time stream of document-level changes (insert, update, replace, delete). You can configure DocumentDB as a Lambda event source — Lambda polls the change stream and invokes your function with batches of change events. Change stream events are retained for up to 7 days. If your consumer falls behind beyond the retention window, it loses its position and must restart from the current point. EventBridge Pipes can also read from DocumentDB change streams.

**Q64: What is the biggest operational surprise when migrating from MongoDB to DocumentDB?**
A: Two things catch most teams. First, storage never shrinks — same as Aurora. If you bulk-load data and delete it, you still pay for the high-water mark. Second, not all MongoDB features work: no server-side JavaScript ($where, $function, $accumulator, map-reduce), no capped collections, no hashed indexes in standard clusters, and the change stream implementation uses polling with resume tokens rather than MongoDB's oplog. Aggregation pipeline differences can also cause subtle query result variations.
