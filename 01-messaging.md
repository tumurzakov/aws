# Part 1: Messaging and Event Services

This is the most critical section. Messaging and eventing are the connective tissue of any cloud-native architecture, and interview questions about queues, topics, and event buses are common and detailed.

---

## Amazon SQS (Simple Queue Service)

### Overview

SQS is a fully managed message queue that decouples producers from consumers. It guarantees at-least-once delivery (standard queues) or exactly-once processing (FIFO queues). Messages are stored redundantly across multiple AZs. There is no server to provision — you create a queue and start sending messages immediately.

### How It Works Internally

When a producer sends a message to SQS, the service stores the message redundantly across multiple servers in the region. When a consumer polls the queue, SQS delivers the message and starts a visibility timeout clock. During this window, the message is invisible to other consumers. The consumer must delete the message before the visibility timeout expires; otherwise, the message becomes visible again and another consumer can pick it up.

The queue itself is a distributed system. Standard queues use a best-effort ordering model where messages are stored across many partitions for high throughput, which means ordering is not guaranteed. FIFO queues use a single ordered log per message group ID, which guarantees order but limits throughput.

SQS does not push messages to consumers. Consumers must poll. There are two polling modes: short polling and long polling. Short polling queries a subset of SQS servers and returns immediately, even if the response is empty. Long polling queries all servers and waits up to 20 seconds for a message to arrive before returning. Long polling reduces empty responses, reduces cost (fewer API calls), and is almost always what you want. You enable it by setting ReceiveMessageWaitTimeSeconds to a value greater than 0 (up to 20).

### Standard vs FIFO Queues

Standard queues offer virtually unlimited throughput. Messages are delivered at least once and may occasionally be delivered more than once. Ordering is best-effort — messages generally arrive in the order they were sent, but this is not guaranteed. Use standard queues when your consumer is idempotent and you need maximum throughput.

FIFO queues guarantee that messages are processed exactly once and in the exact order they were sent within a message group. The message group ID is a tag you attach to each message; messages within the same group are strictly ordered, while messages across different groups can be processed in parallel. This is the key scaling mechanism for FIFO — you scale by increasing the number of distinct message group IDs.

FIFO throughput limits: 300 messages per second (300 send, receive, or delete operations per second) without batching. With batching (up to 10 messages per batch), you get 3,000 messages per second. With the high-throughput FIFO mode, you can reach up to 70,000 messages per second per partition, but this relaxes the exactly-once guarantee to at-least-once within each 5-minute deduplication window.

### Visibility Timeout

The visibility timeout is the period during which a message, once received by a consumer, is hidden from other consumers. The default is 30 seconds. The maximum is 12 hours. If your consumer needs more time to process a message, you can call ChangeMessageVisibility to extend the timeout before it expires.

A common mistake is setting the visibility timeout too short. If the consumer takes longer than the visibility timeout to process and delete the message, the message reappears in the queue and gets processed again by another consumer — leading to duplicate processing. The rule of thumb: set the visibility timeout to at least 6x your average processing time.

When Lambda processes SQS messages, Lambda automatically manages the visibility timeout. It sets the visibility timeout of the queue to 6x the Lambda function timeout. If you change the Lambda timeout, make sure the queue visibility timeout is updated as well.

### Message Lifecycle

1. Producer sends a message (SendMessage or SendMessageBatch)
2. SQS stores the message redundantly across AZs
3. Consumer polls (ReceiveMessage) — SQS delivers the message and starts the visibility timeout
4. Consumer processes the message
5. Consumer deletes the message (DeleteMessage) — this is the acknowledgment
6. If the consumer fails to delete before visibility timeout expires, the message becomes visible again
7. After a configured number of receive attempts (maxReceiveCount), the message is moved to the dead-letter queue if configured

Messages have a maximum retention period of 14 days (default 4 days). After this period, unprocessed messages are automatically deleted.

### Dead-Letter Queues (DLQ)

A DLQ is just another SQS queue that receives messages that could not be processed successfully. You configure a redrive policy on the source queue specifying the DLQ ARN and a maxReceiveCount. When a message has been received more than maxReceiveCount times without being deleted, it is moved to the DLQ.

Important: The DLQ must be the same type as the source queue. A FIFO queue's DLQ must also be a FIFO queue. A standard queue's DLQ must be a standard queue.

The DLQ does not preserve the original receive count or timestamps — the message gets a new MessageId in the DLQ. However, the original message attributes are preserved.

SQS now supports DLQ redrive, which allows you to move messages from the DLQ back to the source queue (or another queue) for reprocessing. This is available in the console and via the StartMessageMoveTask API. You can set a maximum rate for the redrive to avoid overwhelming downstream systems.

### Deduplication

FIFO queues have built-in deduplication with a 5-minute deduplication window. There are two deduplication methods: content-based deduplication (SQS generates a hash of the message body) or explicit deduplication IDs (you provide a MessageDeduplicationId). If two messages with the same deduplication ID arrive within the 5-minute window, the second one is accepted but not delivered.

Standard queues have no built-in deduplication. You must handle deduplication at the consumer level (idempotency).

### Batch Processing with Lambda

When you configure an SQS queue as a Lambda event source, Lambda polls the queue on your behalf and invokes your function with a batch of messages. Key settings:

- Batch size: 1 to 10,000 messages (but Lambda payload is max 6MB, so practical limits apply)
- Batch window: how long to wait to fill a batch (0 to 300 seconds)
- Maximum concurrency: limits how many Lambda instances process the queue simultaneously (2 to 1,000)
- Report batch item failures: allows your function to return partial successes — only failed messages return to the queue, not the entire batch. Without this, any failure causes the entire batch to return to the queue.

For FIFO queues with Lambda, Lambda respects message group ordering. It processes one batch per message group at a time. If you have 100 message groups, Lambda can process up to 100 batches concurrently.

### Delay Queues

A delay queue postpones delivery of new messages for a configurable period (0 to 15 minutes). When a message is sent to a delay queue, it is invisible to consumers for the delay duration. After the delay, it becomes available for polling.

