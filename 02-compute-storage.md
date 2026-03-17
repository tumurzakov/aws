# Part 2: Core Compute and Storage

---

## AWS Lambda

### Overview

Lambda is a serverless compute service that runs your code in response to events. You upload your code (as a ZIP or container image), configure a trigger (API Gateway, SQS, S3 event, schedule, etc.), and Lambda handles provisioning, scaling, and infrastructure management. You pay only for the compute time you consume, billed per millisecond.

### How It Works Internally

When a Lambda function is invoked, the service must find or create an execution environment to run it. This execution environment is a lightweight Firecracker microVM — the same technology that powers AWS Fargate. Each execution environment runs a single concurrent invocation of your function.

**Cold start**: When no warm execution environment is available, Lambda must create one. This involves: downloading your code, starting the runtime (Node.js, Python, Java, etc.), running your initialization code (code outside the handler). Cold starts typically add 100ms-1s for interpreted languages (Python, Node.js) and 1-10s for compiled languages (Java, .NET). Container images have similar cold start times to ZIP packages of equivalent size because Lambda caches the image layers.

**Warm start**: After a function invocation completes, the execution environment is frozen (not terminated) and reused for subsequent invocations. This means your initialization code runs only once, and subsequent invocations skip straight to the handler. Warm environments persist for minutes to hours depending on traffic patterns (Lambda does not publish exact timeouts).

The /tmp directory (up to 10 GB, configurable) persists across warm invocations. This is useful for caching downloaded files, database connections, or ML models. However, you should never rely on /tmp being present — always code defensively for cold starts.

### Memory and CPU Scaling

Lambda memory is configurable from 128 MB to 10,240 MB (10 GB). CPU is allocated proportionally to memory — at 1,769 MB you get one full vCPU, at 10 GB you get approximately 6 vCPUs. There is no way to configure CPU independently of memory.

This means that CPU-bound workloads need more memory even if they do not use the extra RAM. A function doing heavy JSON parsing or image processing will run faster (and often cheaper) with 1 GB of memory than 256 MB, because it gets 4x the CPU. Use AWS Lambda Power Tuning (a Step Functions-based tool) to find the optimal memory setting for your function.

### Concurrency

Concurrency is the number of function instances running simultaneously. Lambda manages this automatically, but there are controls:

- **Unreserved concurrency**: The default pool shared by all functions in the account. Account-level limit is 1,000 concurrent executions (soft limit, can be increased to tens of thousands).
- **Reserved concurrency**: Guarantees a fixed number of concurrent instances for a specific function, carved from the account limit. Also acts as a maximum — the function cannot exceed this number. Setting reserved concurrency to 0 effectively disables the function.
- **Provisioned concurrency**: Pre-initializes a specified number of execution environments, eliminating cold starts. These environments are always warm and ready. You pay for them whether or not they are used. Provisioned concurrency is configured per function version or alias (not $LATEST).

The burst limit is how quickly Lambda can scale up concurrency. In most regions, it can add 500-3,000 instances per minute (varies by region), with the first burst of 500-3,000 being instantaneous. After the initial burst, it scales at 500 instances per minute. If your traffic spikes faster than this, invocations are throttled.

### Event Source Mappings and Invocation Models

Lambda has three invocation models:

1. **Synchronous**: The caller waits for the response. API Gateway, ALB, and direct Invoke calls are synchronous. If the function fails, the caller gets the error immediately.

2. **Asynchronous**: The caller gets an immediate 202 response. Lambda manages retries (2 retries by default) and can send failures to a Destination (SQS, SNS, EventBridge, or another Lambda) or a DLQ (SQS or SNS only). S3 events, SNS, EventBridge, and CloudWatch Events use async invocation.

3. **Event source mapping (polling)**: Lambda polls a source (SQS, Kinesis, DynamoDB Streams, Kafka, MQ) and invokes your function with batches of records. The polling behavior and error handling vary by source type.

**Destinations vs DLQ**: Destinations are the newer, more flexible mechanism. A Destination receives the full invocation result (success or failure), including the response payload. A DLQ only receives failed events and only the event payload, not the error. Destinations support 4 target types (SQS, SNS, EventBridge, Lambda); DLQ supports only SQS and SNS. Use Destinations for new workloads.

Important: Destinations work only with asynchronous invocations. For event source mappings (SQS, Kinesis), Destinations are used for failure handling, but the mechanism is different — it is the event source mapping's on-failure destination, not the async invocation destination.

### VPC-Attached Lambda

By default, Lambda functions run in an AWS-managed VPC with internet access. When you attach a Lambda to your own VPC (to access private resources like RDS or ElastiCache), the function runs inside your VPC subnets.

Since 2019, VPC-attached Lambda uses Hyperplane ENIs (shared elastic network interfaces) that are created when the function is configured, not at cold start time. This eliminated the old 10-15 second cold start penalty for VPC Lambda. Now, VPC-attached functions have cold starts comparable to non-VPC functions.

However, VPC-attached Lambda loses internet access unless you configure a NAT Gateway or VPC endpoints. This is a common gotcha — your function cannot call AWS APIs or external services without a path to the internet. For AWS services, use VPC endpoints (gateway endpoints for S3 and DynamoDB are free; interface endpoints for other services cost money). For internet access, use a NAT Gateway ($0.045/hr + $0.045/GB).

### Recursive Loop Detection

Lambda automatically detects recursive loops (Lambda -> SQS -> Lambda -> SQS in an infinite cycle) and stops the loop after approximately 16 iterations. This applies to Lambda-SQS-Lambda and Lambda-SNS-Lambda loops. The detection sends a notification via AWS Health Dashboard and stops invoking the function for the detected loop.

### Timeout

Lambda functions have a maximum timeout of 15 minutes (900 seconds). If your function exceeds this, it is forcefully terminated. Important consideration: if your Lambda calls a downstream service (an API, a database query), that downstream call may have its own timeout. Set Lambda's timeout to be greater than the sum of all downstream timeouts plus processing time.

### Layers

Lambda Layers let you package libraries, custom runtimes, or other dependencies separately from your function code. A function can use up to 5 layers. The total unzipped size (function code + all layers) cannot exceed 250 MB (for ZIP deployments).

### Limits and Quotas

- Maximum memory: 10,240 MB
- Maximum timeout: 15 minutes
- Maximum deployment package size: 50 MB (ZIP, compressed) / 250 MB (unzipped) / 10 GB (container image)
- Maximum /tmp storage: 10,240 MB
- Maximum environment variables: 4 KB total
- Maximum concurrent executions: 1,000 per account (soft limit)
- Maximum payload: 6 MB (synchronous), 256 KB (asynchronous)
- Maximum layers: 5 per function
- Burst concurrency: 500-3,000 (region-dependent)

### Pricing Model

Lambda charges per request ($0.20 per million) and per GB-second of compute time. The first 1 million requests and 400,000 GB-seconds per month are free. Provisioned concurrency has an additional charge for the pre-initialized environments.

### Edge Cases and Gotchas

- Cold starts are not configurable or predictable. If you need consistent latency, use provisioned concurrency.
- The /tmp directory persists between warm invocations but is not shared across concurrent instances. Do not use it for coordination between invocations.
- Lambda environment variables have a 4 KB total limit. For larger configuration, use SSM Parameter Store or Secrets Manager.
- The 6 MB synchronous payload limit includes both request and response. If your API returns large responses, consider streaming or returning presigned S3 URLs.
- Lambda functions are stateless. Any state (database connections, caches) stored in the execution environment may disappear at any time.
- Reserved concurrency of 0 disables the function. Reserved concurrency also limits the function — setting it to 10 means no more than 10 concurrent executions, which can cause throttling under load.
- $LATEST is a mutable pointer to the latest code. It cannot be used with provisioned concurrency or aliases for traffic shifting. Publish versions for production use.
- Lambda retries asynchronous invocations twice with delays between retries. If your function is not idempotent, these retries can cause duplicate processing.
- Container image Lambda functions share the same cold start characteristics as ZIP functions because Lambda caches image layers. However, the initial image pull (on first deployment) may be slower.
- Recursive loop detection only works for SQS and SNS loops, not for all possible recursive patterns.

### Common Interview Questions

**Q: How do you eliminate Lambda cold starts?**
A: Use provisioned concurrency to pre-initialize a fixed number of execution environments. Alternatively, minimize cold start impact by keeping deployment packages small, using interpreted languages (Python/Node.js over Java), moving initialization code outside the handler so it runs during container init, and using Lambda SnapStart (for Java) which snapshots the initialized state.

**Q: When should you NOT use Lambda?**
A: When you need consistent sub-100ms latency (cold starts), when processing takes longer than 15 minutes, when you need persistent websocket connections (use Fargate or ECS instead), when you have sustained high-throughput compute (EC2 or Fargate is cheaper), or when you have large deployment packages exceeding 10 GB.

**Q: How does Lambda handle failures for different event sources?**
A: For synchronous invocations, the error is returned to the caller. For asynchronous invocations, Lambda retries twice, then sends to a Destination or DLQ. For event source mappings (SQS), the message returns to the queue after visibility timeout and is retried until maxReceiveCount is reached, then goes to the SQS DLQ. For event source mappings (Kinesis/DynamoDB Streams), Lambda retries the entire batch (blocking the shard) until success, data expiry, or maximum retry age — configure an on-failure destination and bisect-on-error to prevent shard blocking.

### Interactions with Other Services

- **API Gateway**: The most common synchronous trigger. API Gateway routes HTTP requests to Lambda.
- **SQS**: Lambda polls SQS and processes batches. Failed messages return to the queue.
- **S3**: S3 events trigger Lambda asynchronously for file processing workflows.
- **DynamoDB Streams**: Lambda processes change data capture events from DynamoDB.
- **EventBridge**: EventBridge invokes Lambda asynchronously as a rule target.
- **Step Functions**: Step Functions orchestrate multiple Lambda invocations.
- **CloudWatch**: Scheduled Lambda invocations via EventBridge Scheduler or CloudWatch Events.
- **RDS Proxy**: Use RDS Proxy to pool database connections from many Lambda instances to Aurora/RDS.

---

## Amazon ECS (Elastic Container Service) and EKS (Elastic Kubernetes Service)

### Overview

ECS and EKS are managed container orchestration services. ECS is an AWS-native service designed for simplicity and deep integration with AWS. EKS is a managed Kubernetes service for organizations that want Kubernetes compatibility and portability.

### ECS: Two Launch Types

**Fargate**: Serverless compute for containers. You pay for the vCPU and memory you request per task. No servers to manage, no scaling groups to configure. This is the "Serverless" way to run containers.

**EC2**: You manage a cluster of EC2 instances (using an Auto Scaling Group). You are responsible for patching, scaling, and managing the instances. Use this when you need specific instance types (GPUs, high-memory), need to run containers that require host-level access, or want to save money with Reserved Instances/Savings Plans on the underlying EC2.

### ECS Key Concepts

- **Task Definition**: A blueprint (JSON) that defines your container (image, CPU, memory, environment variables, logging).
- **Task**: A running instance of a Task Definition.
- **Service**: Manages the desired number of tasks, integrates with an ALB, and handles deployments (rolling updates).
- **Cluster**: A logical grouping of services and tasks.

### ECS Deployment Strategies

- **Rolling update** (default): Replace tasks gradually. Configure minimum healthy percent and maximum percent to control rollout speed. No additional infrastructure cost.
- **Blue/Green with CodeDeploy**: Create a new target group with new tasks, shift traffic (all-at-once, canary, or linear), automatically roll back on CloudWatch alarm. Requires CodeDeploy.
- **Circuit breaker**: ECS built-in deployment circuit breaker detects failures during rolling updates and automatically rolls back if new tasks fail to stabilize. Enable with `deploymentCircuitBreaker: { enable: true, rollback: true }`.

### ECS Auto Scaling

ECS Service Auto Scaling uses Application Auto Scaling to adjust the desired task count based on metrics:
- **Target tracking**: Maintain a target value for a metric (e.g., average CPU at 70%, ALB requests per target at 1,000). ECS adds/removes tasks to stay near the target.
- **Step scaling**: Add/remove tasks based on CloudWatch alarm thresholds with configurable steps.
- **Scheduled scaling**: Set desired count at specific times (peak hours, batch windows).