You can set delay at two levels: per-queue (DelaySeconds attribute on the queue) or per-message (DelaySeconds parameter on SendMessage). Per-message delay overrides the queue-level delay for standard queues. For FIFO queues, per-message delay is not supported — only queue-level delay works.

Use cases for delay queues: rate-limiting downstream processing (introduce a buffer), scheduling retry attempts with increasing delays, allowing time for dependent resources to become available before processing.

### Encryption

SQS supports server-side encryption with two options:

**SSE-SQS** (SQS-managed encryption): SQS manages the encryption keys. No additional cost. No KMS API calls. No KMS rate limits to worry about. This is sufficient for most use cases where you need encryption at rest for compliance but do not need customer-managed key control.

**SSE-KMS** (KMS-managed encryption): You provide a KMS key (AWS managed or customer managed). Every SendMessage/ReceiveMessage/DeleteMessage triggers KMS Encrypt/Decrypt calls under the hood. This means: additional cost per KMS API call, subject to KMS rate limits (5,500-30,000 per key per second), and you can use key policies and grants for fine-grained access control. Use SSE-KMS when you need to control who can encrypt/decrypt messages via KMS key policy, need key rotation control, or need audit trail of key usage via CloudTrail.

Important gotcha: If your SQS queue uses SSE-KMS and an SNS topic or S3 bucket publishes to it, the KMS key policy must grant the source service (sns.amazonaws.com or s3.amazonaws.com) permission to use the key. Without this, messages are silently dropped.

### Message Attributes vs Message Body

The message body is the main content of the message (up to 256 KB). Message attributes are structured key-value metadata attached alongside the body. You can have up to 10 message attributes per message. Attributes have a name, type (String, Number, or Binary), and value.

Message attributes are important for two reasons. First, SNS message filtering operates on message attributes (not the body), so if your SQS queue is subscribed to an SNS topic with filters, the attributes determine routing. Second, message attributes allow consumers to inspect metadata without parsing the body — useful for routing, prioritization, or content-type detection.

Message system attributes are a separate set of attributes managed by AWS (e.g., AWSTraceHeader for X-Ray). You cannot set these directly.

### Access Policies

SQS queues have resource-based policies (queue policies) that control who can send to or receive from the queue. This is critical for cross-service and cross-account access.

Common patterns: Allow an SNS topic in another account to send messages to your queue. Allow S3 in another account to send event notifications to your queue. Allow a Lambda function in another account to poll your queue. All of these require the queue policy to grant the source principal permission.

Without a proper queue policy, cross-service integrations fail silently — the source service gets a permission denied and the message is lost (for SNS) or the event notification is never delivered (for S3).

### Monitoring: ApproximateNumberOfMessages Metrics

SQS publishes three key CloudWatch metrics for queue depth:

- **ApproximateNumberOfMessagesVisible**: Messages available for processing. Use this for autoscaling consumers.
- **ApproximateNumberOfMessagesNotVisible**: Messages currently being processed (in-flight, within visibility timeout).
- **ApproximateNumberOfMessagesDelayed**: Messages in delay period, not yet visible.

The word "Approximate" is important. These metrics are eventually consistent and may not reflect the exact current state, especially during high-throughput bursts. Do not use them for exact counting. They are reliable enough for autoscaling decisions and operational monitoring but should not be used for business logic that requires precision.

For autoscaling, the common formula is: `backlog per instance = ApproximateNumberOfMessagesVisible / number of consumers`. Target a backlog per instance that gives your consumer enough time to process within the visibility timeout.

### Message Size and Extended Client Library

SQS messages have a maximum size of 256 KB. For larger payloads, use the SQS Extended Client Library, which stores the actual payload in S3 and sends a reference pointer as the SQS message. The extended client handles the S3 upload/download transparently. Maximum payload size with the extended client is 2 GB.

### Limits and Quotas

- Maximum message size: 256 KB (2 GB with Extended Client Library)
- Maximum retention: 14 days
- Maximum visibility timeout: 12 hours
- Maximum long poll wait: 20 seconds
- Maximum batch size: 10 messages per SendMessageBatch/DeleteMessageBatch/ChangeMessageVisibilityBatch
- Maximum inflight messages: 120,000 (standard), 20,000 (FIFO)
- FIFO throughput: 300 TPS without batching, 3,000 with batching, 70,000 with high-throughput mode
- Maximum message groups per FIFO queue: unlimited
- Deduplication window: 5 minutes (FIFO only)
- Maximum delay: 15 minutes (per-message or per-queue delay)

### Pricing Model

SQS charges per API request. The first 1 million requests per month are free. After that, standard queues cost approximately $0.40 per million requests and FIFO queues cost approximately $0.50 per million requests. A single SendMessageBatch of 10 messages counts as 10 requests. Data transfer out of SQS to the internet is charged at standard AWS data transfer rates.

### Edge Cases and Gotchas

- Standard queues can deliver messages more than once. Your consumer must be idempotent or you must use FIFO.
- FIFO queues do not guarantee exactly-once when using high-throughput mode — they fall back to at-least-once.
- Setting maxReceiveCount to 1 means the message goes to the DLQ after a single failed processing attempt — you get no retries.
- Messages in the DLQ still have the original queue's retention period clock ticking. If a message sat in the source queue for 13 days before going to the DLQ, it only has 1 day left in the DLQ (assuming 14-day retention on both).
- A deleted queue name cannot be reused for 60 seconds.
- PurgeQueue can only be called once every 60 seconds.
- FIFO queue names must end in ".fifo" — this is enforced by the API.
- Long polling with WaitTimeSeconds=0 is the same as short polling.
- If you set a delay on a message and it also has a visibility timeout, the delay happens first, then the visibility timeout starts on first receive.
- Lambda event source mapping for SQS does not use Lambda Destinations for failures — it relies on the SQS DLQ configuration instead.

### Common Interview Questions

**Q: How do you ensure exactly-once processing with SQS?**
A: Use a FIFO queue (which deduplicates within a 5-minute window) combined with an idempotent consumer. The FIFO queue prevents duplicate delivery; the idempotent consumer handles the edge case where processing succeeds but the delete fails. For standard queues, you can only achieve effectively-once by making your consumer idempotent (e.g., using a DynamoDB condition expression to check if a message ID was already processed).