For Fargate, scaling only adjusts task count. For EC2 launch type, you also need EC2 Auto Scaling for the underlying instances (Capacity Providers manage this relationship).

### EKS: Managed Kubernetes

EKS provides a managed Kubernetes control plane (API server, etcd) across multiple AZs. You manage the worker nodes. EKS is more complex but more flexible than ECS. Use EKS if you already use Kubernetes on-premises, need CRDs/operators, or want Kubernetes portability.

### EKS Worker Node Options

**Managed node groups**: EKS manages the EC2 Auto Scaling Group for you. Automatic AMI updates, graceful draining, and simplified lifecycle management. You choose the instance type, EKS handles the rest.

**Self-managed node groups**: You manage the EC2 instances and Auto Scaling Group directly. Full control over AMI, bootstrap scripts, and instance configuration. More operational overhead but maximum flexibility.

**Fargate profiles**: Serverless pods — no nodes to manage. Each pod runs in its own Firecracker microVM. Define which pods run on Fargate using namespace and label selectors. Limited: no DaemonSets, no privileged containers, no GPUs, no EBS volumes (EFS only).

### EKS Auto Scaling: Karpenter vs Cluster Autoscaler

**Cluster Autoscaler** (legacy): Watches for unschedulable pods and scales node groups up. Scales down nodes that are underutilized. Limitations: slow to react (minutes), works per-node-group, limited instance type flexibility.

**Karpenter** (modern, recommended): Purpose-built Kubernetes node autoscaler. Provisions nodes directly (bypassing Auto Scaling Groups). Responds in seconds, not minutes. Automatically selects the optimal instance type and size for pending pods. Consolidates workloads onto fewer nodes when utilization drops. Karpenter is the AWS-recommended autoscaler for EKS.

### EKS IAM Integration

**IRSA (IAM Roles for Service Accounts)**: Associates an IAM role with a Kubernetes service account. Pods using that service account automatically receive temporary AWS credentials scoped to the role. This provides pod-level IAM permissions instead of node-level — a critical security improvement.

**EKS Pod Identity** (newer): Simplified alternative to IRSA. Direct association between a Kubernetes service account and an IAM role without managing OIDC providers. Easier to set up but same security outcome.

Without IRSA or Pod Identity, all pods on a node share the node's IAM role — any pod can access any AWS resource the node role allows. This is a security anti-pattern.

### EKS Add-ons

EKS manages core Kubernetes components as add-ons:
- **VPC CNI**: Assigns VPC IP addresses to pods (awsvpc networking)
- **CoreDNS**: Cluster DNS for service discovery
- **kube-proxy**: Network proxy for Kubernetes services
- **EBS CSI driver**: Persistent volumes backed by EBS
- **EFS CSI driver**: Shared file system access from pods

### Networking: awsvpc Mode

For both ECS and EKS (with the VPC CNI), it is highly recommended to use the `awsvpc` network mode. This gives each task/pod its own ENI and its own private IP address from the VPC subnet. This allows you to use Security Groups at the task/pod level, providing fine-grained network security.

### ECS vs EKS vs Lambda

- **Use Lambda when**: Event-driven, short-lived (<15 min), unpredictable traffic, no infrastructure management desired
- **Use ECS/Fargate when**: Long-running processes, simple container orchestration, deep AWS integration preferred, no Kubernetes expertise needed
- **Use EKS when**: Kubernetes portability required, CRDs/operators/Helm needed, existing Kubernetes skills, multi-cloud strategy

### Common Interview Questions

**Q: What is the difference between Fargate and EC2 launch types in ECS?**
A: Fargate is serverless — you manage only the containers, and AWS manages the underlying infrastructure. You pay per task. EC2 launch type requires you to manage the EC2 instances in your cluster. EC2 gives more control over instance types and can be cheaper for sustained, high-utilization workloads, while Fargate is simpler and reduces operational overhead.

**Q: How do you handle secrets in ECS?**
A: Reference secrets from AWS Secrets Manager or SSM Parameter Store directly in the ECS Task Definition. ECS fetches the secrets and injects them as environment variables into the container at startup. This is more secure than hardcoding secrets or passing them in plain text.

**Q: What is Karpenter and why is it preferred over Cluster Autoscaler?**
A: Karpenter is a Kubernetes node autoscaler that provisions nodes directly (bypassing ASGs), responds in seconds, and automatically selects optimal instance types for pending pods. Cluster Autoscaler scales node groups (minutes to react, limited to pre-defined instance types). Karpenter also consolidates workloads to reduce cost and supports spot instances natively.

**Q: How do you give pods access to AWS services securely in EKS?**
A: Use IRSA (IAM Roles for Service Accounts) or EKS Pod Identity. Both associate an IAM role with a Kubernetes service account. Pods using that service account get temporary credentials scoped to the role. This provides pod-level permissions. Without IRSA/Pod Identity, all pods on a node share the node's IAM role — a major security risk.

---

## Amazon DynamoDB

### Overview

DynamoDB is a fully managed NoSQL database that delivers single-digit millisecond latency at any scale. It is a key-value and document database that supports both simple lookups and complex queries. Data is automatically replicated across three AZs in a region. DynamoDB is the default database choice for serverless architectures on AWS.

### How It Works Internally: Partitions

DynamoDB stores data in partitions. A partition is a unit of storage and throughput backed by SSD drives. Each partition can hold up to 10 GB of data and supports up to 3,000 RCU (read capacity units) and 1,000 WCU (write capacity units).

When you create a table, DynamoDB allocates partitions based on your provisioned capacity or expected throughput. The partition key is hashed to determine which partition stores each item. The number of partitions is:

- max(throughput-based partitions, size-based partitions)
- Throughput: (total RCU / 3000) + (total WCU / 1000), rounded up
- Size: total data / 10 GB, rounded up

Partitions are never deallocated, even if you reduce throughput. This is why "partition split" is a one-way operation — once DynamoDB splits a partition (because it exceeded 10 GB or throughput limits), the data is distributed across the new partitions and the old partition is retired.

### Adaptive Capacity

DynamoDB's adaptive capacity automatically redistributes throughput from underutilized partitions to heavily accessed (hot) partitions. This means that if one partition key is receiving disproportionate traffic, DynamoDB can allocate up to the entire table's provisioned capacity to that single partition (as long as the partition does not exceed 3,000 RCU or 1,000 WCU — the per-partition hard limit).

Adaptive capacity kicks in within 5-30 minutes. It does not solve burst issues or cases where a single partition key genuinely exceeds 3,000 RCU or 1,000 WCU.

### Hot Partitions

A hot partition occurs when a disproportionate amount of traffic is directed to a single partition key. Even with adaptive capacity, there is a hard limit per partition (3,000 RCU, 1,000 WCU). To avoid hot partitions:

- Use high-cardinality partition keys (user IDs, request IDs, not status codes)
- Add random suffixes to partition keys if access is truly uniform (write sharding)
- Use DynamoDB Accelerator (DAX) to cache reads and reduce partition pressure
- For time-series data, avoid using a timestamp as the partition key (leads to all writes hitting the latest partition)

### Global Secondary Indexes (GSI)

GSIs are essentially separate DynamoDB tables maintained by the system. When you write to the base table, DynamoDB asynchronously replicates the data to all GSIs. Important implications:

- GSIs have their own throughput capacity, separate from the base table.
- **GSI back-pressure throttling**: If a GSI does not have enough write capacity to keep up with the base table's writes, DynamoDB throttles writes on the base table, not the GSI. This is one of the most common causes of unexpected throttling. Your base table might have plenty of capacity, but if the GSI is under-provisioned, writes to the base table are throttled.
- GSIs only support eventually consistent reads (no strongly consistent reads on GSIs).
- GSI projections: KEYS_ONLY, INCLUDE (specific attributes), or ALL. Choose carefully — projecting ALL doubles your storage cost.

### Read Consistency

DynamoDB offers two read consistency models:

- **Eventually consistent reads** (default): May not reflect the results of a recently completed write. Reads are served from any of the three replicas. Cost: 0.5 RCU per 4 KB item.
- **Strongly consistent reads**: Returns the most up-to-date data, reflecting all writes that received a successful response prior to the read. Reads are served from the leader replica. Cost: 1 RCU per 4 KB item (2x the cost of eventually consistent).

Strongly consistent reads are not available for GSIs or Global Tables.

### Query vs Scan

**Query** retrieves items from a single partition (specified by partition key) and optionally filters by sort key conditions. It reads only the items that match and is efficient.

**Scan** reads every item in the entire table (or GSI), then applies a filter. The filter reduces the result set but does not reduce the data read — you pay for the full scan. Scans are expensive and slow on large tables. Use them only for administrative tasks or small tables.

### TTL (Time to Live)

TTL automatically deletes items after a specified timestamp. You designate an attribute as the TTL attribute, and items where that attribute's value is a Unix epoch timestamp in the past are eventually deleted.

Important: TTL deletions are not immediate. DynamoDB typically deletes items within 48 hours of their TTL expiration time. Until the item is actually deleted, it still appears in queries and scans (you should filter on the TTL attribute in your application). TTL deletions are free and do not consume WCU.

TTL deletions appear in DynamoDB Streams as system deletes, so you can react to them (e.g., archive to S3 before the item is gone).

### DynamoDB Streams

Streams capture a time-ordered sequence of item-level changes in a table. Each stream record contains the change (INSERT, MODIFY, REMOVE) along with the old and/or new item image (configurable: KEYS_ONLY, NEW_IMAGE, OLD_IMAGE, NEW_AND_OLD_IMAGES).

- Retention: 24 hours
- Consumers: Up to 2 Lambda functions per stream (recommended). You can have more, but this requires the KCL and increases read throttling risk.
- Ordering: Records are in order per partition key (same as Kinesis)
- Exactly-once: Each change appears exactly once in the stream
- Uses: Replication, event-driven processing, materialized views, cross-region replication (powers Global Tables)

### Transactions

DynamoDB supports ACID transactions across multiple items and tables within a single account and region. Transactions use a two-phase commit protocol (prepare + commit).

- TransactWriteItems: Up to 100 items per transaction (25 items prior to a limit increase)
- TransactGetItems: Up to 100 items per transaction
- Cost: 2x the WCU/RCU of non-transactional operations (because of the two-phase protocol)
- Size: Total transaction size cannot exceed 4 MB
- Transactions are all-or-nothing — if any condition fails, the entire transaction is rolled back
- Transactions conflict with each other: two transactions writing to the same item will cause one to fail with TransactionConflict

### On-Demand vs Provisioned Capacity

**Provisioned**: You specify RCU and WCU. DynamoDB pre-allocates capacity. You pay for what you provision whether you use it or not. Supports auto-scaling (adjusts capacity based on utilization, but has a reaction time of 5-15 minutes).

**On-demand**: DynamoDB automatically scales to any traffic level. You pay per request (read/write). No capacity planning needed. However:

- On-demand has a "ramp" period: if a table has had no traffic and suddenly receives a burst, DynamoDB can handle up to 2x the previous peak throughput instantly. Beyond that, it takes up to 30 minutes to fully adapt. For unpredictable burst traffic on a previously idle table, you may still get throttled.
- On-demand is generally more expensive per request than well-tuned provisioned capacity. Use provisioned with auto-scaling for predictable workloads.

### Condition Expressions (Optimistic Locking)

DynamoDB supports condition expressions on write operations (PutItem, UpdateItem, DeleteItem). The write succeeds only if the condition is true. This enables optimistic locking:

1. Read an item, note the version number
2. Write the updated item with a condition: `version = :expectedVersion`
3. If another process updated the item between your read and write, the condition fails and the write is rejected

This is cheaper and more scalable than pessimistic locking (which DynamoDB does not support natively).

### Backup and Recovery

- **On-demand backups**: Manual, full table backups. Retained until you delete them. No performance impact.
- **Point-in-Time Recovery (PITR)**: Continuous backups with second-level granularity. Restore to any point in the last 35 days. Must be explicitly enabled. Restores create a new table (not in-place).

### Global Tables

Global Tables provide multi-region, multi-active replication. Each region has a full read/write replica. Conflict resolution is last-writer-wins based on timestamps.

- Replication latency: typically under 1 second
- Strongly consistent reads are not available on Global Tables (only eventually consistent)
- All replicas must have the same table settings (billing mode, indexes)
- Writes to any region are replicated to all other regions
- Replicated writes do not consume WCU on the destination tables (they have their own replicated write capacity)

### Single-Table vs Multi-Table Design

Single-table design puts all entity types (users, orders, products) in one table, using overloaded partition and sort keys. This enables transactions across entity types and reduces the number of tables to manage. However, it is harder to understand, harder to evolve, and makes GSI design more complex.

Multi-table design is simpler, more intuitive, and aligns with each entity having its own access patterns. It is easier to manage capacity per table and to evolve schemas independently. The trade-off is that cross-entity transactions require DynamoDB transactions across tables (still supported, but you cannot do a Query that joins data across tables).

The AWS community has shifted toward multi-table design for most use cases, reserving single-table design for specific high-performance scenarios where minimizing round trips is critical.

### Limits and Quotas

- Maximum item size: 400 KB
- Maximum partition key size: 2,048 bytes
- Maximum sort key size: 1,024 bytes
- Maximum GSIs per table: 20
- Maximum LSIs per table: 5 (must be created at table creation time)
- Per-partition throughput: 3,000 RCU and 1,000 WCU
- Maximum transaction items: 100
- Maximum transaction size: 4 MB
- BatchWriteItem: up to 25 items, 16 MB total
- BatchGetItem: up to 100 items, 16 MB total
- Query/Scan result: up to 1 MB per call (use pagination)
- DynamoDB Streams retention: 24 hours

### Pricing Model

Provisioned: per RCU-hour and WCU-hour. On-demand: per read request unit ($0.25 per million) and write request unit ($1.25 per million). Storage: $0.25 per GB/month. Free tier: 25 GB storage, 25 WCU, 25 RCU.

### Edge Cases and Gotchas

- GSI throttling propagates to the base table. Always provision GSI capacity to handle the table's write throughput.
- TTL deletes are not immediate — up to 48 hours delay. Filter expired items in your application.
- Strongly consistent reads are 2x the cost and not available on GSIs or Global Tables.
- The 400 KB item size limit includes attribute names. Use short attribute names for large items.
- Scan consumes RCU for the entire table, not just matching items. Never scan large tables in production.
- On-demand tables still have a ramp-up limit. A cold table with a sudden spike may be throttled for up to 30 minutes.
- LSIs (Local Secondary Indexes) share the base table's partition and have a 10 GB partition limit per partition key. This can cause writes to fail if a partition key's data exceeds 10 GB.
- PITR restores to a new table — you cannot restore in place. You must switch your application to the new table.
- Global Tables use last-writer-wins. Concurrent writes to the same item in different regions will result in one write being silently overwritten.
- DynamoDB Streams are limited to 2 concurrent Lambda consumers without KCL. More consumers risk throttling.

### Common Interview Questions

**Q: Explain DynamoDB adaptive capacity.**
A: Adaptive capacity redistributes throughput across partitions based on actual access patterns. When one partition (hot partition) receives more traffic than its fair share, DynamoDB borrows unused capacity from other partitions and allocates it to the hot one. This can provide up to the table's full provisioned capacity to a single partition, as long as the partition stays within the per-partition hard limit of 3,000 RCU / 1,000 WCU. Adaptive capacity takes 5-30 minutes to fully kick in.

**Q: How would you handle a hot partition?**
A: First, redesign the partition key for higher cardinality. If that is not possible, use write sharding (append a random suffix to the partition key and scatter-gather on reads). For read-heavy hot keys, use DAX as a write-through cache. Switch to on-demand capacity mode to handle burst traffic. As a last resort, consider splitting the data into multiple tables.

**Q: What is the difference between DynamoDB Streams and Kinesis Data Streams for DynamoDB?**
A: DynamoDB Streams is the native change data capture mechanism with 24-hour retention and up to 2 Lambda consumers. Kinesis Data Streams for DynamoDB is a newer option that sends change data to a Kinesis stream with up to 365-day retention, unlimited consumers (via enhanced fan-out), and higher throughput. Use Kinesis integration when you need longer retention, more consumers, or want to use existing Kinesis consumers.

### Interactions with Other Services

- **Lambda**: DynamoDB Streams trigger Lambda functions for event-driven processing.
- **S3**: Export DynamoDB table to S3 for analytics (DynamoDB Export to S3). Import from S3 for bulk loads.
- **EventBridge Pipes**: Pipes can read from DynamoDB Streams for routing change events.
- **Step Functions**: Step Functions can call DynamoDB directly (GetItem, PutItem, etc.).
- **CloudWatch**: DynamoDB publishes metrics for throttling, capacity utilization, latency, etc.
- **DAX**: DynamoDB Accelerator is an in-memory cache for DynamoDB, providing microsecond read latency.

---

## Caching: ElastiCache, DAX, and MemoryDB

### Overview

Caching is one of the most impactful performance optimizations in cloud architecture. AWS offers three managed caching services, each for different use cases. Understanding when to use which is a common interview question.

### Amazon ElastiCache

ElastiCache is a managed in-memory data store supporting two engines: Redis and Memcached.

**ElastiCache for Redis**:
- Rich data structures: strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial indexes
- Persistence: Optional snapshotting (RDB) and append-only file (AOF) for durability. Data can survive a restart, but ElastiCache Redis is not designed as a primary database — persistence is for recovery, not durability.
- Replication: Primary-replica with automatic failover (Multi-AZ). Up to 5 replicas per shard.
- Clustering: Redis Cluster mode for horizontal scaling. Data is partitioned across shards. Each shard has a primary + replicas. Up to 500 nodes per cluster.
- Pub/sub: Built-in messaging for real-time features
- Lua scripting: Server-side logic
- Transactions: MULTI/EXEC for atomic operations

**ElastiCache for Memcached**:
- Simple key-value store. No data structures beyond strings.
- No persistence — data is lost on restart
- No replication — no automatic failover
- Multi-threaded — can utilize multiple CPU cores (Redis is single-threaded per shard)
- Simpler and slightly cheaper than Redis

**Decision**: Use Redis for almost everything. Use Memcached only when you need a simple, ephemeral cache with multi-threaded performance and do not need persistence, replication, or rich data structures. New projects should default to Redis.

### ElastiCache Serverless

ElastiCache Serverless (Redis and Memcached) automatically scales cache capacity based on traffic. No nodes to manage, no shards to configure. It scales to handle millions of requests per second and scales down during low traffic.

Pricing is per ECPU (ElastiCache Compute Unit) consumed + per GB of data stored per hour. More expensive per unit than provisioned ElastiCache at high utilization, but eliminates capacity planning and over-provisioning waste.

Use ElastiCache Serverless for: unpredictable traffic patterns, new applications where you do not know the cache size yet, or when operational simplicity is more important than cost optimization.

### Common Caching Patterns

**Cache-aside (lazy loading)**: Application checks the cache first. If miss, reads from database, writes result to cache, returns. Pros: only requested data is cached. Cons: cache miss penalty (extra round trip), stale data possible.

**Write-through**: Application writes to the cache and the database simultaneously. Pros: cache is always up to date. Cons: write penalty (write to two places), caches data that may never be read.

**Write-behind (write-back)**: Application writes to cache only. Cache asynchronously writes to database. Pros: fastest writes. Cons: risk of data loss if cache fails before writing to database.

**TTL (Time-to-Live)**: Set an expiration time on cached items. After TTL expires, the item is evicted. This bounds staleness — you accept that data may be up to TTL seconds old. TTL is the simplest staleness control and works with all patterns.

**Cache invalidation**: Explicitly delete or update cache entries when the underlying data changes. This is the hardest part of caching — "there are only two hard things in computer science: cache invalidation and naming things." Use DynamoDB Streams or EventBridge to trigger cache invalidation when data changes.

### DynamoDB Accelerator (DAX)

DAX is a fully managed, in-memory cache specifically for DynamoDB. It sits between your application and DynamoDB and is API-compatible — you change the SDK endpoint, and reads/writes go through DAX transparently.

How it works:
- **Read-through cache**: If the item is in DAX, it returns immediately (microseconds). If not, DAX reads from DynamoDB, caches the result, and returns it.
- **Write-through cache**: Writes go to DAX and DynamoDB simultaneously. DAX updates its cache immediately.
- **Item cache**: Caches individual GetItem and BatchGetItem results by primary key.
- **Query cache**: Caches Query and Scan result sets by the exact query parameters.

When to use DAX:
- Read-heavy DynamoDB workloads with repeated reads of the same items
- Applications that need microsecond read latency (DAX: ~200 microseconds vs DynamoDB: ~1-5 milliseconds)
- Reducing DynamoDB RCU consumption (and cost) for hot items
- Use cases where eventually consistent reads are acceptable (DAX does not support strongly consistent reads)

When NOT to use DAX:
- Write-heavy workloads (DAX adds write latency, and writes go to both DAX and DynamoDB)
- Applications that require strongly consistent reads (DAX is eventually consistent only)
- Applications that primarily use Scan (caching scans is usually not useful)
- When ElastiCache would be better (complex data structures, cross-service caching, pub/sub)

DAX cluster runs inside your VPC. Minimum 3 nodes for production (multi-AZ). Pricing: per node-hour (similar to ElastiCache pricing, based on node type).

### DAX vs ElastiCache for DynamoDB Caching

**DAX**: DynamoDB-specific, API-compatible (transparent to application), microsecond latency, only caches DynamoDB data, write-through automatically, simpler to integrate.

**ElastiCache Redis**: General-purpose, requires application-level cache logic (cache-aside pattern), sub-millisecond latency, caches data from any source (DynamoDB, Aurora, APIs, computed results), richer data structures and query capabilities.

Decision: Use DAX when you only need to cache DynamoDB reads and want transparent, zero-code caching. Use ElastiCache when you need to cache data from multiple sources, need rich data structures (leaderboards with sorted sets, session stores with hashes), or need features like pub/sub.

### Amazon MemoryDB for Redis

MemoryDB is a durable, Redis-compatible database (not just a cache). Unlike ElastiCache Redis (where persistence is optional and not guaranteed), MemoryDB stores data durably with a multi-AZ transaction log. Data survives node failures, restarts, and full cluster replacements.

Key differences from ElastiCache Redis:
- **Durability**: MemoryDB provides 11 nines of durability (same as S3). ElastiCache may lose data on failover.
- **Primary database**: MemoryDB can be your primary database for use cases that fit the Redis data model. ElastiCache is a cache layer in front of another database.
- **Write latency**: MemoryDB writes are slightly slower because they write to the transaction log before acknowledging. ElastiCache acknowledges writes immediately.
- **Read latency**: Both deliver microsecond read latency.
- **Cost**: MemoryDB is more expensive than ElastiCache (you pay for the durability).

When to use MemoryDB:
- You want Redis as your primary database (session stores, user profiles, leaderboards, real-time inventory)
- You need the Redis data model with database-level durability
- You want to eliminate the separate database + cache architecture for specific use cases

When to use ElastiCache:
- You need a cache layer in front of DynamoDB, Aurora, or another primary database
- Durability is not required (data can be regenerated from the primary store)
- You want the lowest possible cost for caching

### Caching Limits and Quotas

- ElastiCache Redis: up to 500 nodes per cluster, 340 TB total data per cluster
- ElastiCache Memcached: up to 300 nodes per cluster
- DAX: up to 11 nodes per cluster, item cache max 10 GB per node, query cache max 10 GB per node
- MemoryDB: up to 500 nodes per cluster
- ElastiCache max item size: 512 MB (Redis), 1 MB (Memcached)

### Caching Edge Cases and Gotchas