**Q: What happens when a Lambda function fails while processing an SQS message?**
A: The message becomes visible again after the visibility timeout expires and will be retried. If using batch processing without ReportBatchItemFailures, the entire batch returns to the queue. With ReportBatchItemFailures, only the specific failed messages return. After maxReceiveCount failures, the message moves to the DLQ if configured. Lambda does not use Destinations for SQS event source failures.

**Q: How do you handle poison messages?**
A: Configure a DLQ with a reasonable maxReceiveCount (e.g., 3-5). Monitor the DLQ with a CloudWatch alarm on ApproximateNumberOfMessagesVisible. Investigate and fix the root cause, then use DLQ redrive to replay messages back to the source queue.

### Interactions with Other Services

- **Lambda**: SQS is a Lambda event source. Lambda polls the queue, manages visibility timeout, and invokes your function with batches.
- **SNS**: SNS can fan out to multiple SQS queues. This is the classic fan-out pattern — one SNS topic publishes to N SQS queues for parallel processing.
- **EventBridge**: EventBridge can target SQS queues. SQS queues can also send events to EventBridge via custom rules.
- **S3**: S3 event notifications can be sent directly to SQS. Used for processing uploaded files.
- **Step Functions**: Step Functions can send messages to SQS as a service integration.
- **API Gateway**: API Gateway can proxy directly to SQS without a Lambda in between, which is a cost-effective pattern for ingesting requests.

---

## Amazon SNS (Simple Notification Service)

### Overview

SNS is a fully managed pub/sub messaging service. A publisher sends a message to a topic, and SNS delivers copies of that message to all subscribers of the topic. Subscribers can be SQS queues, Lambda functions, HTTP/HTTPS endpoints, email addresses, SMS numbers, or mobile push endpoints. SNS is fire-and-forget — it does not persist messages. If a subscriber is down when the message is published, the message may be lost (unless the subscriber is SQS, which has its own persistence).

### How It Works Internally

When you publish a message to an SNS topic, SNS stores the message temporarily and immediately begins delivering it to all active subscriptions. Delivery is attempted concurrently to all subscribers. The message is not durably stored after delivery — SNS is not a message store.

For each subscriber type, SNS has different retry policies. For SQS and Lambda subscribers, SNS retries delivery with exponential backoff for up to 23 days (100,015 attempts). For HTTP/HTTPS endpoints, the retry policy is configurable (immediate retries, then linear backoff, then exponential backoff). For email and SMS, there are limited or no retries.

### Topics: Standard vs FIFO

Standard SNS topics support virtually unlimited throughput and deliver messages at least once in no particular order.

FIFO SNS topics guarantee strict ordering per message group ID and exactly-once delivery to subscribers. However, FIFO topics can only deliver to SQS FIFO queues — not to Lambda, HTTP, email, or any other subscriber type. This is a critical limitation. If you need ordered fan-out to Lambda, you must go through an SQS FIFO queue first.

### Message Filtering

Subscribers can define a filter policy that specifies which messages they want to receive. Messages that do not match the filter are not delivered to that subscriber — they are simply dropped for that subscription (not sent to a DLQ).

Filter policies operate on message attributes, not the message body (though a newer option allows body-based filtering for JSON payloads). Supported filter operators include: exact match, prefix, anything-but, numeric range, and exists/not-exists.

Message filtering is powerful because it moves routing logic from application code to the infrastructure. Instead of having a consumer receive all messages and discard irrelevant ones, SNS does the filtering for you. This saves cost (no unnecessary Lambda invocations or SQS deliveries) and reduces complexity.

### Raw Message Delivery

By default, SNS wraps the message in a JSON envelope containing metadata (TopicArn, MessageId, Timestamp, etc.) before delivering it to the subscriber. This is useful for HTTP subscribers that need context about the notification.

For SQS and HTTP subscribers, you can enable raw message delivery, which sends the original message as-is without the SNS wrapper. This is important when the consumer expects a specific message format and does not want to unwrap the SNS envelope.

### Limits and Quotas

- Maximum message size: 256 KB
- Maximum subscriptions per topic: 12,500,000
- Maximum topics per account: 100,000 (soft limit)
- Maximum filter policies per account: 200 per topic (soft limit)
- FIFO topic throughput: 300 publishes per second (or 10 MB/s), same as SQS FIFO
- FIFO topic subscribers: SQS FIFO queues only
- Message attributes: up to 10 per message
- Delivery retries (SQS/Lambda): up to 100,015 attempts over 23 days

### Pricing Model

SNS charges per publish and per delivery. The first 1 million publishes per month are free. After that, approximately $0.50 per million publishes. Deliveries to SQS and Lambda are free. Deliveries to HTTP are $0.60 per million. SMS and email have separate (higher) pricing.

### Edge Cases and Gotchas

- SNS does not persist messages. If a subscriber is unavailable and all retries are exhausted, the message is lost. Always pair SNS with SQS for durable delivery.
- FIFO topics only support SQS FIFO subscribers. You cannot have Lambda, HTTP, or email subscribers on a FIFO topic.
- Message filtering drops non-matching messages silently — there is no dead-letter mechanism for filtered-out messages by default. You can configure a DLQ per subscription for delivery failures, but filtering is not considered a failure.
- SNS delivers messages to all subscribers concurrently. There is no way to order or prioritize delivery across subscribers.
- The maximum message size is 256 KB, same as SQS. For larger payloads, you must use a reference pointer pattern (store in S3, send the S3 key).
- You cannot recall or delete a published message.
- Subscription confirmation is required for HTTP/HTTPS and email endpoints. Until confirmed, messages are not delivered.
- Cross-account subscriptions are supported but require topic policy changes.

### Common Interview Questions

**Q: What is the difference between SNS and SQS?**
A: SNS is pub/sub (one-to-many fan-out) while SQS is a queue (point-to-point or competing consumers). SNS does not persist messages; SQS stores them for up to 14 days. SNS pushes to subscribers; SQS requires consumers to poll. They are often used together: SNS fans out to multiple SQS queues, each processed independently.

**Q: How does SNS message filtering work and why is it useful?**
A: Subscribers define filter policies based on message attributes (or JSON body fields). SNS evaluates each message against each subscriber's filter and only delivers matching messages. This eliminates the need for consumers to receive and discard irrelevant messages, saving cost and reducing code complexity. The filtering happens server-side at the SNS level with no additional charge.

**Q: Can you guarantee ordered delivery with SNS?**
A: Yes, using FIFO topics with message group IDs — but only to SQS FIFO queue subscribers. Standard SNS topics do not guarantee ordering. If you need ordered processing in Lambda, you must route through SNS FIFO to SQS FIFO to Lambda.

### Dead-Letter Queue per Subscription

SNS supports configuring a DLQ (SQS queue) per subscription. If SNS fails to deliver a message to a specific subscriber after exhausting all retries, the message is sent to that subscription's DLQ. This is different from SQS DLQs (which catch consumer processing failures). SNS DLQs catch delivery failures — the message never reached the subscriber at all.

This is critical for HTTP/HTTPS subscribers, which are the most likely to be temporarily unavailable. Without a subscription DLQ, messages lost after retry exhaustion are gone forever. SQS and Lambda subscribers rarely need subscription DLQs because SNS retries to them for 23 days (100K+ attempts).

### Large Message Payloads

Like SQS, SNS has a 256 KB message size limit. The SNS Extended Client Library for Java (and equivalent patterns in other languages) stores the payload in S3 and sends a reference pointer as the SNS message. The consuming SQS Extended Client Library on the subscriber side transparently retrieves the full payload from S3.

If you use the extended client pattern, both the publishing side (SNS Extended Client) and the consuming side (SQS Extended Client) must be configured. If only one side uses the library, the consumer receives the S3 reference pointer as a raw string instead of the actual payload.

### Interactions with Other Services

- **SQS**: The classic SNS+SQS fan-out pattern. One topic, multiple queue subscribers, each processing independently.
- **Lambda**: SNS can invoke Lambda directly (asynchronous invocation). Failures follow Lambda's async retry policy (2 retries, then to a Lambda DLQ/Destination).
- **EventBridge**: EventBridge can target SNS topics. SNS cannot directly send to EventBridge.
- **S3**: S3 event notifications can publish to SNS topics.
- **CloudWatch**: CloudWatch Alarms use SNS topics for notifications.

---

## Amazon EventBridge

### Overview

EventBridge is a serverless event bus that connects applications using events. It evolved from CloudWatch Events and is now the recommended way to build event-driven architectures on AWS. Unlike SNS (which is message-oriented), EventBridge is event-oriented — events have a defined structure and are routed using content-based rules that can inspect any field in the event payload.

### Event Structure

Every EventBridge event is a JSON object with a standard envelope. The top-level fields are:

- **version**: Always "0"
- **id**: Unique event ID (UUID), generated by EventBridge
- **source**: Identifies the service or application that generated the event (e.g., "aws.ec2", "com.myapp.orders")
- **detail-type**: A free-form string describing the event type (e.g., "EC2 Instance State-change Notification", "OrderPlaced")
- **account**: The AWS account ID where the event was generated
- **time**: ISO 8601 timestamp of when the event occurred
- **region**: The AWS region
- **resources**: An array of ARNs identifying resources involved (optional)
- **detail**: A JSON object with the event-specific payload — this is where all your business data lives

Interviewers often ask about this structure because it determines how you write event patterns. The `source` and `detail-type` combination is the primary routing key. Best practice: use a reverse-DNS-style source (com.company.service) and a descriptive detail-type. This makes rules readable and prevents collisions.

### How It Works Internally

EventBridge has three types of event buses:

1. **Default event bus**: Every AWS account has one. AWS services automatically publish events here (EC2 state changes, CodePipeline events, etc.).
2. **Custom event buses**: You create these for your application events. You can share them across accounts.
3. **Partner event buses**: Third-party SaaS providers (Datadog, Zendesk, Auth0, etc.) publish events to these.

When an event arrives on a bus, EventBridge evaluates it against all rules on that bus. Each rule has an event pattern (the filter) and one or more targets (the actions). If an event matches a rule's pattern, EventBridge delivers the event to all of that rule's targets.

EventBridge processes events asynchronously with at-least-once delivery. There is no ordering guarantee — events may arrive out of order and may be delivered more than once.

### Rules and Event Patterns

Rules are the routing mechanism. An event pattern is a JSON object that defines which events match. EventBridge supports sophisticated matching:

- **Exact match**: `"source": ["my.app"]` — matches events where source is exactly "my.app"
- **Prefix**: `"source": [{"prefix": "my."}]` — matches sources starting with "my."
- **Suffix**: `"detail-type": [{"suffix": ".created"}]` — matches detail-types ending with ".created"
- **Numeric**: `"detail": {"price": [{"numeric": [">", 100]}]}` — matches when price > 100
- **Exists**: `"detail": {"error": [{"exists": true}]}` — matches when the error field exists
- **Anything-but**: `"source": [{"anything-but": "internal"}]` — matches any source except "internal"
- **Wildcard**: `"detail": {"key": [{"wildcard": "prod-*-east"}]}` — glob-style matching

You can combine these operators and nest them deeply within the event structure. This is far more powerful than SNS filtering, which only operates on flat message attributes.

Each rule can have up to 5 targets. If you need more, you can create additional rules with the same event pattern.

### Targets

EventBridge supports over 20 target types including Lambda, SQS, SNS, Step Functions, Kinesis, ECS tasks, CodePipeline, API Gateway, Redshift, and more. It also supports API Destinations (calling any HTTP endpoint), cross-account event buses, and same-account event buses (for event forwarding).

Each target can have an input transformer that restructures the event before delivery. This allows you to extract specific fields, add static values, or completely reshape the event payload to match what the target expects. Input transformers have two parts: an input path map (extracts values from the event into variables) and a template (assembles those variables into the output). This eliminates many "glue Lambda" functions that existed only to reshape an event before passing it to the next service.

### API Destinations

API Destinations let EventBridge call any HTTP endpoint as a rule target. You define a Connection (which holds authentication credentials — API key, OAuth client credentials, or Basic auth) and an API Destination (the HTTP endpoint URL, method, and rate limit).