- Cache stampede: When a popular cached item expires, hundreds of concurrent requests all miss the cache and hit the database simultaneously. Mitigate with: TTL jitter (randomize expiry times), probabilistic early expiration, or locking (only one request refreshes the cache).
- ElastiCache Redis is single-threaded per shard for data operations. A single slow command (KEYS *, large SORT) blocks all other operations on that shard. Never use KEYS in production — use SCAN instead.
- DAX is eventually consistent only. If you write to DynamoDB directly (bypassing DAX), the DAX cache may serve stale data until TTL expires. Always write through DAX.
- DAX cluster runs in your VPC. Lambda functions must be VPC-attached to use DAX. This means you need NAT Gateway or VPC endpoints for Lambda to reach other services.
- ElastiCache does not support IAM authentication by default. Use Redis AUTH tokens or IAM-based authentication (supported for Redis 7+). Without authentication, anyone with network access to the cluster can read/write data.
- MemoryDB is not a drop-in replacement for ElastiCache. Some Redis commands behave differently, and the write latency increase may affect applications that depend on sub-millisecond writes.
- Cluster mode disabled (single-shard) Redis has a data limit of ~50 GB. For larger datasets, you must use cluster mode with sharding.

### Caching Interview Questions

**Q: When would you use ElastiCache vs DAX vs MemoryDB?**
A: Use DAX for transparent, zero-code caching of DynamoDB reads with microsecond latency. Use ElastiCache Redis as a general-purpose cache for any data source when you need rich data structures (sorted sets for leaderboards, pub/sub for real-time). Use MemoryDB when you want Redis as your primary database with durability guarantees, eliminating the need for a separate database+cache architecture.

**Q: How do you handle cache invalidation?**
A: For DynamoDB, use DAX (automatic write-through) or DynamoDB Streams triggering a Lambda that invalidates specific ElastiCache keys. For Aurora, use event-driven invalidation (application publishes to SNS/EventBridge on write, consumer invalidates cache). Always set TTL as a safety net — even with active invalidation, TTL ensures stale data eventually expires. For cache-aside patterns, prefer short TTLs (seconds to minutes) over complex invalidation logic.

**Q: Explain the cache-aside pattern and its trade-offs.**
A: Application checks cache first. On miss, reads from database, writes to cache, returns data. Pros: only caches data that is actually requested (no wasted memory), naturally adapts to access patterns. Cons: first request for any item always hits the database (cold start), stale data if database is updated without invalidating cache. Mitigate staleness with short TTLs. This is the most common caching pattern and the default choice when you do not have a reason to use write-through.

**Q: How do you prevent cache stampede?**
A: Add jitter to TTL values (e.g., TTL = 300 + random(0, 60) seconds) so items do not expire simultaneously. Use probabilistic early expiration — each request has a small chance of refreshing the cache before TTL, spreading the refresh load. For critical hot keys, use a lock/mutex — only one request refreshes the cache while others wait or serve the stale value. Some applications use a "stale-while-revalidate" pattern where the stale value is served while a background task refreshes the cache.

---

## Amazon S3

### Overview

S3 is an object storage service with virtually unlimited capacity, 99.999999999% (11 nines) durability, and 99.99% availability (standard tier). Since December 2020, S3 provides strong read-after-write consistency for all operations — if you PUT an object and immediately GET it, you are guaranteed to receive the latest version. This was a major change from the previous eventual consistency model for overwrite PUTs and DELETE operations.

### How It Works Internally

S3 stores objects across a minimum of three AZs (for standard storage class). Objects are identified by a bucket name and key (the full path). S3 is a flat namespace — there are no real directories. The "/" in keys is just a convention; prefixes are not folders.

S3 automatically partitions data by key prefix for parallel access. AWS has continuously improved this so that individual prefixes now support 3,500 PUT/COPY/POST/DELETE and 5,500 GET/HEAD requests per second. These limits are per prefix, and you can achieve higher throughput by parallelizing across multiple prefixes.

### Strong Consistency

Since December 2020, S3 is strongly consistent for all read and write operations. Specifically:
- Read-after-write consistency for PUT of new objects
- Read-after-write consistency for overwrite PUT and DELETE
- List operations reflect completed writes

This means you no longer need to worry about stale reads. If a PUT returns 200 OK, subsequent GETs will return the new version. There is no additional cost or latency for strong consistency — it is the default behavior.

### Prefix Throughput and 503 Slow Down

Each prefix supports 3,500 writes/s and 5,500 reads/s. If you exceed these limits, S3 returns 503 Slow Down errors. To avoid this:

- Distribute objects across multiple prefixes
- Avoid sequential key names (timestamps, sequential IDs) that concentrate traffic on a single prefix
- Use randomized prefixes or hash-based prefixes for high-throughput workloads
- S3 now automatically repartitions hot prefixes, but sudden spikes can still cause 503s before repartitioning completes

### Object Lock

Object Lock prevents objects from being deleted or overwritten for a specified retention period. Two modes:

- **Governance mode**: Users with the s3:BypassGovernanceRetention permission can override the lock. Use for organizational policies that might need exceptions.
- **Compliance mode**: No one can override the lock, not even the root account. Use for regulatory compliance (WORM storage). Once set, the retention period cannot be shortened.

Object Lock requires versioning to be enabled on the bucket.

### Presigned URLs

Presigned URLs grant temporary access to a specific S3 object without requiring the requester to have AWS credentials. They include the signer's credentials, an expiration time, and a signature.

Gotcha: A presigned URL remains valid until its expiration time, even if the underlying IAM credentials or role are revoked. There is no way to invalidate a presigned URL before its expiration except by deleting the object itself. For sensitive objects, use short expiration times.

When presigned URLs are generated with temporary credentials (IAM role or STS), the URL expires when the temporary credentials expire, even if the URL's specified expiration is later.

### Event Notifications

S3 can send event notifications (ObjectCreated, ObjectRemoved, etc.) to Lambda, SQS, or SNS. There is a limitation: you can only have one destination per prefix per event type using native S3 event notifications. If you need multiple consumers for the same event, use Amazon EventBridge for S3 (must be enabled per bucket), which sends all S3 events to EventBridge where you can create multiple rules with different targets.

### Cross-Region Replication (CRR)

CRR copies objects from a source bucket to a destination bucket in a different region. Important caveats:

- Existing objects are not replicated when you enable CRR. Only new objects written after CRR is configured are replicated. You can use S3 Batch Replication to copy existing objects.
- Deletes are not replicated by default. If you delete an object in the source, it is not deleted in the destination. You can enable delete marker replication, but actual permanent deletes of versioned objects are never replicated (to prevent accidental data loss).
- Both buckets must have versioning enabled.
- Replication is asynchronous — most objects replicate within 15 minutes, but there is no SLA for replication time. S3 Replication Time Control (RTC) provides a 15-minute SLA at additional cost.

### Multipart Upload

For objects larger than 100 MB (recommended) or up to 5 TB (required), use multipart upload. Multipart upload splits the object into parts, uploads them in parallel, and assembles them. If a part fails, only that part needs to be re-uploaded.

Gotcha: If you initiate a multipart upload and never complete or abort it, the parts remain in S3 and you are charged for storage. Create a lifecycle policy to automatically abort incomplete multipart uploads after a specified number of days.

### Storage Classes

- **S3 Standard**: Default, low latency, high throughput
- **S3 Intelligent-Tiering**: Automatically moves objects between access tiers based on access patterns. No retrieval fees. Small monitoring fee per object. Best for unpredictable access patterns.
- **S3 Standard-IA (Infrequent Access)**: Lower storage cost, but per-GB retrieval fee. Minimum 128 KB charge per object, 30-day minimum storage duration.
- **S3 One Zone-IA**: Same as IA but stored in one AZ. 20% cheaper than Standard-IA. Use for reproducible data.
- **S3 Glacier Instant Retrieval**: Millisecond retrieval, lowest cost for data accessed quarterly.
- **S3 Glacier Flexible Retrieval**: Minutes to hours retrieval (Expedited: 1-5 min, Standard: 3-5 hr, Bulk: 5-12 hr).
- **S3 Glacier Deep Archive**: Cheapest storage ($0.00099/GB/month). 12-48 hour retrieval. Minimum 180-day storage.

### Encryption

S3 supports four encryption options:

- **SSE-S3** (default since Jan 2023): S3 manages the keys. AES-256. No configuration needed. No additional cost. Sufficient for most use cases.
- **SSE-KMS**: You choose a KMS key (AWS managed or customer managed). Provides audit trail via CloudTrail (who decrypted what). Subject to KMS rate limits. Use when you need key access control or audit.
- **SSE-C**: You provide the encryption key with each request. S3 encrypts/decrypts but does not store the key. You manage key storage. Rare use case.
- **Client-side encryption**: Encrypt before upload, decrypt after download. S3 never sees plaintext. Use when you need end-to-end encryption that you fully control.

Since January 2023, all new objects are encrypted with SSE-S3 by default. You can override the default to SSE-KMS at the bucket level.

### Versioning

Versioning keeps multiple variants of an object in the same bucket. Every overwrite or delete creates a new version instead of replacing the object.

Key mechanics:
- Each version has a unique version ID
- Deleting a versioned object creates a delete marker (a placeholder), not an actual deletion. The data is still there.
- To permanently delete a version, you must delete the specific version ID. This is intentional — prevents accidental data loss.
- MFA Delete: Require MFA to delete versions or change versioning status. Additional protection against malicious deletion.
- Versioning cannot be disabled once enabled — only suspended. Suspended versioning stops creating new versions but preserves existing ones.
- Versioned objects still cost money. Old versions accumulate storage costs silently. Use lifecycle policies to expire old versions.

### Bucket Policies vs ACLs

**Bucket policies** (recommended): JSON-based resource policies attached to the bucket. Full control over who can access what, with conditions. Use for all access control.

**ACLs** (legacy): Coarse-grained access control (READ, WRITE, FULL_CONTROL) per object or bucket. AWS recommends disabling ACLs entirely using the **BucketOwnerEnforced** setting (now the default for new buckets). With BucketOwnerEnforced, the bucket owner owns all objects and ACLs are ignored — only bucket policies and IAM policies control access.

### S3 Access Points

Access Points simplify managing access to shared buckets. Each access point has its own DNS name, IAM policy, and optional VPC restriction. Instead of one complex bucket policy with dozens of conditions, create an access point per application or team with a simple, focused policy.

VPC-restricted access points only allow access from a specific VPC — an additional security layer.

### S3 Batch Operations

Batch Operations performs bulk actions on billions of objects with a single request. Operations include: copy, invoke Lambda (per object), replace tags, change storage class, restore from Glacier, delete.

Use cases: migrate objects between buckets, change encryption settings on existing objects, tag objects for compliance, invoke Lambda to transform every object in a bucket.

You provide a manifest (list of objects from S3 Inventory or a CSV), specify the operation, and S3 executes it at scale. Progress is tracked with completion reports.

### S3 Transfer Acceleration

Transfer Acceleration speeds up long-distance uploads to S3 by routing data through CloudFront edge locations. Instead of uploading directly to the S3 region (which may be far away), the upload hits the nearest edge location and is transferred to S3 over AWS's optimized backbone network.

Use when: users upload large files from distant geographic locations. Improvement varies — typically 50-500% faster for cross-continent uploads. No improvement for uploads close to the S3 region.

Pricing: $0.04-0.08/GB on top of standard transfer charges. Only charged when acceleration actually improves speed.

### S3 Select

S3 Select lets you use SQL expressions to filter the contents of an S3 object before downloading it. Instead of downloading a 1 GB CSV file to find 10 rows, you can query the file in-place and return only the matching rows. Supports CSV, JSON, and Parquet. Saves data transfer and processing time.

### Limits and Quotas

- Maximum object size: 5 TB
- Maximum PUT in single operation: 5 GB (use multipart for larger)
- Maximum parts in multipart upload: 10,000
- Prefix throughput: 3,500 PUT/5,500 GET per second per prefix
- Maximum buckets per account: 100 (soft limit, can be increased to 1,000)
- Maximum object key length: 1,024 bytes
- No limit on number of objects per bucket

### Pricing Model

S3 charges for storage (per GB/month), requests (per 1,000 requests), data transfer out (per GB), and retrieval (for IA/Glacier classes). S3 Standard storage costs approximately $0.023/GB/month in us-east-1. PUT requests: $0.005/1,000. GET requests: $0.0004/1,000. Data transfer to the internet: $0.09/GB (first 10 TB).

### Edge Cases and Gotchas