Key features:
- Built-in authentication: OAuth 2.0 client credentials flow (EventBridge manages token refresh), API key (injected as a header or query parameter), Basic auth
- Rate limiting: You set a maximum invocations-per-second limit per destination. EventBridge queues excess events and delivers them as capacity allows.
- Retry and DLQ: Same retry/DLQ behavior as other targets

Use cases: calling third-party webhooks (Slack, PagerDuty, Stripe), sending events to on-premises systems via HTTPS, integrating with partner APIs without writing Lambda glue code.

### Schema Registry and Schema Discovery

EventBridge Schema Registry stores event schemas (in JSON Schema or OpenAPI 3 format) so that teams can discover and understand the events flowing through the bus.

**Schema Discovery**: When enabled on an event bus, EventBridge automatically infers schemas from observed events and registers them. This is useful for understanding what events exist in your system without documentation.

**Code bindings**: From a registered schema, EventBridge can generate code bindings (TypeScript, Python, Java) that give you typed objects for producing and consuming events. This catches structural errors at compile time instead of runtime.

Schema Registry is most valuable in large organizations where multiple teams publish events and consumers need to discover what is available. It replaces tribal knowledge and out-of-date wikis with a live registry.

### Global Endpoints

EventBridge Global Endpoints provide automatic failover for event ingestion across two regions. You configure a primary and secondary region, and a Route 53 health check determines which region receives events. If the primary region becomes unhealthy, EventBridge routes PutEvents calls to the secondary region automatically.

Events published to the secondary region during failover can be replicated back to the primary region when it recovers (using event replication rules). This enables active-passive event ingestion without application-level failover logic.

### Archive and Replay

EventBridge can archive events that match a specific pattern. Archived events are stored in a managed, compressed format and can be replayed later. This is powerful for:

- Debugging production issues by replaying events through the system
- Populating a new service with historical events
- Testing event-driven workflows with real production data

Archives have configurable retention (indefinite or a specific number of days). Replay sends events back to the same event bus, where they are re-evaluated against current rules.

### EventBridge Scheduler

EventBridge Scheduler is a separate service (not part of event bus rules) for scheduling one-time or recurring invocations. It supports cron expressions and rate expressions, just like the older CloudWatch Events cron rules, but with significant improvements:

- Over 200 target types (vs. the limited targets of cron rules)
- Built-in retry policies and DLQ support
- Flexible time windows (for spreading load)
- One-time schedules (at a specific datetime)
- Up to 1 million schedules per account (vs. 300 rules per event bus)

Use Scheduler for any "do this at a specific time" or "do this every N minutes" use case. Use event bus rules for "when this event happens, do that."

### EventBridge Pipes

Pipes connect a source to a target with optional filtering, enrichment, and transformation in between. Supported sources include SQS, Kinesis, DynamoDB Streams, and Managed Streaming for Kafka. The pipe reads from the source, optionally filters events, optionally enriches them (calling Lambda, Step Functions, API Gateway, or API Destination), and delivers to a target.

Pipes are useful when you want point-to-point integration without the fan-out semantics of an event bus. They replace patterns that previously required a Lambda function gluing two services together.

### Dead-Letter Queues

When EventBridge fails to deliver an event to a target (the target returns an error or is unavailable), it retries with exponential backoff for up to 24 hours (185 retries by default). You can configure the retry policy (maximum age and maximum retry attempts) per target. If all retries fail, the event is dropped — unless you configure a DLQ (an SQS queue) on the target. The DLQ receives the failed event along with error details.

This DLQ is configured per target, not per rule or per bus. If a rule has 5 targets and only one fails, only that target's DLQ receives the failed event.

### Cross-Account and Cross-Region

EventBridge supports cross-account event delivery. You can set up rules that send events to an event bus in another account. This requires a resource-based policy on the destination event bus allowing the source account to PutEvents.

Cross-region event delivery is also supported by creating rules that target an event bus in another region. This is the foundation for multi-region event-driven architectures.

### Limits and Quotas

- Maximum event size: 256 KB
- Maximum rules per event bus: 300 (soft limit, can be increased)
- Maximum targets per rule: 5
- Throughput: varies by region, typically thousands of events per second (soft limit, can be increased significantly)
- Archive retention: up to indefinite
- Scheduler: 1,000,000 schedules per account
- Pipes: 1 source, 1 target per pipe
- Retry policy: up to 185 retries over 24 hours (configurable)
- Latency: typically sub-second, but not guaranteed (at-least-once, no ordering)

### Pricing Model

EventBridge charges per event published to a custom or partner event bus. Events published by AWS services to the default event bus are free. Custom events cost approximately $1.00 per million events. Scheduler invocations cost $1.00 per million. Pipes are charged per request. Archive and replay are charged per GB stored and per GB replayed.

### Edge Cases and Gotchas

- EventBridge is at-least-once with no ordering guarantee. Consumers must be idempotent.
- Events are immutable once published — you cannot recall or modify them.
- The 256 KB event size limit is strict. Unlike SQS (which has the Extended Client Library), EventBridge has no built-in mechanism for oversized events. You must use a claim-check pattern (store payload in S3, send reference in event).
- Rule evaluation order is not guaranteed. If two rules match the same event, both targets are invoked but the order is undefined.
- Archive replay sends events to the bus, not directly to targets. This means replayed events are evaluated against current rules, not the rules that existed when the event was originally published. This can be a feature or a trap depending on your use case.
- The default event bus receives all AWS service events. Be careful with overly broad rules — you might match events you did not intend.
- PutEvents API supports batching up to 10 events per call. Each event in the batch is evaluated independently.
- Cross-account delivery requires explicit resource policies on both sides.
- EventBridge Scheduler and event bus cron rules are different services. Scheduler is newer and more capable. Prefer Scheduler for new workloads.

### Common Interview Questions

**Q: When would you choose EventBridge over SNS?**
A: Choose EventBridge when you need content-based routing (filtering on any field in the event body), when you want to integrate with SaaS partners, when you need archive and replay, or when you have many consumers with different filtering needs. Choose SNS when you need simple fan-out with minimal filtering, or when you need to deliver to email/SMS/mobile push. EventBridge is also better for decoupled architectures because producers do not need to know about consumers — they just publish events.