- S3 is strongly consistent since December 2020. Any remaining documentation suggesting eventual consistency is outdated.
- 503 Slow Down errors indicate prefix throttling. Distribute keys across prefixes and implement retries with backoff.
- Presigned URLs cannot be revoked. Keep expiration times short for sensitive data.
- CRR does not copy existing objects or replicate permanent deletes.
- Event notifications have a one-destination-per-prefix-per-event-type limit. Use EventBridge integration for complex routing.
- Lifecycle policies evaluate at midnight UTC and may take up to 48 hours to execute.
- S3 versioning cannot be disabled once enabled — only suspended. Suspended versioning stops creating new versions but preserves existing ones.
- Deleting a versioned object creates a delete marker, not an actual deletion. The data is still there (and still costs money) until you delete the specific version.
- S3 storage class transitions via lifecycle policies are one-way (Standard -> IA -> Glacier). You cannot transition back automatically.
- The minimum storage duration charges for IA (30 days) and Glacier (90/180 days) mean that storing short-lived objects in these tiers can be more expensive than Standard.

### Common Interview Questions

**Q: Is S3 eventually consistent or strongly consistent?**
A: S3 has been strongly consistent for all operations since December 2020. Read-after-write consistency for both new objects and overwrite PUTs, and list consistency for all operations. There is no additional cost or configuration needed.

**Q: How do you handle high-throughput access to S3?**
A: S3 supports 3,500 PUT/5,500 GET per prefix per second. For higher throughput, distribute objects across multiple prefixes, use random or hash-based key prefixes, enable S3 Transfer Acceleration for cross-region uploads, and use multipart upload for large objects. S3 automatically repartitions hot prefixes, but sudden spikes may cause temporary 503s.

**Q: What is the difference between S3 Glacier modes?**
A: Glacier Instant Retrieval provides millisecond access for data accessed once per quarter. Glacier Flexible Retrieval provides retrieval in minutes to hours (3 retrieval tiers). Glacier Deep Archive is the cheapest at $0.00099/GB/month with 12-48 hour retrieval. Choose based on how quickly you need the data and how rarely you access it.

### Interactions with Other Services

- **Lambda**: S3 events trigger Lambda for file processing. One of the most common serverless patterns.
- **CloudFront**: CloudFront distributes S3 content globally. Use OAC (Origin Access Control) to restrict direct S3 access.
- **Athena**: Query S3 data directly with SQL. No data loading needed.
- **EventBridge**: S3 events can be routed through EventBridge for complex event processing.
- **DynamoDB**: Common pattern: store large objects in S3, store metadata and S3 keys in DynamoDB.
- **Glacier**: S3 lifecycle policies can automatically transition objects to Glacier storage classes.

---

## Amazon Aurora PostgreSQL

### Overview

Aurora is a cloud-native relational database compatible with MySQL and PostgreSQL. It separates compute and storage: the compute layer (database instances) runs the query engine, while the storage layer is a shared, distributed, fault-tolerant storage system. Aurora provides up to 5x the throughput of standard PostgreSQL on the same hardware.

### How It Works Internally: Storage

Aurora's storage is a distributed, log-structured system spread across three AZs, with two copies in each AZ, for a total of six copies of your data. It uses a quorum protocol:

- **Write quorum**: 4 out of 6 copies must acknowledge a write for it to succeed
- **Read quorum**: 3 out of 6 copies must respond for a read to succeed (only used during recovery)

This means Aurora can lose an entire AZ (2 copies) and still write. It can lose an AZ plus one additional copy and still read.

Storage is auto-scaling: it starts at 10 GB and grows automatically in 10 GB increments up to 128 TB. However, storage never shrinks — even if you delete data, the allocated storage does not decrease. The space is reused internally but you continue to pay for the high-water mark.

The key innovation is that Aurora replicates only the redo log, not the data pages. This dramatically reduces network traffic compared to standard PostgreSQL replication (6x less I/O).

### Read Replicas

Aurora supports up to 15 read replicas within the same region. Read replicas share the same storage layer (they do not have their own copy of the data), so creating a replica is fast and does not double storage costs.

Replica lag is typically less than 10 milliseconds (often sub-millisecond) because replicas read from the same shared storage. Compare this to standard PostgreSQL replication which can have seconds to minutes of lag.

### Failover

If the primary instance fails, Aurora automatically promotes a read replica to primary. Failover typically takes approximately 30 seconds. If there are no read replicas, Aurora creates a new primary instance (which takes longer).

Aurora provides a cluster endpoint (points to the primary) and a reader endpoint (load balances across read replicas). Applications should use these endpoints rather than connecting to specific instances.

### Aurora Serverless v2

Aurora Serverless v2 scales compute capacity automatically based on demand. Capacity is measured in ACUs (Aurora Capacity Units), where 1 ACU is approximately 2 GB of memory.

Important limitations:
- Minimum capacity is 0.5 ACU — Aurora Serverless v2 does not scale to zero. You always pay for at least 0.5 ACU.
- Scaling is faster than v1 (seconds vs minutes) but not instantaneous
- Supports all Aurora features (read replicas, Global Database, etc.) unlike v1 which had many restrictions

For true scale-to-zero, Aurora Serverless v1 is still available but is an older product with significant limitations (no read replicas, no Global Database, limited availability).

### RDS Proxy for Lambda

Lambda functions open database connections on every cold start. With hundreds of concurrent Lambda invocations, this can exhaust the database's connection limit. RDS Proxy sits between Lambda and Aurora, pooling and reusing connections.

RDS Proxy maintains a warm pool of connections, multiplexes Lambda connections onto the pool, and handles connection failures transparently. It adds approximately 5ms of latency but prevents connection storms.

### Connection Limits

Aurora connection limits depend on instance size. A db.r5.large instance supports approximately 1,000 connections. Running out of connections causes new connections to be refused — a common failure mode with Lambda.

### Limits and Quotas

- Maximum storage: 128 TB
- Maximum read replicas: 15 (within a region)
- Minimum Serverless v2 capacity: 0.5 ACU
- Maximum Serverless v2 capacity: 256 ACU
- Failover time: approximately 30 seconds
- Replica lag: typically less than 10ms
- Maximum connections: depends on instance size (approximately 1,000 for r5.large)
- Backup retention: up to 35 days
- Global Database replication: typically under 1 second cross-region

### Pricing Model

Aurora charges per instance-hour (compute), per GB-month (storage), and per million I/O requests (for Aurora Standard) or per GB-month with no I/O charges (for Aurora I/O-Optimized). Serverless v2 charges per ACU-hour. Data transfer between AZs for replication is included. Backups beyond the free retention period are charged.

### Edge Cases and Gotchas

- Storage never shrinks. If you bulk-load 1 TB and then delete it, you still pay for 1 TB of storage until you recreate the cluster.
- Failover takes approximately 30 seconds — this is not zero-downtime. Applications must handle connection drops and reconnect.
- Aurora Serverless v2 does not scale to zero. For infrequently used databases, you still pay for 0.5 ACU (approximately $43/month).
- Reader endpoint uses DNS round-robin, not intelligent load balancing. Long-lived connections may unevenly distribute across replicas.
- Aurora PostgreSQL compatibility is not 100% — some extensions and features may not be available or behave differently.
- Cloning is fast (copy-on-write) but the clone shares the storage billing with the source until pages diverge.
- Backtrack (point-in-time rewind) is only available for Aurora MySQL, not PostgreSQL.

### Common Interview Questions

**Q: How does Aurora achieve higher throughput than standard PostgreSQL?**
A: Aurora separates compute from storage and replicates only the redo log (not full data pages) to the storage layer. This reduces network I/O by 6x. The distributed storage system handles replication, recovery, and caching independently of the compute instances. This architecture eliminates checkpoint stalls and reduces write latency.

**Q: How does Aurora handle failures?**
A: The storage layer survives the loss of an entire AZ (4/6 quorum for writes, 3/6 for reads). If the primary compute instance fails, Aurora promotes a read replica in approximately 30 seconds. If no replicas exist, Aurora creates a new instance. The shared storage layer means no data is lost during failover because replicas read from the same storage.

### Interactions with Other Services

- **Lambda**: Lambda connects to Aurora via RDS Proxy for connection pooling.
- **S3**: Aurora can export snapshots to S3 and import data from S3.
- **Secrets Manager**: Store and rotate database credentials automatically.
- **CloudWatch**: Aurora publishes detailed performance metrics, including Performance Insights.

---

## Amazon DocumentDB (with MongoDB Compatibility)

### Overview

DocumentDB is a fully managed document database service designed to be compatible with MongoDB. It stores data as JSON-like documents and supports MongoDB APIs, drivers, and tools. DocumentDB uses the same shared storage architecture as Aurora — decoupled compute and storage with data replicated 6 ways across 3 AZs. It is not a MongoDB fork; it is a purpose-built database that implements the MongoDB wire protocol.

### How It Works Internally

DocumentDB's architecture is very similar to Aurora. The compute layer (instances) handles query processing, and the storage layer is a distributed, fault-tolerant, self-healing system that replicates data across three AZs.

- Storage auto-grows in 10 GB increments up to 128 TB
- Storage never shrinks (same as Aurora — high-water mark billing)
- Uses a quorum-based replication model (same 4/6 write, 3/6 read as Aurora)
- Write operations go to the primary instance, reads can be distributed across up to 15 read replicas
- Replica lag is typically under 10 milliseconds

### MongoDB Compatibility

DocumentDB implements the MongoDB 3.6, 4.0, and 5.0 APIs. Most common MongoDB operations work: CRUD, aggregation pipeline, indexes (including compound, multikey, text, geospatial, partial, sparse, TTL, unique, wildcard), change streams, and transactions.

However, DocumentDB is not 100% MongoDB-compatible. Notable differences and missing features:

- No support for the `$where` operator (server-side JavaScript execution)
- No support for `$accumulator` or `$function` in aggregation (no user-defined JavaScript)
- Map-reduce is not supported (use aggregation pipeline instead)
- Some MongoDB shell commands behave differently or are not available
- Client-side field-level encryption is not supported
- Atlas Search equivalent does not exist (use OpenSearch for full-text search)
- Capped collections are not supported
- Change streams use a polling model with a change stream token, not MongoDB's oplog-based approach

The general rule: if your application uses standard CRUD, aggregation pipelines, and indexes, DocumentDB works well. If it relies on server-side JavaScript, advanced MongoDB features, or exact MongoDB internals, test thoroughly before migrating.

### Elastic Clusters

DocumentDB Elastic Clusters provide horizontal scaling through sharding. Unlike the standard instance-based clusters (which scale vertically by increasing instance size), Elastic Clusters distribute data across shards based on a shard key.

- Scales to millions of reads/writes per second and petabytes of storage
- You choose a shard key (similar to a MongoDB shard key)
- Supports up to 32 shards, each with up to 64 vCPUs
- Automatic shard management and data distribution
- Compatible with MongoDB sharding APIs

Use Elastic Clusters when your workload exceeds what a single-writer, multi-reader cluster can handle, or when your data exceeds 128 TB.

### Change Streams

DocumentDB supports change streams, which provide a real-time stream of document-level changes (insert, update, replace, delete). Change streams use a poll-based model — your application polls for changes using a change stream cursor and a resume token.

Change streams are commonly used for:
- Event-driven processing: trigger a Lambda function when a document changes
- Data replication: sync changes to another data store (OpenSearch, S3, DynamoDB)
- Audit logging: capture every change to a collection

To trigger Lambda from DocumentDB changes, you configure DocumentDB as a Lambda event source. Lambda polls the change stream and invokes your function with batches of change events. This is similar to DynamoDB Streams + Lambda.

Change stream events are retained for up to 7 days (configurable via the `change_stream_log_retention_duration` parameter). If your consumer falls behind by more than the retention window, it loses its position and must start from the current point.

### Indexing

DocumentDB supports the same index types as MongoDB:

- **Single field** and **compound indexes**: Standard B-tree indexes
- **Multikey indexes**: For array fields, indexing each element
- **Text indexes**: Basic full-text search (not as capable as MongoDB Atlas Search or OpenSearch)
- **Geospatial indexes** (2dsphere): For location queries
- **TTL indexes**: Automatically delete documents after a specified time
- **Partial indexes**: Index only documents matching a filter expression
- **Sparse indexes**: Index only documents where the indexed field exists
- **Wildcard indexes**: Index all fields or all fields matching a pattern

Missing: DocumentDB does not support hashed indexes (used for MongoDB hash-based sharding in standard clusters) or Atlas Search indexes. For advanced search, pair DocumentDB with Amazon OpenSearch.

### Transactions

DocumentDB supports multi-document ACID transactions (introduced with MongoDB 4.0 compatibility). Transactions can span multiple documents, collections, and statements within a single cluster.

- Maximum transaction duration: 1 minute
- Maximum transaction size: 16 MB
- Transactions use snapshot isolation — reads within a transaction see a consistent snapshot
- Write conflicts cause the later transaction to fail and retry

### Global Clusters

DocumentDB Global Clusters provide cross-region replication with read replicas in up to 5 secondary regions. Replication latency is typically under 1 second. In a disaster recovery scenario, you can promote a secondary region to a standalone cluster in under 1 minute.

Unlike DynamoDB Global Tables (which are multi-active), DocumentDB Global Clusters are single-writer — all writes go to the primary region. Secondary regions handle reads only. This avoids write conflict issues but means writes have the latency of the primary region.

### DocumentDB vs DynamoDB

This is a common interview question. The answer depends on the data model and access patterns:

**Use DocumentDB when:**
- Your data is naturally document-shaped (nested, variable schema)
- You need ad-hoc queries and complex aggregation pipelines
- You are migrating from MongoDB and want compatibility
- You need multi-document transactions with complex conditions
- Your query patterns are not fully known at design time (flexible querying via indexes)

**Use DynamoDB when:**
- You need single-digit millisecond latency at any scale
- Your access patterns are known and can be modeled with partition/sort keys
- You need truly serverless with on-demand scaling and zero management
- You need multi-region active-active replication (Global Tables)
- You need unlimited throughput scaling

**Use both when:**
- DocumentDB for complex queries and flexible document storage
- DynamoDB for high-speed lookups and event-driven workflows
- S3 for large objects referenced by either database

### DocumentDB vs MongoDB on EC2 vs Amazon MemoryDB

**DocumentDB**: Managed, Aurora-style storage, MongoDB wire protocol compatibility. Best for teams that want managed MongoDB-like experience without operational overhead.

**MongoDB on EC2 (or Atlas on AWS)**: Full MongoDB feature set, community edition or Enterprise. Use when you need 100% MongoDB compatibility, features DocumentDB does not support, or MongoDB Atlas-specific features.

**Amazon MemoryDB for Redis**: If you need a document database with microsecond read latency, MemoryDB supports Redis JSON and can serve as an ultra-fast document store. Very different use case — in-memory, not disk-based.

### Limits and Quotas

- Maximum storage: 128 TB (standard clusters), petabytes (Elastic Clusters)
- Maximum read replicas: 15 per cluster
- Maximum instance size: db.r6g.16xlarge (64 vCPU, 512 GB RAM)
- Maximum document size: 16 MB (same as MongoDB)
- Maximum index key size: 2,048 bytes
- Maximum indexes per collection: 64
- Maximum collections per database: no hard limit (practical limits based on instance memory)
- Change stream retention: up to 7 days
- Transaction timeout: 1 minute
- Transaction size: 16 MB
- Global Clusters: up to 5 secondary regions

### Pricing Model

DocumentDB charges per instance-hour (compute), per GB-month (storage), per million I/O requests (for standard pricing) or per GB-month with no I/O charges (for I/O-Optimized pricing — same model as Aurora). Data transfer between AZs for replication is included. Backups beyond 1x the cluster storage are charged. Elastic Clusters charge per vCPU-hour.

Rough costs: a db.r6g.large instance (2 vCPU, 16 GB RAM) costs approximately $0.277/hr (~$200/month). Storage costs $0.10/GB/month. I/O costs $0.20 per million requests (or use I/O-Optimized to eliminate I/O charges at higher storage cost).

### Edge Cases and Gotchas

- DocumentDB is not MongoDB. Test your application thoroughly — subtle differences in query behavior, aggregation operators, and error messages can cause issues.
- Storage never shrinks (same as Aurora). Deleted data space is reused but not reclaimed.
- Change streams have a maximum retention of 7 days. If your consumer is offline for more than 7 days, it loses its position.
- No server-side JavaScript (`$where`, `$function`, `$accumulator`, map-reduce). This breaks applications that rely on these features.
- Failover takes approximately 30 seconds (same as Aurora). Applications must handle connection drops.
- DocumentDB uses TLS by default. Some MongoDB drivers need explicit TLS configuration. The DocumentDB CA certificate must be trusted by your application.
- Connection string format differs slightly from MongoDB. Use the DocumentDB-specific connection string, not a standard MongoDB URI.
- No support for capped collections. If your application uses capped collections for fixed-size buffers, you need a different approach (e.g., TTL indexes).
- Read preference configuration: reads from replicas are eventually consistent. For strongly consistent reads, you must read from the primary.
- DocumentDB Profiler logs slow queries (similar to MongoDB profiler) but is configured differently. Enable it for performance tuning.
- Index builds are foreground by default in older engine versions, which blocks writes. Use background index builds in production.

### Common Interview Questions

**Q: When would you choose DocumentDB over DynamoDB?**
A: Choose DocumentDB when your data is document-shaped with nested structures, when you need flexible ad-hoc queries and aggregation pipelines, when you are migrating from MongoDB, or when your query patterns are not fully known at design time. Choose DynamoDB when you need single-digit millisecond latency at scale, when access patterns are well-defined, or when you need multi-region active-active replication.

**Q: Is DocumentDB actually MongoDB?**
A: No. DocumentDB implements the MongoDB wire protocol (API compatibility) but uses a completely different storage engine (Aurora-like distributed storage). It is not a fork of MongoDB. This means most MongoDB applications work, but applications relying on MongoDB internals (oplog, specific server-side JavaScript, exact error codes) may not work identically. Always test before migrating.

**Q: How does DocumentDB handle high availability?**
A: DocumentDB replicates data 6 ways across 3 AZs (same as Aurora). It supports up to 15 read replicas. If the primary instance fails, a replica is promoted in approximately 30 seconds. Global Clusters extend this to cross-region with under 1 second replication lag and under 1 minute promotion time.

**Q: How do you handle schema migrations in DocumentDB?**
A: DocumentDB is schema-less — documents in the same collection can have different structures. Schema evolution is handled at the application level. For backward compatibility, use the "expand and contract" pattern: add new fields without removing old ones, update application code to use new fields, then clean up old fields once all consumers have migrated. For index changes, build indexes in the background to avoid blocking writes.

### Interactions with Other Services

- **Lambda**: DocumentDB change streams can trigger Lambda functions. Lambda can also query DocumentDB directly (use VPC-attached Lambda since DocumentDB runs in a VPC).
- **EventBridge Pipes**: Pipes can read from DocumentDB change streams and route to any supported target.
- **S3**: Export data to S3 for analytics. Use mongodump/mongorestore or custom ETL.
- **OpenSearch**: Sync DocumentDB data to OpenSearch for full-text search (via change streams + Lambda or a connector).
- **Secrets Manager**: Store and rotate DocumentDB credentials automatically.
- **VPC**: DocumentDB runs exclusively inside a VPC. All clients must be in the same VPC or access via VPC peering/PrivateLink.
- **CloudWatch**: DocumentDB publishes metrics for connections, CPU, memory, disk I/O, replication lag, and operations per second.

---

## AWS Step Functions

### Overview

Step Functions is a serverless workflow orchestration service. You define a state machine as a series of steps (states), and Step Functions executes them in order, handles errors, retries, branching, parallelism, and waits. It integrates with over 200 AWS services directly, often eliminating the need for glue code Lambda functions.

### Standard vs Express Workflows

**Standard workflows**:
- Maximum duration: 1 year
- Exactly-once execution semantics
- Priced per state transition ($0.025 per 1,000 transitions)
- Execution history retained for 90 days
- Maximum 25,000 history events per execution (a hard limit — see below)
- Suitable for long-running, auditable workflows

**Express workflows**:
- Maximum duration: 5 minutes
- At-least-once execution semantics
- Priced per execution, duration, and memory ($1.00 per million executions + duration charges)
- Execution history is sent to CloudWatch Logs (not retained natively)
- Much higher throughput (100,000 concurrent executions vs 1,000,000 for Standard but with throttling)
- Suitable for high-volume, short-duration event processing

### Payload Size Limit

The maximum payload (input/output of each state) is 256 KB. This is one of the most common limits hit. If your workflow processes large data, do not pass it through the state machine — store it in S3 and pass the S3 reference.

### History Events Limit

Each state transition generates one or more history events. A Standard workflow has a hard limit of 25,000 history events per execution. If your workflow has many iterations (e.g., a Map state processing thousands of items inline), you will hit this limit and the execution fails.

The solution is to use child workflows (StartExecution of another state machine) for large iterations, or use Distributed Map for massive parallelism.

### Map State

**Inline Map**: Processes items within the same execution. Each iteration counts toward the 25,000 history event limit. Maximum concurrency is 40.

**Distributed Map**: Processes items as child executions. Each item is a separate execution with its own 25,000 event limit. Can process millions of items. Items can be read directly from S3 (CSV, JSON, or inventory). Maximum concurrency of 10,000.

### Callback Pattern (waitForTaskToken)

The Callback pattern allows a state to pause and wait for an external process to complete. The state generates a task token and passes it to the target (Lambda, SQS, etc.). The workflow pauses until the external process calls SendTaskSuccess or SendTaskFailure with the token. This is ideal for human approval steps, long-running external processes, or third-party integrations.

Task tokens can wait for up to 1 year (Standard workflows) before timing out.

### Service Integrations

Step Functions can call over 200 AWS services directly from a state definition. Three integration patterns:

- **Request-Response** (default): Call the service and immediately continue to the next state.
- **Run a Job (.sync)**: Call the service and wait for the job to complete. Supported for ECS, Batch, Glue, CodeBuild, SageMaker, and others. Step Functions polls the job status automatically.
- **Wait for Callback (.waitForTaskToken)**: Send a task token to the service and wait for a callback.

### Error Handling

Step Functions has built-in error handling with Retry and Catch blocks:

- **Retry**: Automatically retry a failed state with configurable parameters (ErrorEquals, IntervalSeconds, BackoffRate, MaxAttempts).
- **Catch**: If all retries fail, catch the error and transition to a fallback state.
- **States.ALL**: Matches any error type. Use as a catch-all in Retry or Catch.
- **States.TaskFailed**: Matches any error from a task state.
- **States.Timeout**: Matches a state or execution timeout.

### Limits and Quotas

- Maximum payload: 256 KB per state input/output
- Maximum history events: 25,000 per Standard execution
- Maximum execution duration: 1 year (Standard), 5 minutes (Express)
- Maximum concurrent Standard executions: 1,000,000
- Maximum Distributed Map concurrency: 10,000
- Maximum activities per account: 10,000
- Maximum state machine size: 1 MB (ASL definition)

### Pricing Model

Standard: $0.025 per 1,000 state transitions. Free tier: 4,000 transitions/month. Express: $1.00 per million requests + duration charges based on memory and duration.

### Edge Cases and Gotchas

- The 256 KB payload limit is per state. Avoid passing large data through the workflow — use S3 references.
- The 25,000 history event limit is per execution. Use child workflows or Distributed Map for large iterations.
- Express workflows are at-least-once. Your tasks must be idempotent.
- Service integration (.sync) uses a poller that counts as state transitions. Long-running jobs generate many transitions.
- Step Functions is not suitable for sub-second latency orchestration — state transitions have inherent latency.
- The state machine definition language (ASL) is JSON-based and verbose. CDK and the Workflow Studio help, but complex workflows can be hard to maintain.

### Common Interview Questions

**Q: When would you use Step Functions vs coding orchestration in Lambda?**
A: Use Step Functions when you need visual workflow monitoring, built-in retry/error handling, long waits (callbacks, approvals), or parallel execution with bounded concurrency. Use direct Lambda orchestration only for simple, fast, synchronous chains where the overhead of Step Functions is not justified. Lambda-to-Lambda (direct invocation) is generally an anti-pattern because it couples functions, loses visibility, and wastes money on idle wait time.