**Q: How does EventBridge handle failures?**
A: When a target fails, EventBridge retries with exponential backoff (configurable, up to 24 hours / 185 retries). If all retries fail, the event goes to the target's DLQ (if configured) or is dropped. Each target has its own retry policy and DLQ, so failure of one target does not affect others.

### Interactions with Other Services

- **Lambda**: The most common EventBridge target. EventBridge invokes Lambda asynchronously.
- **Step Functions**: EventBridge can start Step Functions executions directly.
- **SQS/SNS**: Both can be EventBridge targets for buffering or further fan-out.
- **S3**: S3 events can be routed through EventBridge (must be enabled per bucket) instead of native S3 event notifications. This is preferred when you need content-based filtering or multiple consumers for the same event type.
- **CloudTrail**: API call events flow through EventBridge via the default event bus, enabling you to react to any AWS API call.

---

## Amazon Kinesis

### Overview

Kinesis is a platform for real-time streaming data. The core product is Kinesis Data Streams, which ingests and stores streaming data for real-time processing. Unlike SQS (which is a queue where each message is consumed once), Kinesis is a log where multiple consumers can read the same data independently, and data is retained for a configurable period.

### How It Works Internally

A Kinesis data stream is composed of shards. Each shard is an ordered sequence of records with a fixed capacity: 1 MB/s or 1,000 records/s for writes, and 2 MB/s for reads. When a producer puts a record, it specifies a partition key, which Kinesis hashes to determine which shard receives the record. Records with the same partition key always go to the same shard, guaranteeing ordering per partition key.

Each record in a shard gets a unique sequence number. Consumers track their position in the stream using these sequence numbers (called checkpointing). Multiple consumers can read the same shard independently, each maintaining their own checkpoint.

Data retention is 24 hours by default, extendable up to 365 days (7 days for standard retention, beyond that is long-term retention at a higher cost).

### Kinesis Client Library (KCL)

The KCL is a Java library (with multi-language support via the MultiLangDaemon) that simplifies building Kinesis consumers. It handles shard discovery, load balancing across multiple worker instances, checkpointing, and lease management. The KCL uses a DynamoDB table to track shard leases and checkpoints.

Key KCL behavior: one KCL worker processes one shard. If you have 10 shards, you can have up to 10 workers running in parallel. If you have 5 workers and 10 shards, each worker processes 2 shards. You cannot have more workers than shards (excess workers sit idle).

### Kinesis Producer Library (KPL)

The KPL is the producer-side counterpart to the KCL. It provides two key optimizations:

**Aggregation**: The KPL combines multiple user records into a single Kinesis record. Since Kinesis charges per PUT payload unit (25 KB), sending many small records individually is wasteful. The KPL packs multiple small records into one Kinesis record (up to the 1 MB limit), dramatically reducing costs and increasing throughput.

**Collection**: The KPL batches multiple Kinesis records into a single PutRecords API call (up to 500 records). This reduces HTTP overhead and improves throughput.

The KPL also handles retries with backoff, rate limiting, and CloudWatch metrics emission. It operates asynchronously — your producer code sends records to the KPL, which buffers, aggregates, and flushes them in the background.

Important gotcha: If you use KPL aggregation on the producer side, your consumer must use the KCL or the Kinesis Client Library de-aggregation module to unpack the aggregated records. Lambda has built-in KPL de-aggregation support if you enable it in the event source mapping configuration.

### Shard Iterator Types

When a consumer starts reading from a shard, it must specify where to begin. There are four iterator types:

- **TRIM_HORIZON**: Start at the oldest record still available in the shard. Use for processing all data from the beginning.
- **LATEST**: Start at the newest record and only read records arriving after the consumer starts. Use when you only care about new data.
- **AT_TIMESTAMP**: Start at a specific timestamp. Use for replaying data from a known point in time.
- **AT_SEQUENCE_NUMBER / AFTER_SEQUENCE_NUMBER**: Start at or after a specific sequence number. Use when resuming from a checkpoint.

When Lambda is the consumer, it starts at TRIM_HORIZON by default (processes everything in the shard). You can configure the starting position when creating the event source mapping.

### On-Demand vs Provisioned Mode

**Provisioned mode**: You manage the shard count manually. Each shard provides 1 MB/s write and 2 MB/s read. You split or merge shards to adjust capacity. You pay per shard-hour whether the shard is utilized or not.

**On-demand mode**: Kinesis automatically manages shards based on traffic. It scales up within seconds as throughput increases and scales down when traffic drops. Capacity starts at 4 MB/s write and 8 MB/s read and scales up to 200 MB/s write and 400 MB/s read per stream.

On-demand is simpler (no shard management) but more expensive per GB than a well-tuned provisioned stream. It may also take minutes to fully scale during sudden, extreme spikes. Use on-demand for unpredictable workloads or when operational simplicity is more important than cost optimization. Use provisioned for predictable, high-throughput workloads where you want tight cost control.

### Enhanced Fan-Out

Standard consumers share the 2 MB/s read throughput per shard. If you have 5 consumers reading from a shard, they share 2 MB/s. Enhanced fan-out gives each consumer a dedicated 2 MB/s per shard via HTTP/2 push (instead of polling). This is critical when you have multiple consumers that all need low-latency access to the same data.

Enhanced fan-out consumers register with the stream and receive data pushed to them via SubscribeToShard. There is a limit of 20 enhanced fan-out consumers per stream.

### Kinesis Data Streams vs Firehose vs Analytics

- **Data Streams**: Real-time ingestion and processing. You manage consumers, scaling (shard count), and checkpointing. Sub-second latency.
- **Firehose (now called Data Firehose)**: Managed delivery to destinations (S3, Redshift, OpenSearch, Splunk, HTTP endpoints). Near-real-time (60-second minimum buffer interval). No shard management. Use when you want zero-management delivery to a data store.
- **Analytics (now called Managed Service for Apache Flink)**: SQL or Flink-based processing of streaming data. Use when you need windowed aggregations, joins, or complex stream processing.

### When Kinesis vs SQS

Use Kinesis when:
- Multiple consumers need to read the same data independently
- You need strict ordering per partition key
- You need to replay data (retention up to 365 days)
- You have high-throughput real-time streaming (logs, metrics, clickstreams)
- You need sub-second processing latency

Use SQS when:
- Each message should be processed once by one consumer (or a competing consumer pool)
- You do not need ordering (or use FIFO for ordered processing)
- You want fully serverless with no capacity management
- Message-level acknowledgment is important (delete after processing)
- You have variable throughput with unpredictable spikes

The key conceptual difference: SQS is a queue (messages are consumed and removed), Kinesis is a log (records are retained for a period and can be read multiple times by multiple consumers).

### Limits and Quotas

- Write: 1 MB/s or 1,000 records/s per shard
- Read: 2 MB/s per shard (shared) or 2 MB/s per consumer per shard (enhanced fan-out)
- Maximum record size: 1 MB
- Default retention: 24 hours (extendable to 365 days)
- Maximum shards per stream: varies by region, typically 500 (soft limit)
- Enhanced fan-out consumers per stream: 20
- PutRecords batch: up to 500 records per call
- On-demand mode: automatically manages shards, scales up to 200 MB/s write and 400 MB/s read

### Pricing Model

Kinesis charges per shard-hour for provisioned capacity, plus per PUT payload unit (25 KB units). Enhanced fan-out charges per consumer-shard-hour plus per GB of data retrieved. Extended retention has additional per-GB charges. On-demand mode charges per GB written and read, with no shard management.

### Edge Cases and Gotchas

- Hot shards: if one partition key receives disproportionate traffic, the shard for that key becomes a bottleneck. Use high-cardinality partition keys or add random suffixes to distribute load.
- Resharding (splitting/merging shards) is manual in provisioned mode and takes time. The stream is still available during resharding, but it creates parent-child shard relationships that consumers must handle correctly.
- KCL uses a DynamoDB table for lease management. If this table is throttled (insufficient capacity), your consumers stall. Provision sufficient RCU/WCU or use on-demand.
- GetRecords has a limit of 5 calls per second per shard. If you have more than 5 standard consumers on a shard, use enhanced fan-out.
- When Lambda is a Kinesis consumer, it processes one batch per shard concurrently (by default). You can increase parallelization factor (up to 10) to process multiple batches per shard.
- Kinesis does not have a built-in DLQ. If a Lambda consumer fails on a batch, it retries the same batch until it succeeds or the data expires from the stream (blocking the shard). Configure a maximum retry age and on-failure destination to avoid this.
- On-demand mode handles scaling automatically but may take minutes to react to sudden traffic spikes. If you know the traffic pattern in advance, provisioned mode with pre-scaled shards is more predictable.

### Common Interview Questions

**Q: How does Kinesis guarantee ordering?**
A: Kinesis guarantees ordering per shard. Records with the same partition key go to the same shard and are stored in the order received, with monotonically increasing sequence numbers. Consumers read records from a shard in order. However, there is no ordering guarantee across different shards.

**Q: What happens when a Kinesis shard reaches its throughput limit?**
A: Write attempts are throttled (ProvisionedThroughputExceededException). You must split the shard into two shards to increase capacity, or switch to on-demand mode which handles this automatically. The producer should implement retries with backoff.

**Q: How is Kinesis different from Kafka?**
A: Kinesis is a managed service — no brokers, ZooKeeper, or clusters to manage. It uses shards (vs Kafka partitions), has simpler scaling (split/merge), and integrates natively with AWS services. Kafka generally offers higher throughput per partition, more flexible consumer group semantics, log compaction, and unlimited retention. Amazon MSK is the managed Kafka offering if you need Kafka-specific features.

### Interactions with Other Services

- **Lambda**: Kinesis is a Lambda event source. Lambda polls shards and invokes functions with batches of records.
- **Firehose**: Kinesis Data Streams can feed into Firehose for delivery to S3, Redshift, etc.
- **EventBridge Pipes**: Pipes can read from Kinesis and route to any supported target with filtering and enrichment.
- **S3**: Through Firehose, or through custom Lambda consumers.
- **DynamoDB Streams**: Conceptually similar (ordered per-partition log), but DynamoDB Streams is for change data capture of a DynamoDB table, not general-purpose streaming.

---

## Comparison: When to Use What

### SQS vs SNS

SQS is point-to-point (or competing consumers). One message is processed by one consumer. Use it for work queues, task processing, and decoupling with guaranteed delivery.

SNS is pub/sub. One message is delivered to many subscribers. Use it for fan-out, notifications, and broadcasting events. SNS does not persist messages — pair it with SQS for durability.

The classic pattern is SNS+SQS fan-out: publish to SNS, subscribe multiple SQS queues, each queue is processed independently. This gives you both fan-out (SNS) and durability with independent processing rates (SQS).

### SQS vs Kinesis

Both decouple producers from consumers, but they have fundamentally different models.

SQS is a queue: messages are consumed and deleted. Each message is processed by one consumer. Throughput is virtually unlimited. No ordering (standard) or ordered within message groups (FIFO). Perfect for task processing where each task needs to be done once.

Kinesis is a log: records are retained and can be read by multiple consumers. Ordering is guaranteed per shard. Throughput is bounded by shard count. Perfect for streaming data where multiple downstream systems need the same data (analytics, search indexing, monitoring).

Decision questions:
- Do multiple systems need the same data? Kinesis (or SNS+SQS fan-out).
- Do you need to replay data? Kinesis.
- Is each message a task to be done once? SQS.
- Do you need sub-second latency? Kinesis.
- Do you want zero capacity management? SQS (or Kinesis on-demand, but SQS is simpler).

### SNS vs EventBridge

Both are pub/sub, but they serve different architectural purposes.

SNS is simpler and optimized for high-throughput fan-out. Filtering is limited to message attributes. Best for notification-style use cases and when you need email/SMS/push delivery.

EventBridge is more powerful. Content-based routing on any field in the event body. Archive and replay. Schema registry. SaaS partner integrations. API Destinations for HTTP calls. Better for event-driven architectures where routing logic is complex.

Use SNS when: simple fan-out, need email/SMS, extremely high throughput (millions per second), message attributes are sufficient for filtering.