**Q: How do you handle processing millions of items in Step Functions?**
A: Use Distributed Map state, which creates child executions for each item (or batch of items). It reads items directly from S3, supports up to 10,000 concurrent child executions, and each child has its own 25,000 event limit. This avoids the inline Map's history event limitation.

### Interactions with Other Services

- **Lambda**: The most common Step Functions task target.
- **DynamoDB**: Direct GetItem/PutItem/UpdateItem/DeleteItem without Lambda.
- **SQS/SNS**: Send messages directly from workflow states.
- **EventBridge**: EventBridge can start Step Functions executions.
- **S3**: Distributed Map reads items from S3. Steps can GetObject/PutObject.
- **ECS/Fargate**: Run containers as workflow steps with .sync integration.
- **Bedrock**: Invoke foundation models as workflow steps.

---

## AWS CDK (Cloud Development Kit)

### Overview

CDK is an infrastructure-as-code framework that lets you define cloud resources using familiar programming languages (TypeScript, Python, Java, C#, Go). CDK code is synthesized into CloudFormation templates and deployed via CloudFormation stacks.

### L1, L2, L3 Constructs

**L1 (Cfn) constructs**: Direct one-to-one mappings to CloudFormation resources. Names start with "Cfn" (e.g., CfnBucket). No defaults, no convenience methods. Use when L2 does not exist or when you need full control.

**L2 constructs**: Opinionated, higher-level abstractions with sensible defaults. For example, `new s3.Bucket(this, 'MyBucket')` creates a bucket with encryption, versioning options, lifecycle policies, and grant methods for IAM permissions. Most day-to-day CDK code uses L2 constructs.

**L3 (patterns) constructs**: Even higher-level constructs that combine multiple resources into common patterns. For example, `LambdaRestApi` creates an API Gateway backed by a Lambda function with all the necessary permissions and configurations.

### Construct IDs and Replacement

Every construct has a logical ID derived from its construct ID (the second argument in the constructor). If you change the construct ID, CDK generates a new logical ID, which CloudFormation interprets as a resource deletion + creation (replacement), not an update. For stateful resources (databases, buckets), this means data loss.

This is one of the most dangerous CDK gotchas. Renaming a construct ID for an RDS instance or DynamoDB table will delete the old resource and create a new one — destroying all your data. Always set RemovalPolicy.RETAIN on stateful resources.

### cdk bootstrap

Before deploying CDK stacks to an AWS account/region, you must run `cdk bootstrap`. This creates a CloudFormation stack (CDKToolkit) with an S3 bucket for assets, an ECR repository for Docker images, and IAM roles for deployment. You bootstrap once per account/region combination.

### Aspects

Aspects are a way to apply a visitor pattern to all constructs in a CDK app. You implement the `visit(node)` method and it is called for every construct. Common uses: enforcing tagging standards, adding encryption to all resources, checking compliance rules.

### Removal Policies

By default, CDK sets the CloudFormation DeletionPolicy to DELETE for most resources. This means that when you remove a resource from your CDK code or destroy a stack, the underlying AWS resource is deleted.

For stateful resources (databases, S3 buckets, log groups), always set `removalPolicy: RemovalPolicy.RETAIN` or `RemovalPolicy.SNAPSHOT`. This prevents accidental data loss during refactoring or stack deletion.

### Limits and Quotas

- CloudFormation stack limit: 500 resources per stack (use nested stacks or multiple stacks for larger applications)
- Asset size limit: depends on S3/ECR limits (effectively unlimited)
- Synthesis time: depends on complexity, but large CDK apps can take minutes to synthesize

### Edge Cases and Gotchas

- Changing construct IDs causes resource replacement. Never rename stateful resource construct IDs.
- Default removal policy is DELETE. Always set RETAIN for databases, buckets, and other stateful resources.
- CDK context values (from cdk.json or context lookups) are cached. Stale context can cause unexpected behavior. Use `cdk context --clear` to reset.
- Cross-stack references create CloudFormation exports, which cannot be deleted if another stack references them. This can block stack updates.
- Large CDK apps can hit CloudFormation limits (500 resources per stack, 200 outputs, 60 parameters). Split into multiple stacks.
- CDK deploy uses CloudFormation change sets, which can be slow (30+ seconds just to create the change set). Use `cdk deploy --method=direct` for faster deployments (but no change set review).
- Escape hatches (accessing the underlying L1 construct) are sometimes necessary but should be used sparingly.

### Common Interview Questions

**Q: What happens if you rename a construct ID in CDK?**
A: The logical ID in the CloudFormation template changes. CloudFormation treats this as a delete of the old resource and creation of a new one (replacement). For stateful resources like databases or S3 buckets, this causes data loss. Always set RemovalPolicy.RETAIN on stateful resources to prevent accidental deletion.

**Q: How does CDK differ from Terraform?**
A: CDK uses familiar programming languages with full IDE support (type checking, autocomplete, loops, conditionals). It synthesizes to CloudFormation and uses CloudFormation for state management and deployment. Terraform uses HCL (a declarative DSL), manages its own state file, and supports multi-cloud. CDK is AWS-only (CDKTF exists for Terraform). CDK's L2 constructs provide opinionated defaults that reduce boilerplate compared to Terraform's resource definitions.

### Interactions with Other Services

- **CloudFormation**: CDK synthesizes to CloudFormation templates and deploys via CloudFormation.
- **S3/ECR**: CDK bootstrap creates S3 and ECR repositories for storing deployment assets.
- **IAM**: CDK L2 constructs automatically create least-privilege IAM policies via `grant*` methods.
- **All AWS Services**: CDK can define any AWS resource that CloudFormation supports.

---

## AWS CloudFormation

### Overview

CloudFormation is the underlying deployment engine for CDK and the native IaC service for AWS. You define resources in JSON or YAML templates, and CloudFormation creates, updates, and deletes them as a unit called a stack. CDK synthesizes to CloudFormation, so understanding CloudFormation behavior is essential even if you write CDK.

### Stack Operations and States

When you create or update a stack, CloudFormation determines the changes needed, executes them, and reports the result. Key states:

- **CREATE_COMPLETE / UPDATE_COMPLETE**: Success.
- **ROLLBACK_COMPLETE**: Creation failed and CloudFormation rolled back all resources. The stack exists but has no resources. You must delete it before recreating.
- **UPDATE_ROLLBACK_COMPLETE**: Update failed and CloudFormation rolled back to the previous state. The stack is usable at the previous version.
- **UPDATE_ROLLBACK_FAILED**: Rollback itself failed. This is the worst state — manual intervention required. Common cause: a resource was modified outside of CloudFormation.

### Change Sets

A change set previews what CloudFormation will do before it does it. You create a change set, review the additions/modifications/deletions, then execute or discard it. This is critical for production — you see which resources will be replaced (data loss risk) before applying.

CDK uses change sets by default during `cdk deploy`. The `--method=direct` flag skips change sets for faster deploys (dev environments).

### Drift Detection

Drift occurs when a resource's actual configuration diverges from what CloudFormation expects (someone modified it manually or through another tool). Drift detection compares the live state to the template and reports differences.

Use drift detection to: identify manual changes that bypass IaC, verify compliance (is the actual state what the template says?), and debug unexpected behavior.

### Nested Stacks vs StackSets

**Nested stacks**: Break a large template into smaller, reusable components. A parent stack references child stacks. Used to stay within the 500-resource limit per stack and to share common components (VPC, security groups) across applications.

**StackSets**: Deploy a single template across multiple accounts and regions. Used for organization-wide resources: CloudTrail in every account, GuardDuty in every region, baseline IAM roles. Integrates with AWS Organizations for automatic deployment to new accounts.

### Rollback and Stack Policies

**Automatic rollback**: By default, CloudFormation rolls back all changes if any resource fails during create or update. You can disable rollback for debugging (`--disable-rollback`), but never in production.

**Rollback triggers**: CloudWatch alarms that, if triggered during a stack operation, cause CloudFormation to roll back. Example: deploy a new Lambda version, monitor error rate, roll back if errors spike.

**Stack policies**: JSON policies that prevent accidental updates to critical resources. Example: prevent any update to the production RDS instance. Stack policies are applied per-stack and can be overridden with explicit permission during specific updates.

### Custom Resources

For resources that CloudFormation does not natively support, custom resources invoke a Lambda function during create/update/delete. The Lambda function performs the action and reports success/failure back to CloudFormation.

Use cases: seed a database during stack creation, configure a third-party service, clean up resources on deletion. CDK's `AwsCustomResource` construct simplifies this for AWS API calls.

### Edge Cases and Gotchas

- Resources modified outside CloudFormation cause drift and can break updates. Use drift detection regularly.
- ROLLBACK_COMPLETE stacks cannot be updated — only deleted. This catches many first-time users.
- UPDATE_ROLLBACK_FAILED requires manual intervention — you may need to skip specific resources using `ContinueUpdateRollback` with `ResourcesToSkip`.
- Stack outputs and exports create dependencies between stacks. You cannot delete an export used by another stack.
- Template size limit: 1 MB (from S3). Parameter limit: 200. Output limit: 200. Resource limit: 500 per stack.
- CloudFormation does not parallelize all resource operations. Dependencies (explicit and implicit) create ordering. Large stacks can take 30+ minutes to deploy.

---

## Auto Scaling

### EC2 Auto Scaling

EC2 Auto Scaling automatically adjusts the number of EC2 instances in an Auto Scaling Group (ASG) based on demand.

**Launch templates**: Define the instance configuration (AMI, instance type, key pair, security groups, user data). Launch templates support versioning and are required for new ASGs.

**Scaling policies**:

- **Target tracking**: The simplest and most common. Set a target metric value (e.g., average CPU at 50%). ASG adds/removes instances to maintain the target. Works well for most use cases.
- **Step scaling**: Define ranges (CPU 50-70%: add 1 instance, 70-90%: add 3, >90%: add 5). More granular control than target tracking but more complex to configure.
- **Scheduled scaling**: Set desired capacity at specific times. Use for known traffic patterns (business hours, batch windows, marketing events).
- **Predictive scaling**: Uses ML to predict traffic based on historical patterns and pre-scales. Combines with target tracking for the actual scaling. Good for cyclical workloads.

**Cooldown period**: After a scaling activity, ASG waits before allowing another. Default 300 seconds. Prevents rapid oscillation (scale up, scale down, scale up). Target tracking has its own built-in cooldown that is generally more intelligent.

**Health checks**: ASG can use EC2 status checks (default) or ELB health checks. ELB health checks are recommended — they verify the application is responding, not just that the instance is running.

**Lifecycle hooks**: Execute custom actions during instance launch or termination. Use cases: register with a configuration management tool on launch, drain connections before termination, send logs to S3 before shutdown.

**Warm pools**: Pre-initialized instances in a stopped state that can be started faster than launching from scratch. Reduces scale-out latency from minutes to seconds. You pay for stopped instance storage but not compute.

**Instance refresh**: Rolling replacement of all instances in an ASG (useful for deploying a new AMI). Configure minimum healthy percentage and checkpoint to control the rollout.

### Application Auto Scaling

Application Auto Scaling provides scaling for non-EC2 AWS services:

- **ECS service task count**
- **DynamoDB provisioned capacity** (RCU/WCU)
- **Aurora Serverless v2 ACU**
- **Lambda provisioned concurrency**
- **SageMaker endpoint instance count**
- **ElastiCache replication group replicas**
- **And more**

Same policy types as EC2: target tracking, step, scheduled. Target tracking is the default choice — set the metric and target, let AWS handle the math.

### Auto Scaling Interview Questions

**Q: What is the difference between target tracking and step scaling?**
A: Target tracking is simpler — you set a target (e.g., CPU 50%) and ASG adjusts to maintain it. It handles the scaling math automatically. Step scaling gives explicit control — you define exactly how many instances to add/remove at each threshold. Use target tracking by default; use step scaling only when you need asymmetric responses (scale out aggressively, scale in conservatively).

**Q: How do you handle scaling for a queue-based workload?**
A: Use the SQS backlog-per-instance metric as the target tracking metric: `ApproximateNumberOfMessagesVisible / number of instances`. Set the target to the number of messages one instance can process within the acceptable latency. ASG scales out when the backlog per instance exceeds the target.

---

## Storage Types: EBS, EFS, FSx, and Instance Store

### Amazon EBS (Elastic Block Store)

EBS provides block storage volumes for EC2 instances. Think of it as a virtual hard drive. A volume is attached to a single EC2 instance (or multiple with io2 Multi-Attach) and appears as a block device.

**Volume types**:

- **gp3** (General Purpose SSD): Default choice. 3,000 baseline IOPS + 125 MB/s throughput, independently scalable up to 16,000 IOPS and 1,000 MB/s. Cost: $0.08/GB/month. Use for: boot volumes, development, most workloads.
- **gp2** (older General Purpose SSD): IOPS scales with volume size (3 IOPS/GB, burst to 3,000). Being superseded by gp3 which is cheaper and more flexible. No reason to use gp2 for new volumes.
- **io2 Block Express** (Provisioned IOPS SSD): Up to 256,000 IOPS and 4,000 MB/s. Sub-millisecond latency. Cost: $0.125/GB/month + $0.065/provisioned IOPS. Use for: databases (Aurora on EC2, self-managed databases), latency-sensitive workloads.
- **st1** (Throughput Optimized HDD): High throughput (up to 500 MB/s), low IOPS. Cost: $0.045/GB/month. Use for: big data, data warehouses, log processing. Cannot be a boot volume.
- **sc1** (Cold HDD): Lowest cost. 250 MB/s throughput. Cost: $0.015/GB/month. Use for: infrequently accessed data, cold storage. Cannot be a boot volume.

**Snapshots**: Point-in-time copies of EBS volumes, stored in S3 (managed by AWS). Incremental — only changed blocks are stored. Use for backups, disaster recovery, and creating volumes in other AZs/regions.

**Encryption**: All EBS volume types support encryption (AES-256). Encrypted volumes use KMS keys. You can set a default encryption setting per region so all new volumes are encrypted automatically.

**Multi-Attach** (io2 only): Attach a single volume to up to 16 EC2 instances simultaneously. Use for: clustered applications that manage concurrent write access (e.g., shared storage for a database cluster). The application must handle write coordination — EBS does not manage concurrency.

### Amazon EFS (Elastic File System)

EFS is a managed NFS file system that can be shared across multiple EC2 instances, Lambda functions, ECS tasks, and EKS pods simultaneously. Unlike EBS (attached to one instance), EFS is a shared file system.

**How it works**: Create a file system, create mount targets in your VPC subnets, mount from your instances using NFS. EFS automatically scales storage (no provisioning needed) and replicates data across multiple AZs.

**Performance modes**:
- **General Purpose** (default): Low latency. Use for: web serving, CMS, home directories, most workloads.
- **Max I/O**: Higher aggregate throughput, slightly higher latency. Use for: big data, media processing, highly parallelized workloads with hundreds of instances.

**Throughput modes**:
- **Elastic** (default, recommended): Automatically scales throughput based on workload. Pay for what you use.
- **Provisioned**: Set a fixed throughput level regardless of storage size. Use when you need consistent throughput independent of storage.
- **Bursting**: Throughput scales with storage size (50 MB/s per TB, burst to 100 MB/s per TB). Older mode, less flexible.

**Storage classes**: Standard (frequently accessed, multi-AZ), Infrequent Access (lower cost, per-access fee), One Zone (single AZ, 47% cheaper), One Zone-IA (cheapest). Use lifecycle policies to transition files automatically.

**Lambda + EFS**: Lambda can mount EFS file systems, enabling shared state across invocations and functions. Use for: ML model loading, shared configuration, large reference data. Requires Lambda in VPC.

### Amazon FSx

FSx provides managed file systems for specialized workloads:

**FSx for Lustre**: High-performance parallel file system for compute-intensive workloads (HPC, ML training, media rendering). Integrates natively with S3 — data is lazy-loaded from S3 on first access and can be written back. Hundreds of GB/s throughput, millions of IOPS. Use when EFS throughput is insufficient.

**FSx for Windows File Server**: Managed Windows-native file system (SMB protocol). Active Directory integration. Use for: Windows workloads, .NET applications, SQL Server, SharePoint. The only managed option for SMB on AWS.

**FSx for NetApp ONTAP**: Managed NetApp — supports NFS, SMB, and iSCSI simultaneously. Data deduplication, compression, snapshots, cloning. Use when you need multi-protocol access or are migrating from on-premises NetApp.

**FSx for OpenZFS**: Managed ZFS — high performance, snapshots, cloning, compression. NFS protocol. Use for: Linux workloads migrating from on-premises ZFS.

### Instance Store

Instance store provides temporary block storage physically attached to the host machine. NVMe SSDs with very high IOPS (millions) and throughput. Data is lost when the instance stops, terminates, or the underlying hardware fails.

Use cases: temporary scratch space, buffers, caches, shuffle storage for Spark/Hadoop. Never use for data that must survive instance lifecycle.

### Storage Decision Framework

- **Block storage for one instance**: EBS (gp3 default, io2 for high IOPS)
- **Shared file system (Linux/NFS)**: EFS (general), FSx for Lustre (HPC/ML)
- **Shared file system (Windows/SMB)**: FSx for Windows File Server
- **Object storage**: S3
- **Temporary high-performance scratch**: Instance Store
- **Cache**: ElastiCache/DAX/MemoryDB

### Storage Interview Questions

**Q: What is the difference between EBS, EFS, and S3?**
A: EBS is block storage attached to a single EC2 instance (like a hard drive). EFS is a shared NFS file system accessible from multiple instances/Lambda/ECS simultaneously. S3 is object storage accessed via HTTP API (not mounted as a file system). Use EBS for databases and single-instance storage, EFS for shared files and Lambda state, S3 for unlimited object storage and data lakes.

**Q: When would you choose gp3 vs io2?**
A: gp3 is the default — 3,000 IOPS baseline at $0.08/GB. Use for most workloads. Choose io2 when you need more than 16,000 IOPS, sub-millisecond latency, or Multi-Attach. io2 costs more ($0.125/GB + per-IOPS) but delivers up to 256,000 IOPS. Self-managed databases on EC2 are the primary io2 use case.

---

## Specialized Databases

AWS offers purpose-built databases for different data models. Choosing the right one is a frequent interview topic.

### Amazon RDS (Relational Database Service)

RDS provides managed relational databases: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server. AWS manages patching, backups, replication, and failover. You manage the schema, queries, and performance tuning.

**Multi-AZ**: Synchronous standby replica in another AZ. Automatic failover in 60-120 seconds. The standby is not readable (unlike Aurora replicas). Doubles the cost.

**Read replicas**: Asynchronous replicas for read scaling. Up to 15 replicas (PostgreSQL/MySQL). Replicas can be in other regions (cross-region read replicas). Can be promoted to standalone database.

**RDS vs Aurora**: Aurora provides higher throughput (5x PostgreSQL), faster replication (sub-10ms), shared storage (no storage per-replica), faster failover (30s vs 60-120s), and more replicas (15 vs 5 for PostgreSQL). Use Aurora unless you need Oracle/SQL Server or the specific RDS pricing model.

### Amazon Redshift

Redshift is a columnar data warehouse for analytics. It uses massively parallel processing (MPP) to execute complex queries across petabytes of data.

- **Columnar storage**: Data is stored by column, not by row. Analytics queries that aggregate a few columns across millions of rows are extremely fast because only relevant columns are read.
- **Redshift Serverless**: Auto-scaling, pay-per-query. No clusters to manage. Use for intermittent or unpredictable analytics workloads.
- **Redshift Spectrum**: Query data in S3 directly from Redshift without loading it. Extends your warehouse to the data lake.

**Redshift vs Athena**: Both query data with SQL. Redshift is for repeated, complex analytics with consistent performance (data is loaded and optimized). Athena is for ad-hoc queries directly on S3 (no data loading, pay per scan). Use Athena for exploration, Redshift for production dashboards and regular reporting.

### Amazon OpenSearch Service

OpenSearch (successor to Elasticsearch) provides search and log analytics.

- **Full-text search**: Fast text search with relevance ranking, fuzzy matching, facets, and aggregations
- **Log analytics**: Ingest logs from CloudWatch, Kinesis, Firehose. Built-in Dashboards (Kibana-compatible) for visualization.
- **OpenSearch Serverless**: No clusters to manage, auto-scaling. Two collection types: search (low-latency search) and time series (log analytics).
- **Vector search (k-NN)**: Store and search vector embeddings for RAG, semantic search, and recommendation systems. This is what Bedrock Knowledge Bases uses under the hood.

**OpenSearch vs CloudWatch Logs Insights**: CloudWatch is simpler and cheaper for basic log querying. OpenSearch is for complex search, dashboarding, and when you need full-text search with relevance ranking across large volumes.

### Amazon Neptune

Neptune is a graph database supporting property graph (Gremlin/openCypher) and RDF (SPARQL).

Use cases: social networks (friend-of-friend queries), fraud detection (find connected suspicious accounts), knowledge graphs (entity relationships), recommendation engines (users who bought X also bought Y), identity graphs, network topology.

Neptune stores data as vertices (nodes) and edges (relationships). Queries traverse these relationships efficiently — something that relational databases handle poorly (multiple self-joins).

Neptune Serverless scales automatically from 1 to 128 NCUs (Neptune Capacity Units). Use for variable graph workloads.

### Amazon Timestream

Timestream is a time-series database for IoT metrics, application monitoring, and DevOps data.

- **Built-in time-series functions**: Interpolation, smoothing, time-bucketed aggregation
- **Automatic tiering**: Recent data in memory (fast), older data in magnetic storage (cheap)
- **Serverless**: Auto-scaling, no capacity management
- **InfluxDB-compatible** query interface available via Amazon Timestream for InfluxDB

Use cases: IoT sensor data, application metrics, financial tick data, fleet tracking.

**Timestream vs CloudWatch Metrics**: CloudWatch is for AWS resource monitoring with basic custom metrics. Timestream is for high-cardinality, high-volume time-series data with complex query needs (billions of data points, custom aggregations).

### Amazon Keyspaces (for Apache Cassandra)

Keyspaces is a managed Apache Cassandra-compatible database. Use when migrating from Cassandra or when you need Cassandra's wide-column data model with fully managed operations.

**Keyspaces vs DynamoDB**: Both are wide-column stores. DynamoDB is AWS-native with richer integration. Keyspaces is for Cassandra compatibility (CQL queries, existing Cassandra drivers). For new applications, default to DynamoDB. For Cassandra migrations, use Keyspaces.

### Database Decision Tree

1. **Relational data with complex joins?** Aurora (or RDS for Oracle/SQL Server)
2. **Key-value lookups at scale?** DynamoDB
3. **Document-shaped data with flexible queries?** DocumentDB
4. **Full-text search or log analytics?** OpenSearch
5. **Graph relationships?** Neptune
6. **Time-series data?** Timestream
7. **Data warehouse / analytics?** Redshift
8. **Ad-hoc SQL on S3?** Athena
9. **Cassandra compatibility?** Keyspaces
10. **In-memory cache?** ElastiCache (Redis)
11. **In-memory durable database?** MemoryDB

### Database Interview Questions

**Q: How do you choose between DynamoDB, Aurora, and DocumentDB?**
A: DynamoDB for known access patterns, single-digit millisecond latency, and serverless scaling. Aurora for complex relational queries with joins, transactions, and SQL compatibility. DocumentDB for document-shaped data with flexible ad-hoc queries and MongoDB compatibility. The access pattern determines the choice: if you can model it with partition/sort keys, use DynamoDB. If you need JOINs, use Aurora. If you need nested document queries, use DocumentDB.

**Q: When would you use Redshift vs Athena?**
A: Redshift for production analytics with consistent performance on loaded, optimized data. Athena for ad-hoc exploration of data already in S3 with no infrastructure to manage. Many organizations use both: Athena for data exploration and Redshift for production dashboards. If your analytics are infrequent and data is already in S3, start with Athena. If you run the same complex queries repeatedly and need sub-second performance, use Redshift.