Use EventBridge when: complex routing based on event content, need archive/replay, building event-driven microservices, SaaS integrations, need event schema discovery.

### EventBridge vs Kinesis

EventBridge is for event routing — getting the right event to the right target with content-based filtering. It is at-least-once with no ordering guarantee.

Kinesis is for data streaming — processing a high-volume, ordered stream of records in real time. It guarantees ordering per partition key and allows multiple consumers.

Use EventBridge when: you need to route diverse events to different targets based on content. Use Kinesis when: you have a firehose of data (logs, metrics, clickstreams) that needs ordered, real-time processing by multiple consumers.

### The Grand Decision Tree

1. Is this a notification/alert (email, SMS, push)? Use SNS.
2. Is this a task queue where each item should be processed once? Use SQS.
3. Do multiple consumers need the same ordered stream of data? Use Kinesis.
4. Do you need content-based routing of events to different targets? Use EventBridge.
5. Do you need fan-out to multiple independent processors? Use SNS+SQS.
6. Do you need ordered processing with exactly-once semantics? Use SQS FIFO.
7. Do you need to replay historical events? Use EventBridge (archive) or Kinesis (retention).

---

## Common Messaging Patterns

### Fan-Out

One event triggers processing by multiple independent consumers. The canonical implementation is SNS+SQS: publish to an SNS topic, subscribe N SQS queues, each queue is processed by its own consumer at its own pace. Alternatively, use EventBridge with multiple rules targeting different services.

Fan-out decouples the producer from the number and identity of consumers. Adding a new consumer means adding a new subscription — the producer changes nothing. This is the foundation of extensible, loosely coupled architectures.

### Competing Consumers

Multiple consumers read from the same SQS queue. Each message is delivered to only one consumer. This pattern provides horizontal scalability — add more consumers to increase throughput. SQS standard queues support this natively. The visibility timeout prevents two consumers from processing the same message simultaneously.

Key design consideration: consumers must be idempotent (standard queues) or use FIFO (for ordering). Autoscale the consumer fleet based on the ApproximateNumberOfMessagesVisible metric.

### Claim-Check

When the payload exceeds the messaging service's size limit (256 KB for SQS/SNS/EventBridge, 1 MB for Kinesis), use the claim-check pattern: store the full payload in S3, send only a reference (bucket + key) in the message. The consumer retrieves the full payload from S3 using the reference.

The SQS and SNS Extended Client Libraries implement this pattern transparently. For EventBridge and Kinesis, you must implement it yourself. The claim-check pattern also reduces data transfer costs and message processing time.

### Request-Response with Temporary Queues

For synchronous-style communication over SQS: the requester creates a temporary response queue (or reuses one with a correlation ID), sends the request with a ReplyTo attribute pointing to the response queue, and polls the response queue for the answer. The SQS Temporary Queue Client automates this pattern, creating virtual queues that share a single physical queue to reduce resource overhead.

This pattern is useful when you need request-response semantics but want the decoupling and buffering benefits of a queue. However, if you truly need synchronous responses, consider API Gateway + Lambda instead — it is simpler and lower latency.

### Priority Queue

SQS does not support message priority natively. The standard pattern is to create multiple queues (high, medium, low priority) and have consumers poll the high-priority queue first, then medium, then low. Alternatively, use a single queue with message attributes indicating priority and have consumers process high-priority messages before others.

The multi-queue approach is simpler and more reliable. The single-queue approach requires consumer logic to sort and prioritize, which adds complexity and does not guarantee strict priority ordering (a consumer may receive a low-priority message before a high-priority one arrives).

### Dead Letter Queue Monitoring Pattern

Every production SQS queue should have a DLQ with a CloudWatch alarm on ApproximateNumberOfMessagesVisible > 0. When the alarm fires, investigate the DLQ messages (inspect message attributes and body), fix the root cause (bug in consumer, schema change, downstream outage), and replay messages using DLQ redrive.

For critical workflows, add a Lambda function triggered by the DLQ that logs the failure details, notifies the team (via SNS), and optionally stores the message in DynamoDB or S3 for analysis.

---

## Amazon MQ and Amazon MSK

### When to Use Traditional Brokers

AWS-native services (SQS, SNS, EventBridge, Kinesis) are the default choice for new applications. However, traditional message brokers have their place:

### Amazon MQ

Amazon MQ is a managed broker service for Apache ActiveMQ and RabbitMQ. Use it when migrating existing applications that depend on standard messaging protocols (AMQP 0-9-1, AMQP 1.0, MQTT, OpenWire, STOMP). If your application already uses RabbitMQ or ActiveMQ on-premises and you want a lift-and-shift migration without rewriting messaging code, Amazon MQ provides wire-protocol compatibility.

Amazon MQ is not serverless — you choose broker instance types, manage storage, and handle failover configuration. It has lower throughput than SQS and does not scale automatically. It is a migration tool, not a greenfield choice.

Key decision: If you are building a new application, use SQS/SNS/EventBridge. If you are migrating an existing application that uses JMS, AMQP, or MQTT, use Amazon MQ.

### Amazon MSK (Managed Streaming for Apache Kafka)

MSK is managed Apache Kafka. Use it when you need Kafka-specific features that Kinesis does not provide:

- **Log compaction**: Kafka retains only the latest value per key, creating an always-current snapshot. Kinesis does not support this.
- **Unlimited retention**: Kafka can retain data indefinitely (with tiered storage). Kinesis maxes at 365 days.
- **Consumer groups**: Kafka's consumer group semantics are more flexible than Kinesis KCL for some use cases.
- **Kafka Connect**: A rich ecosystem of connectors for databases, search engines, and other systems.
- **Ecosystem compatibility**: If your data platform already uses Kafka (Kafka Streams, ksqlDB, Schema Registry), MSK provides compatibility.

MSK is not serverless by default — you manage broker instances and storage. MSK Serverless exists but has limitations (no compaction, limited partition count). MSK is more operationally complex than Kinesis but provides more features for Kafka-native workloads.

Key decision: If you need a managed streaming platform and have no existing Kafka dependency, use Kinesis. If you need Kafka-specific features or have an existing Kafka ecosystem, use MSK.
