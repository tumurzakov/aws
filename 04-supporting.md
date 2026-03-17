# Part 4: Supporting Services

---

## IAM (Identity and Access Management)

### Overview

IAM controls who (authentication) can do what (authorization) in your AWS account. Every AWS API call is evaluated against IAM policies. IAM is global — it is not region-specific.

### Policy Evaluation Logic

When a principal (user, role, or service) makes an AWS API call, IAM evaluates all applicable policies in this order:

1. **Explicit deny**: If any policy explicitly denies the action, the request is denied. Period. Explicit deny always wins.
2. **SCPs (Service Control Policies)**: If the account is in an AWS Organization, SCPs define the maximum permissions. Even if IAM allows it, an SCP can deny it.
3. **Resource-based policies**: Policies attached to the resource (S3 bucket policy, SQS queue policy, etc.). Resource-based policies can grant cross-account access without requiring the calling account to have IAM permissions (for some services).
4. **Permission boundaries**: An IAM feature that sets the maximum permissions for a user or role. The effective permissions are the intersection of the identity policy and the permission boundary.
5. **Identity-based policies**: Policies attached to the IAM user, group, or role. These explicitly grant permissions.
6. **Session policies**: Policies passed during role assumption or federation.

The effective permission is the intersection of all allow policies minus any explicit deny. The default is deny — if no policy explicitly allows an action, it is denied.

### Identity-Based vs Resource-Based Policies

**Identity-based policies** are attached to IAM users, groups, or roles. They specify what actions the principal can perform on which resources.

**Resource-based policies** are attached to AWS resources (S3 buckets, SQS queues, KMS keys, Lambda functions, etc.). They specify who (which principals) can perform which actions on the resource.

Key difference for cross-account access: With identity-based policies alone, both the source account (identity policy allowing the action) and the target account (resource policy allowing the principal) must grant access. With resource-based policies, the resource policy alone can grant access to a cross-account principal (for supported services), and the calling principal does not need an identity policy that allows the action — the resource policy is sufficient.

### Condition Keys

Policies can include conditions that must be true for the policy to apply. Common condition keys:

- `aws:SourceIp` — restrict to specific IP ranges
- `aws:MultiFactorAuthPresent` — require MFA
- `aws:PrincipalOrgID` — restrict to principals in your AWS Organization
- `aws:RequestedRegion` — restrict to specific regions
- `aws:CalledVia` — check if the call was made through a specific service (e.g., CloudFormation)
- Service-specific keys like `s3:prefix`, `dynamodb:LeadingKeys`, `ec2:ResourceTag`

### Permission Boundaries

A permission boundary is a managed policy that sets the maximum permissions for an IAM entity. The entity's effective permissions are the intersection of its identity policies and its permission boundary.

Use case: Allow developers to create IAM roles (for their Lambda functions) but constrain the roles they create to a specific set of permissions. Set a permission boundary on the developer's role that limits the permissions they can delegate.

### SCPs (Service Control Policies)

SCPs are organization-level policies that set permission guardrails for all accounts in an organizational unit (OU). SCPs do not grant permissions — they only restrict them. Even the root user in an account is subject to SCPs.

Common SCP patterns: deny access to unused regions, deny disabling CloudTrail, deny creating resources without specific tags, deny leaving the organization.

### Tag-Based Access Control (ABAC)

ABAC uses tags on IAM principals and AWS resources to control access dynamically, instead of listing specific resource ARNs in policies. A single ABAC policy can say "allow access to any resource where the resource's `project` tag matches the principal's `project` tag."

Benefits over traditional RBAC: you do not need to update policies when new resources are created (as long as they are tagged correctly). You do not need separate policies per project or team. One generic policy scales across the organization.

Implementation: Tag IAM roles with attributes (team, project, environment). Tag resources with matching attributes. Write policies using condition keys like `aws:ResourceTag/project` and `aws:PrincipalTag/project`. Use `StringEquals` conditions to match them.

ABAC works best when your tagging strategy is disciplined and enforced. Without consistent tagging, ABAC policies fail silently — access is denied because tags do not match, and debugging is painful.

### IAM Policy Variables

Policy variables let you write generic policies that reference attributes of the current request. Instead of hardcoding resource ARNs, you use variables that are resolved at evaluation time.

Common variables:
- `${aws:username}` — the IAM user name
- `${aws:PrincipalTag/team}` — a tag on the calling principal
- `${aws:PrincipalAccount}` — the account ID of the caller
- `${s3:prefix}` — the S3 prefix being accessed
- `${dynamodb:LeadingKeys}` — the partition key being accessed in DynamoDB

Example use case: Allow each user to access only their own S3 prefix: `arn:aws:s3:::bucket/${aws:username}/*`. One policy, works for all users.

### Cross-Account Access with AssumeRole

The standard pattern for cross-account access:

1. Account B creates an IAM role with a trust policy allowing Account A's principal to assume it
2. Account A's principal calls `sts:AssumeRole` to get temporary credentials for the role in Account B
3. Account A uses the temporary credentials to call APIs in Account B

The trust policy on the role in Account B is a resource-based policy that specifies which principals in Account A can assume the role. You can add conditions (require external ID, require MFA, restrict source IP).

### The Confused Deputy Problem and ExternalId

The confused deputy problem occurs when a third-party service (Service X) assumes a role in your account to perform actions on your behalf. If a malicious actor also uses Service X, they could trick Service X into assuming your role using their request — because Service X uses the same role ARN for all customers.

The solution is the `ExternalId` condition. You set a unique ExternalId (a secret shared between you and Service X) in the role's trust policy. When Service X assumes the role, it must provide the correct ExternalId. The malicious actor does not know your ExternalId, so their request fails.

ExternalId is required whenever you create cross-account roles for third-party services. It is not a security mechanism on its own (it is not a secret in the cryptographic sense) but it prevents the confused deputy attack.

### Session Tags

Session tags are key-value pairs passed during `AssumeRole`, `AssumeRoleWithSAML`, or `AssumeRoleWithWebIdentity`. They are available as `aws:PrincipalTag/*` condition keys for the duration of the session.

Use case: A CI/CD system assumes a deployment role and passes `{"project": "orders", "env": "prod"}` as session tags. The role's policies use ABAC to restrict access to only resources tagged with matching project and environment. One role, many teams, dynamic permissions per session.

Session tags are the foundation of scalable ABAC in multi-team environments. They eliminate the need to create separate roles per team or project.

### Service-Linked Roles

Service-linked roles are IAM roles created and managed by an AWS service. They have predefined permissions that the service needs to operate. You cannot modify their policies or delete them while the service is using them.

Examples: AWSServiceRoleForElasticLoadBalancing (created when you use ALB/NLB), AWSServiceRoleForAutoScaling, AWSServiceRoleForAmazonECS.

Important: If your SCP or permission boundary blocks the creation of service-linked roles, the associated AWS service will fail to operate. Ensure SCPs allow `iam:CreateServiceLinkedRole` for services you use.

### IAM Identity Center (SSO)

IAM Identity Center (formerly AWS SSO) is the recommended way to manage human access to multiple AWS accounts. Instead of creating IAM users in each account, you manage users centrally in Identity Center (or connect it to your corporate directory — Active Directory, Okta, Azure AD).

Key concepts:
- **Permission sets**: Define what a user can do in an account. A permission set is a template that becomes an IAM role in each assigned account.
- **Multi-account assignment**: Assign users/groups to accounts with specific permission sets. One user can have different permissions in different accounts.
- **Temporary credentials**: Identity Center generates temporary STS credentials when users sign in. No long-lived access keys.

Identity Center integrates with the AWS CLI (`aws sso login`) and the AWS console (single sign-on portal with account tiles).

Use Identity Center for all human access. Use IAM roles for service-to-service access. Avoid IAM users with long-lived access keys — they are a security liability.

### IAM Access Analyzer

Access Analyzer helps you identify and refine permissions. It has three main capabilities:

**External access findings**: Analyzes resource-based policies (S3, SQS, KMS, Lambda, IAM roles, etc.) to identify resources shared with principals outside your account or organization. Flags unintended public or cross-account access.

**Unused access findings**: Identifies IAM roles, users, access keys, and permissions that have not been used within a configurable time window. Helps you remove stale permissions.

**Policy generation**: Generates least-privilege IAM policies based on actual API usage recorded in CloudTrail. You provide a CloudTrail trail and a time period, and Access Analyzer creates a policy containing only the actions and resources that were actually used.

Policy generation is the most practical way to implement least privilege at scale. Start with broad permissions, let the workload run for a few weeks, generate a policy from actual usage, then replace the broad policy with the generated one.

### IAM Roles Anywhere

IAM Roles Anywhere extends IAM roles to workloads running outside of AWS (on-premises servers, other cloud providers, IoT devices). Workloads authenticate using X.509 certificates issued by a Certificate Authority (CA) that you register with IAM Roles Anywhere. After authentication, they receive temporary STS credentials — the same temporary credentials that AWS services use.

This eliminates the need for long-lived AWS access keys on non-AWS infrastructure. The workload's certificate is exchanged for short-lived credentials, following the same security model as IAM roles within AWS.

### Limits and Quotas

- Maximum managed policies per user/role: 10
- Maximum inline policy size: 2,048 characters (user), 10,240 (role)
- Maximum managed policy size: 6,144 characters
- Maximum roles per account: 1,000 (soft limit)
- Maximum groups per account: 300
- STS session duration: 1-12 hours

### Edge Cases and Gotchas

- Explicit deny always wins. There is no way to override an explicit deny except by removing the deny statement.
- The root user ignores permission boundaries and identity policies but is subject to SCPs.
- Wildcard in resource ARN (`*`) is different from wildcard in action (`*`). Be careful with both.
- `NotAction` and `NotResource` are not denies — they are "everything except." They are confusing and error-prone. Use them carefully.
- IAM changes are eventually consistent globally. A new policy may take a few seconds to propagate worldwide.
- Service-linked roles cannot be modified or deleted while the service is using them. SCPs blocking `iam:CreateServiceLinkedRole` break AWS services.
- Resource-based policies with `Principal: "*"` grant public access (e.g., public S3 bucket). Always restrict the principal.
- Cross-account access via resource-based policies does not require the calling account to have an explicit IAM allow (for services that support resource-based policies). This is a subtle but important distinction.
- IAM policy size limits are real constraints. A managed policy cannot exceed 6,144 characters. Complex ABAC policies or policies with many resource ARNs can hit this limit. Split into multiple policies or use wildcards with conditions.
- Access keys should never be used for human access. Use IAM Identity Center for humans. Use IAM roles for services. Rotate or delete any existing access keys.
- `aws:SourceIp` condition does not work when the call is made through an AWS service (e.g., Lambda calling DynamoDB). Use `aws:VpcSourceIp` or VPC endpoint conditions instead.

### Common Interview Questions

**Q: How does IAM policy evaluation work when multiple policies apply?**
A: IAM starts with a default deny. It evaluates all applicable policies (identity, resource, SCPs, permission boundaries). If any policy explicitly denies, the request is denied (explicit deny wins). If no policy explicitly allows, the request is denied (default deny). Only if at least one policy allows and no policy denies is the request permitted. The effective permission is the intersection of all allow sets minus all deny sets.

**Q: How do you implement least privilege in practice?**
A: Start with zero permissions and add only what is needed. Use IAM Access Analyzer to generate policies based on actual API usage from CloudTrail. Use permission boundaries for delegated administration. Use SCPs for organization-wide guardrails. Regularly audit with Access Analyzer unused access findings. Prefer specific actions over wildcards. Use ABAC with tags for dynamic, scalable permissions.

**Q: What is the difference between RBAC and ABAC on AWS?**
A: RBAC (Role-Based Access Control) uses different IAM roles per job function, each with explicit resource ARNs in policies. ABAC (Attribute-Based Access Control) uses tags on principals and resources with generic policies that match tags dynamically. RBAC requires policy updates when resources change; ABAC does not (as long as tagging is correct). ABAC scales better but requires disciplined tagging.

**Q: How do you manage access for a team of 50 developers across 10 AWS accounts?**
A: Use IAM Identity Center connected to your corporate directory. Create permission sets (e.g., Developer, ReadOnly, Admin). Assign groups to accounts with appropriate permission sets. Developers sign in once and get temporary credentials for each account. No IAM users, no access keys, centralized audit trail.

**Q: What is the confused deputy problem?**
A: When a trusted third-party service (acting as a "deputy") is tricked by a malicious actor into performing actions on your resources using the malicious actor's context. The solution is ExternalId — a unique token in the role trust policy that only the legitimate deputy knows. The malicious actor cannot provide the correct ExternalId, so the AssumeRole call fails.

---

## AWS Organizations and Resource Access Manager (RAM)

### AWS Organizations

AWS Organizations is an account management service that enables you to consolidate multiple AWS accounts into an organization that you create and centrally manage.

- **Centralized Management**: Manage multiple accounts from a single place.
- **Service Control Policies (SCPs)**: Centralized control over the maximum available permissions for all accounts in your organization.
- **Consolidated Billing**: One bill for all your accounts, allowing you to benefit from volume discounts (e.g., S3 storage tiers, EC2 Savings Plans) shared across the organization.
- **Automated Account Creation**: Use the Organizations API to programmatically create new accounts.

### Resource Access Manager (RAM)

RAM allows you to share AWS resources across AWS accounts or within your AWS Organization. This reduces operational overhead and costs by avoiding resource duplication.

- **Shared Resources**: Share Transit Gateways, Subnets (VPC sharing), License Manager configurations, and more.
- **VPC Sharing**: Allow multiple accounts to create their resources (EC2, Lambda, RDS) in a centrally managed VPC. The owner of the VPC (the "Network Account") shares subnets with "Participant Accounts." This simplifies network management and reduces the number of VPCs and NAT Gateways.

---

## Amazon CloudWatch

### Overview

CloudWatch is AWS's monitoring and observability service. It collects metrics, logs, and traces from AWS resources and applications. It provides alarms, dashboards, and insights for operational visibility.

### Metrics

Every AWS service publishes metrics to CloudWatch. EC2 instances report CPU, network, and disk metrics. Lambda reports invocations, duration, errors, and throttles. DynamoDB reports consumed capacity, throttles, and latency.

**Standard resolution**: 1-minute data points (default for most services). **High resolution**: 1-second data points (available for custom metrics, at higher cost). Metrics are retained based on resolution: 1-second data for 3 hours, 1-minute for 15 days, 5-minute for 63 days, 1-hour for 455 days.

**Custom metrics**: You can publish your own metrics via PutMetricData API. Custom metrics cost approximately $0.30/month each (first 10 are free). This adds up quickly — 1,000 custom metrics costs $300/month.

### Logs

CloudWatch Logs stores, monitors, and accesses log data. Log groups contain log streams. Log groups have configurable retention (1 day to 10 years, or never expire).

**Default retention is never expire** — this is a cost trap. If you do not set a retention policy, logs accumulate forever and storage costs grow indefinitely. Always set a retention policy on every log group.

CloudWatch Logs supports metric filters (extract metric values from log lines), subscription filters (stream logs to Lambda, Kinesis, or Firehose), and cross-account log sharing.

### Logs Insights

CloudWatch Logs Insights is a query engine for CloudWatch Logs. It uses a SQL-like query language to search, filter, aggregate, and visualize log data. Queries are charged per GB scanned.

Key capabilities: filter by field, aggregate (count, avg, sum, max, min, percentile), sort, limit, parse unstructured log lines with regex, join across log groups.

### Alarms

CloudWatch Alarms monitor a metric and trigger actions when the metric crosses a threshold. Actions can include SNS notifications, Auto Scaling actions, EC2 actions, or Systems Manager actions.

Alarm states: OK, ALARM, INSUFFICIENT_DATA. Alarms evaluate metrics over a configurable period and require a configurable number of data points exceeding the threshold (to avoid flapping).

**Composite alarms** combine multiple alarms with AND/OR logic. Use them to reduce alarm noise — for example, only alert if both CPU is high AND error rate is high.

### Limits and Quotas

- Custom metric cost: ~$0.30/month per metric
- Logs Insights: charged per GB scanned
- Alarm evaluation periods: minimum 10 seconds (high resolution), minimum 1 minute (standard)
- Maximum dimensions per metric: 30
- Maximum alarms per account: 5,000 (soft limit)
- Log group retention: 1 day to 10 years, or never expire

### Edge Cases and Gotchas

- Default log retention is "never expire." This is expensive. Set retention policies proactively.
- Custom metrics are expensive at scale. Avoid high-cardinality dimensions (e.g., request ID as a dimension).
- CloudWatch metrics from EC2 do not include memory or disk utilization by default. You must install the CloudWatch Agent for these.
- Alarm evaluation uses the metric's period. If a metric publishes at 5-minute intervals and you set a 1-minute alarm period, you get INSUFFICIENT_DATA states.
- Cross-account and cross-region dashboards require setup (CloudWatch cross-account observability).
- Log group names cannot be changed after creation.

### Common Interview Questions

**Q: How do you monitor Lambda functions effectively?**
A: Use built-in CloudWatch metrics (Invocations, Duration, Errors, Throttles, ConcurrentExecutions). Set alarms on Error rate and Throttle rate. Use CloudWatch Logs Insights to query function logs. Use Lambda Insights (an extension) for enhanced metrics (memory utilization, cold starts). Use X-Ray for distributed tracing.

---

## AWS X-Ray

### Overview

X-Ray provides distributed tracing for microservice architectures. It tracks requests as they flow through your application, across Lambda functions, API Gateway, DynamoDB, SQS, SNS, and other services. X-Ray creates a visual service map showing how services are connected and where latency or errors occur.

### How It Works

X-Ray uses a sampling rule to decide which requests to trace (not every request is traced, to reduce overhead and cost). The default sampling rule traces the first request each second and 5% of additional requests.

A **trace** represents an end-to-end request. It consists of **segments** (one per service) and **subsegments** (subdivisions within a segment, e.g., a DynamoDB call within a Lambda function).

**Annotations** are indexed key-value pairs that you can search and filter on. Use annotations for values you want to query (customer ID, order status).

**Metadata** is non-indexed key-value data for additional context. Use metadata for large or detailed information you do not need to search on.

### Sampling Rules

Custom sampling rules let you control which requests are traced. You define rules with a priority, a matching expression (service name, HTTP method, URL path), a fixed rate (first N requests per second), and a dynamic rate (percentage of remaining requests).

### Limits and Quotas

- Maximum trace size: 500 KB
- Trace data retention: 30 days
- Maximum annotations per segment: 50
- Sampling: customizable rules with reservoir (fixed rate) and rate (percentage)

### Edge Cases and Gotchas

- X-Ray adds a small amount of latency to each traced request. For ultra-low-latency services, adjust sampling rules to trace fewer requests.
- Not all AWS services support active X-Ray tracing. Lambda and API Gateway support it natively; others require the X-Ray SDK.
- X-Ray trace context is propagated via headers. If your application communicates over non-HTTP protocols, you must propagate the trace context manually.
- The X-Ray SDK must be included in your application code. For Lambda, you can enable active tracing without code changes, but subsegments for downstream calls require the SDK.

---

## AWS CloudTrail

### Overview

CloudTrail records API calls made in your AWS account. Every time someone (user, role, or service) calls an AWS API, CloudTrail logs it. This provides an audit trail for security analysis, compliance, and troubleshooting.

### Management Events vs Data Events

**Management events** (control plane): Actions that manage AWS resources — CreateBucket, RunInstances, PutRolePolicy. These are logged by default at no additional charge (90-day event history). For longer retention, create a trail that delivers to S3.

**Data events** (data plane): Actions that operate on data within resources — GetObject/PutObject on S3, GetItem/PutItem on DynamoDB, Invoke on Lambda. Data events are high-volume and are NOT logged by default. You must explicitly enable them per trail, and they cost $0.10 per 100,000 events. For a high-traffic application, data event logging can be expensive.

### Organization Trails

If your account is in an AWS Organization, you can create an organization trail that logs events from all accounts in the organization to a single S3 bucket. This centralizes audit logging.

### Limits and Quotas

- Event history: 90 days (free, management events only)
- Trails: deliver to S3 and/or CloudWatch Logs
- Data events cost: $0.10 per 100,000 events
- Management events: first copy free per region, additional copies charged
- Maximum trails per region: 5

### Edge Cases and Gotchas

- Data events are not logged by default and can be very expensive for high-volume services. Calculate the cost before enabling.
- CloudTrail logs have a delivery delay of approximately 5-15 minutes. They are not real-time.
- CloudTrail logs are delivered to S3 as JSON files. To query them easily, use Athena with a CloudTrail-optimized table.
- CloudTrail does not log every API call — some read-only APIs and some service-internal calls are excluded.
- Ensure CloudTrail logs are protected from tampering: enable log file validation, restrict S3 bucket access, and use CloudTrail Lake or S3 Object Lock for immutable storage.

---

## AWS KMS (Key Management Service)

### Overview

KMS manages cryptographic keys for encrypting data across AWS services. Most AWS services integrate with KMS for encryption at rest and in transit.

### Symmetric vs Asymmetric Keys

**Symmetric keys** (AES-256-GCM): The same key encrypts and decrypts. The key material never leaves KMS — all encrypt/decrypt operations happen within the KMS service. This is the default and most common type.

**Asymmetric keys** (RSA, ECC): Public/private key pairs. The public key can be downloaded for use outside AWS. Use cases: digital signatures, encryption by external parties, TLS certificate private keys.

### Envelope Encryption

For large data, KMS uses envelope encryption:

1. Your application calls KMS GenerateDataKey
2. KMS returns a plaintext data key and an encrypted (ciphertext) copy of the data key
3. Your application encrypts the data with the plaintext data key
4. Your application stores the encrypted data + the encrypted data key together
5. The plaintext data key is discarded from memory

To decrypt: send the encrypted data key to KMS, get back the plaintext data key, decrypt the data.

This pattern avoids sending large data to KMS (which has a 4 KB limit on Encrypt/Decrypt API calls) and reduces KMS API calls (one call to generate/decrypt the key, then local encryption/decryption of the data).

### Key Rotation

AWS managed keys are automatically rotated every year (you cannot change this). Customer managed keys can be configured for automatic rotation (every 90 days to 7 years). Key rotation creates a new backing key but keeps the same key ID and ARN. Old backing keys are retained so that previously encrypted data can still be decrypted.

### Limits and Quotas

- Encrypt/Decrypt/GenerateDataKey: 5,500 to 30,000 requests per second per key (varies by key type and region)
- Maximum data size for direct Encrypt/Decrypt: 4 KB
- Key policies: 32 KB maximum size
- Keys per account: 100,000 (soft limit)

### Edge Cases and Gotchas

- KMS has request rate limits per key. If multiple services (S3, DynamoDB, Lambda, SQS) share the same KMS key, aggregate their KMS API calls and ensure you do not hit the limit. Request a limit increase proactively for high-throughput workloads.
- Deleting a KMS key has a mandatory 7-30 day waiting period. Once deleted, all data encrypted with that key is permanently unrecoverable.
- Cross-region: KMS keys are regional. To use encrypted data in another region, you must re-encrypt with a key in the destination region (or use multi-region keys).
- Multi-region keys replicate key material across regions but cost more and have additional management overhead.
- KMS calls add latency (typically 5-20ms). For latency-sensitive applications, use local caching of data keys (e.g., the AWS Encryption SDK's data key cache).

---

## AWS Secrets Manager vs. Systems Manager Parameter Store

Both services store configuration and secrets, but they have different features and pricing models.

### SSM Parameter Store

- **Two tiers**: Standard (free for up to 10,000 parameters) and Advanced (paid).
- **Features**: Hierarchy support (e.g., `/app/prod/db-url`), parameter versioning, integration with CloudFormation and Lambda.
- **Secrets**: Supports "SecureString" which is encrypted with a KMS key.
- **Rotation**: Does NOT support automatic rotation natively (requires manual Lambda integration).
- **Use case**: General configuration, feature flags, and secrets that do not require automatic rotation.

### AWS Secrets Manager

- **Pricing**: $0.40 per secret per month + $0.05 per 10,000 API calls (expensive).
- **Features**: Automatic rotation for RDS, Redshift, and DocumentDB (using built-in Lambda functions), versioning, cross-region replication of secrets.
- **Rotation**: Built-in support for rotating secrets without application downtime.
- **Use case**: Database credentials, API keys for third-party services, and anything requiring automatic rotation.

### Comparison Table

| Feature | SSM Parameter Store | AWS Secrets Manager |
| :--- | :--- | :--- |
| **Cost** | Free (Standard) | $0.40/secret/month (Paid) |
| **Encryption** | Optional (SecureString) | Mandatory |
| **Rotation** | Manual (via Lambda) | Automatic (Built-in) |
| **Cross-Account** | Difficult | Supported |
| **Cross-Region** | No | Yes (Replication) |

---

## Amazon VPC (Virtual Private Cloud)

### Overview

A VPC is a logically isolated section of the AWS cloud where you launch AWS resources. You control the network configuration: IP ranges, subnets, route tables, network gateways, and security settings.

### Subnet Design Patterns

A well-designed VPC typically has three subnet tiers per AZ:

**Public subnets**: Have a route to an Internet Gateway. Resources here get public IPs and can be reached from the internet. Use for: ALB/NLB, NAT Gateways, bastion hosts (if you still use them). Avoid placing application workloads directly in public subnets.

**Private subnets**: No route to an Internet Gateway. Resources cannot be reached from the internet, but can reach the internet via NAT Gateway. Use for: Lambda functions, ECS tasks, Aurora, DocumentDB, application servers. This is where most workloads run.

**Isolated subnets**: No route to the internet at all — no NAT Gateway, no Internet Gateway. Resources can only communicate within the VPC (or via VPC endpoints). Use for: Databases that should never talk to the internet, sensitive workloads in regulated environments.

Best practice: deploy subnets in at least 2 AZs (3 for production). Size subnets generously — you cannot resize a subnet after creation. Use /20 or /19 for private subnets to allow for growth.

### Security Groups vs Network ACLs (NACLs)

These are two layers of network security that operate differently:

**Security Groups** (instance-level firewall):
- **Stateful**: If you allow inbound traffic, the return outbound traffic is automatically allowed (and vice versa)
- **Allow rules only**: You cannot create deny rules. You specify what is allowed; everything else is implicitly denied.
- **Evaluation**: All rules are evaluated — if any rule allows the traffic, it is permitted
- **Scope**: Attached to ENIs (instances, Lambda in VPC, RDS, etc.)
- **Default**: New security groups deny all inbound, allow all outbound

**Network ACLs** (subnet-level firewall):
- **Stateless**: You must explicitly allow both inbound and outbound traffic (including return traffic with ephemeral ports)
- **Allow and deny rules**: You can create both allow and deny rules
- **Evaluation**: Rules are evaluated in order by rule number (lowest first). First matching rule wins; remaining rules are not evaluated.
- **Scope**: Applied to all traffic entering or leaving a subnet
- **Default**: Default NACL allows all inbound and outbound. Custom NACLs deny all by default.

In practice, most teams rely on security groups and leave the default (allow-all) NACL in place. NACLs are useful when you need explicit deny rules (blocking a specific IP range) or when compliance requires defense-in-depth at the subnet level.

Common gotcha: NACLs require you to allow ephemeral port ranges (1024-65535) for return traffic. If you block these, responses from web servers, databases, and other services fail silently.

### Private Subnets for Lambda/Aurora

Lambda functions attached to a VPC and Aurora instances should be placed in private subnets (no direct internet access). Private subnets do not have a route to an Internet Gateway.

For Lambda in a private subnet to access the internet (to call external APIs or AWS services), you need either:
- A NAT Gateway in a public subnet (costs $0.045/hr + $0.045/GB processed)
- VPC endpoints for specific AWS services (see below)

### VPC Endpoints

VPC endpoints allow private connectivity from your VPC to AWS services without traversing the internet.

**Gateway endpoints**: Free. Available only for S3 and DynamoDB. Created as a route table entry pointing to the endpoint. No additional ENIs or DNS changes needed.

**Interface endpoints** (PrivateLink): Cost $0.01/hr per AZ + $0.01/GB processed. Available for most AWS services (Lambda, SQS, SNS, KMS, Secrets Manager, CloudWatch, etc.). Create ENIs in your subnet. Use private DNS to resolve service endpoints to the private IP.

### VPC Connectivity: Peering vs. Transit Gateway

**VPC Peering**: Connects two VPCs as if they were on the same network. No transitiveness (if A peers with B and B with C, A cannot reach C). Good for simple 1-to-1 connections.

**AWS Transit Gateway**: A hub-and-spoke network transit hub to connect VPCs and on-premises networks.
- **Transitive Routing**: VPC A can reach VPC C through the Transit Gateway.
- **Complexity Management**: Simplifies network architecture as the number of VPCs grows (hub-and-spoke vs. mesh of peering).
- **Cost**: $0.05/hr per VPC attachment + $0.02/GB processed.

**AWS PrivateLink**: Private connectivity between VPCs, AWS services, and on-premises networks without exposing traffic to the public internet. You can create a "Service Provider" VPC and "Consumer" VPCs can connect to it via Interface Endpoints. This is the preferred way to share services across accounts securely.

### VPC Flow Logs

VPC Flow Logs capture metadata about IP traffic going to and from network interfaces in your VPC. They do not capture packet contents — only metadata (source/dest IP, port, protocol, action, bytes, packets).

Flow logs can be published to CloudWatch Logs, S3, or Kinesis Data Firehose. They can be enabled at three levels: VPC (all ENIs in the VPC), subnet (all ENIs in the subnet), or network interface (specific ENI).

Use cases:
- **Security**: Detect unexpected traffic patterns, identify compromised instances communicating with suspicious IPs
- **Troubleshooting**: Diagnose connectivity issues (see rejected traffic that indicates security group or NACL misconfigurations)
- **Compliance**: Audit network access for regulatory requirements
- **Cost analysis**: Identify top talkers and unexpected cross-AZ traffic

Important: Flow logs have a capture delay of approximately 10 minutes (not real-time). They do not capture traffic to/from the VPC metadata service (169.254.169.254), DHCP traffic, or traffic to the VPC DNS server.

For cost-effective storage, publish to S3 in Parquet format and query with Athena. For real-time alerting, publish to CloudWatch Logs and create metric filters.

### PrivateLink Deep Dive

PrivateLink enables private connectivity between VPCs, AWS services, and your on-premises networks without exposing traffic to the public internet. It works in two directions:

**Consuming AWS services** (Interface VPC Endpoints): Your VPC creates an ENI that represents the AWS service. Traffic to the service stays on the AWS network, never traversing the internet. Use for: SQS, SNS, KMS, Secrets Manager, CloudWatch, ECR, and 100+ other services.

**Exposing your own services** (Endpoint Services): You create a Network Load Balancer in your VPC and register it as an endpoint service. Other VPCs (same or different accounts) create interface endpoints to connect to your service. This enables a service-provider/service-consumer model without VPC peering or Transit Gateway.

PrivateLink is the recommended pattern for sharing microservices across accounts or with external partners. The consumer does not need to know the provider's IP range, and the provider does not need to allow the consumer's IP range. Traffic flows through AWS-managed tunnels.

Key detail: PrivateLink is unidirectional. The consumer can call the provider's service, but the provider cannot initiate connections back to the consumer. This is a security feature.

### Hybrid Connectivity: VPN and Direct Connect

**AWS Site-to-Site VPN**: Encrypted IPSec tunnel over the public internet between your on-premises network and your VPC. Quick to set up (minutes), low cost ($0.05/hr per connection), but limited by internet bandwidth and latency (typically 50-500ms). Each VPN connection provides two tunnels for redundancy.

**AWS Direct Connect**: Dedicated physical network connection between your data center and AWS. Provides consistent latency (typically 1-10ms), high bandwidth (1 Gbps, 10 Gbps, or 100 Gbps), and does not traverse the public internet. Takes weeks to set up (physical cross-connect in a colocation facility). Costs per port-hour plus per-GB data transfer.

**Decision**: Use VPN for quick setup, lower cost, or as a backup for Direct Connect. Use Direct Connect for consistent performance, high bandwidth, or when encryption over the internet is not acceptable. Many production setups use Direct Connect as primary with VPN as backup failover.

Both VPN and Direct Connect can terminate at a Transit Gateway for centralized routing to multiple VPCs.

### DNS: Route 53 Resolver and Private Hosted Zones

**Private Hosted Zones**: Route 53 hosted zones that are only resolvable from within associated VPCs. Use for internal service discovery (e.g., `api.internal.company.com` resolves to a private IP). Private hosted zones can be shared across VPCs and accounts using Route 53 Resolver rules or VPC associations.

**Route 53 Resolver**: Provides DNS resolution between your VPC and on-premises networks.

- **Inbound endpoints**: Allow on-premises DNS resolvers to forward queries to Route 53 (e.g., on-premises servers resolve `api.internal.company.com` via Route 53)
- **Outbound endpoints**: Allow Route 53 to forward queries to on-premises DNS resolvers (e.g., EC2 instances resolve `legacy.corp.company.com` via your corporate DNS)
- **Resolver rules**: Define which domain queries are forwarded where

This is essential for hybrid architectures where some services run in AWS and some on-premises. Without Resolver, VPC resources cannot resolve on-premises DNS names and vice versa.

### NAT Gateway vs NAT Instance vs VPC Endpoints: Decision Matrix

**NAT Gateway**: Managed, highly available within an AZ, scales to 100 Gbps. Expensive ($0.045/hr + $0.045/GB). Use for general internet access from private subnets. Deploy one per AZ for HA.

**NAT Instance**: Self-managed EC2 instance running NAT. Cheaper for low traffic (t3.micro costs ~$0.01/hr). You manage patching, scaling, and HA. Not recommended for production — use NAT Gateway instead.

**VPC Endpoints**: Private connectivity to specific AWS services. Gateway endpoints (S3, DynamoDB) are free. Interface endpoints cost $0.01/hr/AZ + $0.01/GB but avoid NAT Gateway data processing charges. Use for all AWS service traffic from private subnets — much cheaper than routing through NAT Gateway.

The cost optimization pattern: use VPC gateway endpoints for S3 and DynamoDB (free), interface endpoints for other heavily-used AWS services, and NAT Gateway only for traffic that must reach the public internet (external APIs, third-party services). This can reduce NAT Gateway costs by 80%+.

### Data Transfer Costs

Data transfer is one of the least understood and most impactful AWS costs:

- **Same AZ, same service**: Free
- **Cross-AZ (within region)**: $0.01/GB in each direction ($0.02/GB round trip)
- **Cross-region**: $0.02/GB (varies by region pair)
- **Internet egress**: $0.09/GB for the first 10 TB/month (decreasing tiers after)
- **NAT Gateway**: $0.045/GB processed (in addition to the underlying data transfer)
- **VPC endpoint (interface)**: $0.01/GB processed

NAT Gateway is particularly expensive because you pay for the NAT Gateway hourly rate, the per-GB processing fee, AND the underlying data transfer charge. For high-throughput applications, VPC endpoints are much cheaper than NAT Gateway for AWS service traffic.

### Limits and Quotas

- VPCs per region: 5 (soft limit)
- Subnets per VPC: 200
- Security groups per VPC: 2,500
- Rules per security group: 60 inbound + 60 outbound
- Network ACL rules: 20 inbound + 20 outbound
- Elastic IPs per region: 5 (soft limit)

### Edge Cases and Gotchas

- NAT Gateway is a single point of failure within an AZ. For high availability, create a NAT Gateway in each AZ with routing from that AZ's private subnets.
- VPC peering does not support transitive routing. If VPC A peers with B and B peers with C, A cannot reach C through B. Use Transit Gateway for hub-and-spoke.
- Security groups are stateful (return traffic is automatically allowed). Network ACLs are stateless (you must explicitly allow return traffic on ephemeral ports 1024-65535).
- DNS resolution within a VPC requires enableDnsHostnames and enableDnsSupport to be true. Without these, private hosted zones and VPC endpoints do not resolve correctly.
- VPC endpoint policies can restrict which resources are accessible through the endpoint — use them for additional security (e.g., restrict S3 endpoint to only your buckets).
- Data transfer costs can surprise you. Cross-AZ traffic for high-throughput services (e.g., RDS read replicas in a different AZ) adds up at $0.01/GB each way.
- Subnets cannot be resized after creation. Over-allocate CIDR ranges at the start. You can add secondary CIDR blocks to a VPC but not to an existing subnet.
- VPC CIDR ranges cannot overlap with peered VPCs or on-premises networks. Plan your IP addressing before creating VPCs.
- Lambda in a VPC with no NAT Gateway or VPC endpoints times out silently when trying to call AWS services. The error looks like a function timeout, not a network error — this is one of the most common Lambda debugging traps.
- Security groups can reference other security groups (e.g., "allow traffic from any instance in SG-web"). This is more maintainable than listing IP ranges and updates automatically as instances are added/removed.
- Default VPC exists in every region and has public subnets with internet access. Do not use the default VPC for production — create a custom VPC with proper subnet design.

### Common Interview Questions

**Q: How do you design a VPC for a production serverless application?**
A: Create a VPC with at least 2 AZs (3 for HA). Each AZ gets a public subnet (for ALB, NAT Gateway), a private subnet (for Lambda, ECS, Aurora), and optionally an isolated subnet (for databases that should never reach the internet). Use gateway endpoints for S3 and DynamoDB (free), interface endpoints for other AWS services, and NAT Gateway only for external API calls. Attach security groups to each resource with least-privilege rules.

**Q: What is the difference between VPC peering, Transit Gateway, and PrivateLink?**
A: VPC peering connects two VPCs directly (no transitivity, no extra hops, low cost). Transit Gateway is a hub that connects multiple VPCs and on-premises networks with transitive routing (centralized management, additional cost). PrivateLink exposes a specific service from one VPC to another via a private endpoint (unidirectional, service-level granularity, no IP overlap concerns). Use peering for simple 1:1 connections, Transit Gateway for multi-VPC networks, and PrivateLink for sharing services across accounts.

**Q: How do you troubleshoot a Lambda function that times out when calling DynamoDB from a VPC?**
A: Lambda in a VPC has no internet access by default. DynamoDB is a public AWS service — Lambda needs a network path to reach it. Check if there is a DynamoDB gateway endpoint in the route table for the Lambda's subnets. If not, create one (free). Alternatively, create a NAT Gateway (expensive). Also verify the security group on the Lambda function allows outbound HTTPS (port 443) traffic.

---

## Elastic Load Balancing (ELB)

### Application Load Balancer (ALB)

ALB operates at Layer 7 (HTTP/HTTPS) and routes traffic based on content (path, host, headers).
- **Target Groups**: Groups of EC2 instances, Lambda functions, or IP addresses (ECS tasks).
- **Rules**: Define how to route traffic to Target Groups.
- **Features**: WAF integration, SSL termination, Sticky sessions, and WebSocket support.
- **Latency**: Adds a few milliseconds of latency as it inspects the request body.

### Network Load Balancer (NLB)

NLB operates at Layer 4 (TCP/UDP/TLS) and is designed for ultra-high performance and low latency.
- **Static IPs**: Provides one static IP address per AZ (ALB's IPs change over time).
- **Performance**: Handles millions of requests per second with microsecond latency.
- **Use case**: Ultra-high performance, non-HTTP protocols, and cases requiring static IP addresses.

---

## Amazon Route 53

Route 53 is a highly available and scalable DNS web service.

### Routing Policies

- **Simple**: Standard DNS record (one record to one or more IP addresses). No health checks.
- **Weighted**: Distribute traffic across multiple resources based on weights (e.g., 80% to Version A, 20% to Version B).
- **Latency**: Route traffic to the region that provides the lowest latency for the user.
- **Failover**: Active-Passive failover. Route 53 monitors a health check and switches to the passive resource if the active one fails.
- **Geolocation**: Route traffic based on the user's physical location (e.g., European users go to eu-central-1).
- **Geoproximity**: Route traffic based on geographic proximity to your resources (requires Route 53 Traffic Flow).
- **Multivalue Answer**: Returns up to 8 healthy records randomly. A basic form of load balancing.

### Health Checks

Route 53 can monitor the health of your resources (HTTP, HTTPS, TCP). If a health check fails, Route 53 stops returning that resource in DNS queries.

### Alias Records

Alias records are a Route 53-specific extension to DNS. They can point to AWS resources (ALB, CloudFront, S3 bucket) directly.
- **Cost**: Queries to Alias records are free.
- **CNAME vs Alias**: Use Alias for the zone apex (e.g., `example.com`). Standard CNAMEs cannot be used at the zone apex.

---

## Amazon API Gateway

### Overview

API Gateway is a managed service for creating, publishing, and managing APIs. It supports REST APIs, HTTP APIs, and WebSocket APIs.

### REST API vs HTTP API

**REST API**: Full-featured. Supports request validation, request/response transformation, WAF integration, caching, API keys, usage plans, resource policies, edge-optimized deployments, and mutual TLS. More expensive.

**HTTP API**: Simpler and cheaper (up to 71% less expensive than REST API). Supports JWT authorizers, Lambda authorizers, CORS, and most common API patterns. Does not support request transformation, caching, WAF, API keys, or resource policies. Use HTTP API for most new workloads unless you need REST API-specific features.

### Timeout

API Gateway has a maximum integration timeout of 29 seconds. If your backend (Lambda, HTTP endpoint) takes longer than 29 seconds to respond, API Gateway returns a 504 Gateway Timeout to the client. This cannot be increased.

For long-running operations, use an asynchronous pattern: accept the request, return a 202 with a task ID, and have the client poll for results or use WebSocket for push notifications.

### Payload Size

Maximum payload size is 10 MB. For larger payloads, use S3 presigned URLs for upload/download.

### Limits and Quotas

- Maximum integration timeout: 29 seconds
- Maximum payload size: 10 MB
- Throttle: 10,000 requests/second (account-level, across all APIs, soft limit)
- Burst: 5,000 requests (account-level)
- Maximum APIs per region: 600

### Edge Cases and Gotchas

- The 29-second timeout is a hard limit. Design for async patterns for long-running operations.
- REST API is significantly more expensive than HTTP API. Use HTTP API unless you specifically need REST API features.
- Binary payloads require explicit content type configuration.
- CORS must be configured explicitly on both the API Gateway and the backend.
- Lambda proxy integration sends the entire request to Lambda (headers, query parameters, body). Non-proxy integration allows request mapping but adds complexity.
- Stage variables and deployment model differences between REST and HTTP APIs can be confusing.

---

## AWS AppSync

AppSync is a managed GraphQL service that simplifies application development by letting you create a flexible API to securely access, manipulate, and combine data from one or more data sources.

### Key Features

- **GraphQL Support**: Real-time updates with GraphQL Subscriptions.
- **Data Sources**: Direct integration with DynamoDB, Lambda, Aurora (Serverless), and Elasticsearch/OpenSearch.
- **Resolvers**: Map GraphQL fields to data source operations.
- **Real-time**: Built-in support for WebSockets to push data to clients.
- **Caching**: Built-in server-side caching to reduce load on data sources.

### AppSync vs. API Gateway

| Feature | API Gateway | AppSync |
| :--- | :--- | :--- |
| **Protocol** | REST, HTTP, WebSocket | GraphQL |
| **Interface** | Path-based (e.g., `/users/1`) | Schema-based (Queries, Mutations) |
| **Real-time** | Custom via WebSockets | Built-in via Subscriptions |
| **Data Fetching** | Client may need multiple calls | Client gets exactly what it needs in one call |
| **Caching** | Available (REST API) | Available |

Use **AppSync** when you need GraphQL, complex data relationships, or real-time updates for a mobile/web app. Use **API Gateway** for standard REST/HTTP APIs or when you need full control over the HTTP request/response lifecycle.

---

## Amazon Cognito

### Overview

Cognito provides authentication and authorization for web and mobile applications. It has two main components:

**User Pools**: A user directory for sign-up and sign-in. Supports username/password, social login (Google, Facebook, Apple), SAML, and OIDC federation. Issues JWT tokens (ID, access, refresh). Think of User Pools as your authentication provider.

**Identity Pools** (Federated Identities): Provides temporary AWS credentials to users authenticated via User Pools, social providers, or SAML. Maps authenticated users to IAM roles. Think of Identity Pools as the bridge between authentication and AWS authorization.

Common pattern: User signs in via User Pool (gets JWT) -> Identity Pool exchanges JWT for temporary AWS credentials -> user accesses AWS resources (S3, DynamoDB) directly with those credentials.

### Edge Cases and Gotchas

- Cognito User Pools have a hard limit of 40 custom attributes per user pool.
- The free tier is 50,000 monthly active users for User Pools (ESSENTIALS tier) — generous for most applications.
- Advanced security features (adaptive authentication, compromised credentials detection) are available at additional cost.
- Cognito custom domains require an ACM certificate in us-east-1.
- Token revocation is supported but not instantaneous — cached tokens may remain valid until expiration.

---

## Amazon CloudFront

### Overview

CloudFront is AWS's global CDN. It caches content at edge locations worldwide, reducing latency for end users. It supports static content (S3), dynamic content (API Gateway, ALB, custom origins), and streaming.

### OAC (Origin Access Control)

OAC restricts S3 bucket access to only CloudFront, preventing direct S3 access. OAC replaces the older OAI (Origin Access Identity). OAC supports SSE-KMS encrypted objects and POST/PUT requests, which OAI did not.

### Signed URLs vs Signed Cookies

**Signed URLs**: Restrict access to individual files. Each URL contains an access policy and signature. Use when you want to restrict access to specific files.

**Signed Cookies**: Restrict access to multiple files with a single cookie. Use when you want to restrict access to an entire section of a site (e.g., a paid video library) without modifying individual URLs.

### When to Use CloudFront

- Serving static assets (S3) globally with low latency
- Reducing load on origin servers
- DDoS protection (CloudFront + AWS Shield)
- Custom SSL certificates for your domain
- Edge computing (CloudFront Functions, Lambda@Edge)
- Reducing S3 costs (CloudFront data transfer is cheaper than direct S3 egress)

### Edge Cases and Gotchas

- CloudFront caches error responses too. A 503 from your origin can be cached and served to all users. Set appropriate error caching TTLs.
- Cache invalidation costs $0 for the first 1,000 paths/month, then $0.005 per path. Use versioned file names instead of invalidation when possible.
- Default TTL is 24 hours. If your content changes frequently, lower the TTL or use cache-control headers.
- Lambda@Edge has cold starts and runs in a single region (replicated to edges), while CloudFront Functions run at the edge with sub-millisecond startup but have limited capabilities.

---

## Amazon Athena

### Overview

Athena is a serverless interactive query service that analyzes data in S3 using standard SQL. It uses Presto/Trino under the hood. No data loading or ETL required — you define a schema over your S3 data and query it.

### Pricing

Athena charges $5 per TB of data scanned. This makes optimization critical:

### Optimization Strategies

- **Columnar formats**: Use Parquet or ORC instead of CSV/JSON. Columnar formats store data by column, so Athena only reads the columns needed for the query (column pruning). This can reduce data scanned by 90%+.
- **Partition pruning**: Partition your data in S3 by commonly filtered fields (e.g., `s3://bucket/year=2024/month=03/day=15/`). Athena reads only the partitions matching your WHERE clause.
- **Compression**: Compress files to reduce S3 storage and data scanned. Use Snappy or GZIP with Parquet.
- **File size**: Use files between 128 MB and 1 GB for optimal performance. Too many small files causes overhead; too few large files limits parallelism.

### Limits and Quotas

- Maximum query execution time: 30 minutes (configurable up to 8 hours)
- Maximum query string length: 262,144 bytes
- Maximum concurrent queries: 25-50 per account (soft limit)
- Maximum databases: 10,000 per catalog

### Edge Cases and Gotchas

- Querying uncompressed CSV/JSON is expensive. Always convert to Parquet and partition.
- Athena is not suitable for transactional queries or low-latency lookups. It is for analytical/ad-hoc queries.
- Schema-on-read means Athena does not validate data quality. Malformed rows cause query errors.
- Cost is per query. A poorly written query scanning the entire dataset repeatedly can be expensive. Use partitions and columnar formats to minimize scans.
- Athena results are stored in S3. Ensure your results bucket has appropriate access controls and lifecycle policies.

### Common Interview Questions

**Q: How do you optimize Athena costs?**
A: Three main strategies: (1) Use columnar formats (Parquet/ORC) for column pruning, (2) Partition data by common query filters for partition pruning, (3) Compress data to reduce bytes scanned. Together, these can reduce costs by 90-99% compared to querying raw CSV/JSON.

---

## Security Services

### AWS WAF (Web Application Firewall)

WAF protects web applications from common exploits. It integrates with CloudFront, ALB, API Gateway, and AppSync.

**Web ACLs**: A collection of rules that define what to allow, block, or count. Rules are evaluated in order of priority.

**Rule types**:
- **Rate-based rules**: Block IPs that exceed a request threshold (e.g., 2,000 requests per 5 minutes). This is basic DDoS and brute-force protection.
- **IP set rules**: Allow or block specific IP ranges.
- **Regex and string match rules**: Inspect headers, query strings, body, or URI path for patterns.
- **Managed rule groups**: Pre-built rule sets maintained by AWS or third-party vendors. AWS Managed Rules include: Core rule set (XSS, SQL injection), Known bad inputs, Bot Control, Account takeover prevention, and IP reputation lists.

**Common patterns**: Block SQL injection and XSS with the Core rule set. Rate-limit login endpoints. Geo-restrict to specific countries. Block known bad bots.

Pricing: $5/month per web ACL + $1/month per rule + $0.60 per million requests inspected. Managed rule groups have additional per-group charges. Bot Control is expensive ($10/month + $1/million requests).

Gotchas: WAF rules are evaluated in order — put rate-based rules early to block floods before expensive regex rules run. WAF on API Gateway REST API works, but WAF on HTTP API is not supported (use CloudFront in front of HTTP API for WAF). WAF does not inspect the full body by default — only the first 8 KB (configurable up to 64 KB).

### AWS Shield

Shield provides DDoS protection.

**Shield Standard**: Free, automatic, included for all AWS customers. Protects against common Layer 3/4 attacks (SYN floods, UDP reflection) on CloudFront, Route 53, and ALB. No configuration needed.

**Shield Advanced**: $3,000/month per organization (expensive). Provides: advanced detection for Layer 7 attacks, real-time metrics and reporting, DDoS cost protection (credits for scaling charges caused by DDoS), access to the AWS DDoS Response Team (DRT) for 24/7 assistance during attacks, and WAF fees included (no additional WAF charges for protected resources).

Decision: Shield Standard is sufficient for most applications. Shield Advanced is for high-profile targets that need guaranteed DDoS response SLA, cost protection, and the DRT. The $3,000/month price makes it a serious commitment.

### Amazon GuardDuty

GuardDuty is a threat detection service that continuously monitors for malicious activity and unauthorized behavior. It analyzes:

- **CloudTrail management and data events**: Detects unusual API calls, unauthorized access patterns
- **VPC Flow Logs**: Detects communication with known malicious IPs, unusual traffic patterns
- **DNS logs**: Detects communication with known command-and-control domains
- **S3 data events**: Detects suspicious S3 access patterns (data exfiltration)
- **EKS audit logs**: Detects suspicious Kubernetes API activity
- **Lambda network activity**: Detects suspicious outbound connections from Lambda functions

GuardDuty uses machine learning and threat intelligence to generate findings (severity levels: low, medium, high). Findings can be sent to EventBridge for automated remediation (e.g., EventBridge -> Lambda -> revoke security group, disable IAM user).

Pricing: Based on volume of data analyzed. Free 30-day trial. Approximately $4/million CloudTrail events, $1/GB for VPC Flow Logs and DNS logs.

Key value: GuardDuty is one of the first security services you should enable in every account. It requires zero configuration and immediately starts detecting threats. It works across accounts via Organizations integration.

### AWS Security Hub

Security Hub aggregates security findings from GuardDuty, Inspector, Macie, Firewall Manager, IAM Access Analyzer, and third-party tools into a single dashboard. It provides:

- **Automated compliance checks**: Continuous evaluation against security standards (CIS Benchmarks, PCI DSS, AWS Foundational Security Best Practices)
- **Centralized findings**: All findings in one place with severity scoring and prioritization
- **Cross-account aggregation**: Via Organizations, aggregate findings from all accounts into a delegated admin account
- **Automated remediation**: Findings trigger EventBridge rules for automated response

Security Hub is the "single pane of glass" for security posture. Enable it in all accounts, enable all security standards, and route high-severity findings to your incident response process.

### AWS Config

Config continuously records and evaluates the configuration of your AWS resources. It answers: "What does my resource look like now?", "What did it look like before?", and "Does it comply with my rules?"

**Config Rules**: Evaluate whether resource configurations comply with your policies. AWS provides 300+ managed rules (e.g., "S3 buckets must have encryption enabled," "Security groups must not allow 0.0.0.0/0 on SSH"). You can also write custom rules using Lambda.

**Conformance Packs**: Bundles of Config rules and remediation actions for a specific compliance framework (PCI DSS, HIPAA, CIS Benchmarks). Deploy a conformance pack and Config continuously evaluates compliance.

**Configuration history**: Config keeps a history of every configuration change for every recorded resource. You can query "what changed on this resource and when." This is essential for incident investigation and change tracking.

**Auto-remediation**: Config rules can trigger automatic remediation actions via SSM Automation documents. For example, if a security group is opened to 0.0.0.0/0, Config can automatically revoke the rule.

Pricing: $0.003 per configuration item recorded + $0.001 per Config rule evaluation. Can be expensive with many resources and frequent changes.

### Security Service Integration Pattern

The recommended security stack for production AWS accounts:

1. **GuardDuty** in every account — threat detection (enable first)
2. **Config** with managed rules — compliance monitoring
3. **Security Hub** — aggregate findings from all services
4. **CloudTrail** — organization trail for audit logging
5. **IAM Access Analyzer** — identify external/unused access
6. **WAF** on internet-facing endpoints — application-layer protection
7. **EventBridge** — automated remediation triggered by security findings

All of these services support Organizations-level deployment, so you enable them once and they propagate to all accounts.

### Common Interview Questions

**Q: How do you build a security monitoring pipeline on AWS?**
A: Enable GuardDuty for threat detection, Config for compliance monitoring, and Security Hub to aggregate findings. Route high-severity findings from Security Hub to EventBridge, which triggers Lambda functions for automated remediation (isolate instances, revoke credentials, notify team via SNS). Store all findings in S3 for long-term analysis with Athena. Enable CloudTrail organization trail for complete audit logging.

**Q: What is the difference between WAF, Shield, and GuardDuty?**
A: WAF inspects and filters HTTP/HTTPS traffic at the application layer (Layer 7) — SQL injection, XSS, rate limiting. Shield protects against volumetric DDoS attacks at the network/transport layer (Layer 3/4). GuardDuty is a detective control — it does not block anything but detects threats by analyzing CloudTrail, VPC Flow Logs, and DNS logs. Use all three together: Shield for DDoS, WAF for application exploits, GuardDuty for threat detection.

**Q: How do you enforce compliance across 50 AWS accounts?**
A: Use AWS Organizations with SCPs for preventive guardrails (deny non-compliant actions). Deploy AWS Config conformance packs for detective controls (evaluate resource configurations against compliance rules). Aggregate all findings in Security Hub with a delegated admin account. Use Config auto-remediation for automatic fixes. Enable CloudTrail organization trail and GuardDuty across all accounts for detection and audit.

---

## Observability: The Full Picture

CloudWatch, X-Ray, and CloudTrail together form the three pillars of AWS observability. Understanding how they complement each other is a common interview topic.

### Three Pillars

**Metrics (CloudWatch Metrics)**: Numerical time-series data. "How many errors per minute?" "What is the P99 latency?" Metrics tell you something is wrong. They are the foundation for alerting.

**Logs (CloudWatch Logs)**: Detailed text records of what happened. "This specific request failed with this error message." Logs tell you what went wrong. They are the foundation for debugging.

**Traces (X-Ray)**: End-to-end request flow across services. "This request went from API Gateway to Lambda to DynamoDB, and the DynamoDB call took 800ms." Traces tell you where in the system things went wrong. They are the foundation for performance analysis.

**Audit (CloudTrail)**: Who did what and when at the AWS API level. "User X deleted this S3 bucket at 3:47 PM." Audit tells you who caused the problem. It is the foundation for security and compliance.

### CloudWatch Application Signals

Application Signals (part of CloudWatch) provides application performance monitoring (APM) for services running on ECS, EKS, and EC2. It auto-discovers services, generates service maps, tracks SLIs (latency, availability, error rate), and lets you define SLOs. It uses OpenTelemetry for instrumentation.

This is AWS's answer to Datadog APM / New Relic — a managed APM that works with CloudWatch without third-party tools. It is still relatively new and less feature-rich than dedicated APM vendors.

### CloudWatch Contributor Insights

Contributor Insights analyzes log data in real time to identify top-N contributors to a metric. For example: "Which 10 IP addresses generate the most 5xx errors?" or "Which DynamoDB partition keys receive the most throttled requests?" It creates time-series data from CloudWatch Logs using rules that you define.

Use cases: Identify top talkers causing throttling, find the API endpoints generating the most errors, detect the user accounts consuming the most resources.

### CloudWatch Synthetics (Canaries)

Synthetics creates canary functions — scripts that run on a schedule and monitor your endpoints and APIs. Canaries simulate user behavior: navigate to a URL, check that specific elements are present, verify API responses, take screenshots.

Use cases: Monitor external-facing websites and APIs from the outside. Detect outages before users report them. Verify that login flows, checkout processes, and critical paths work end-to-end.

Canaries run on Lambda under the hood. You can write them in Node.js or Python. They support visual monitoring (screenshot comparison) and API monitoring (HTTP request validation).

### CloudWatch Embedded Metric Format (EMF)

EMF lets you generate custom CloudWatch metrics from structured log data without separate PutMetricData API calls. You write a specially formatted JSON log line to stdout, and CloudWatch automatically extracts metrics from it. This is much cheaper than PutMetricData (you pay for log ingestion, not per-metric) and allows high-cardinality dimensions that would be prohibitively expensive as custom metrics.

EMF is the recommended way to publish custom metrics from Lambda functions. Lambda Powertools for Python/TypeScript has built-in EMF support.

### Amazon Managed Grafana and Amazon Managed Prometheus

**Managed Grafana**: Fully managed Grafana for dashboards and visualization. Integrates with CloudWatch, Prometheus, X-Ray, and many other data sources. Use when your team prefers Grafana over CloudWatch dashboards, or when you need to visualize data from multiple sources (AWS + on-premises + third-party) in one place.

**Managed Prometheus**: Fully managed Prometheus for container metrics. Collects metrics from EKS and ECS workloads using the Prometheus data model. Use when your team already uses Prometheus (PromQL queries, Prometheus alerting rules) and wants AWS to manage the infrastructure.

These services are for teams that want open-source observability tools (Grafana/Prometheus) without managing the infrastructure themselves. They are alternatives to, not replacements for, CloudWatch.

### Observability Interview Questions

**Q: How do you debug a slow API request in a microservices architecture?**
A: Start with X-Ray to see the trace — it shows you which service in the chain is slow. Then check CloudWatch metrics for that service (latency, error rate, throttling). Finally, dive into CloudWatch Logs for the specific request ID to see detailed error messages. X-Ray annotations let you search for traces by business attributes (order ID, customer ID).

**Q: How do you set up alerting for a serverless application?**
A: Create CloudWatch Alarms on key metrics: Lambda Errors > 0, Lambda Throttles > 0, DynamoDB ThrottledRequests > 0, SQS DLQ ApproximateNumberOfMessagesVisible > 0, API Gateway 5XXError > threshold. Use composite alarms to reduce noise. Route alarms to SNS for team notification. Use CloudWatch Synthetics canaries to monitor the external-facing API from the user's perspective.

**Q: What is the difference between CloudWatch and third-party tools like Datadog?**
A: CloudWatch is the native AWS monitoring service — zero-agent for most AWS services, deep integration with IAM and AWS APIs, no data leaving AWS. Third-party tools (Datadog, New Relic, Splunk) offer richer visualization, better correlation across signals (metrics + logs + traces in one view), more powerful query languages, and support for multi-cloud and on-premises. Choose CloudWatch for cost-effective AWS-native monitoring. Choose third-party when you need advanced APM, multi-cloud observability, or team-preferred tooling.

---

## Developer Tools and CI/CD

### AWS CodeCommit (Deprecated)

CodeCommit was AWS's managed Git repository service. It is now deprecated — AWS recommends migrating to third-party Git providers (GitHub, GitLab, Bitbucket). Mentioned here because it still appears in some exam materials and existing architectures.

### AWS CodeBuild

CodeBuild is a fully managed build service. It compiles source code, runs tests, and produces artifacts. There are no build servers to manage — CodeBuild provisions a fresh container for each build.

Key features:
- **Build environments**: Pre-built Docker images for common languages (Java, Python, Node.js, Go, .NET, Ruby) or bring your own Docker image
- **buildspec.yml**: A YAML file in your source root that defines build phases (install, pre_build, build, post_build), environment variables, artifacts, and cache configuration
- **Caching**: Supports local caching (within the build container), S3 caching (shared across builds), and Docker layer caching (for container image builds)
- **Scaling**: Automatically starts new build containers for concurrent builds. No queue bottleneck.
- **VPC support**: Build containers can run inside your VPC to access private resources (databases, internal registries)

Pricing: Per build-minute. Compute types range from small (3 GB RAM, 2 vCPU, ~$0.005/min) to 2xlarge (145 GB RAM, 72 vCPU, ~$0.20/min). Free tier: 100 build-minutes/month on the small instance.

Gotchas: Build timeout default is 60 minutes (max 8 hours). Docker layer caching requires privileged mode. Environment variables marked as "parameter" are pulled from SSM Parameter Store; "secrets-manager" pulls from Secrets Manager.

### AWS CodeDeploy

CodeDeploy automates application deployments to EC2 instances, ECS services, Lambda functions, and on-premises servers. It supports multiple deployment strategies:

**EC2/On-premises**:
- **In-place**: Stop application on each instance, deploy new version, restart. Instances are temporarily out of service.
- **Blue/Green**: Create new instances with the new version, shift traffic from old to new, terminate old instances.

**ECS**:
- **Blue/Green**: Create new task set with the new version, shift traffic (all-at-once, linear, or canary), terminate old task set.
- **Linear**: Shift traffic in equal increments (e.g., 10% every 10 minutes).
- **Canary**: Shift a small percentage first (e.g., 10%), wait, then shift the rest.

**Lambda**:
- **Canary**: Route a percentage of traffic to the new version (e.g., 10% for 5 minutes), then shift all traffic.
- **Linear**: Shift traffic in equal increments over time.
- **All-at-once**: Immediate switch (fastest but riskiest).

**Rollback**: CodeDeploy can automatically roll back if CloudWatch alarms fire during deployment. This is the safety net — define alarms on error rate, latency, or custom metrics, and CodeDeploy reverts to the previous version if thresholds are breached.

**appspec.yml**: A YAML file that defines what to deploy (files, permissions) and lifecycle hooks (scripts to run before/after install, before/after traffic shift).

### AWS CodePipeline

CodePipeline is a continuous delivery service that orchestrates build, test, and deploy stages. It connects source providers (GitHub, CodeCommit, S3, ECR), build providers (CodeBuild, Jenkins), and deploy providers (CodeDeploy, ECS, S3, CloudFormation, CDK).

Key concepts:
- **Pipeline**: A series of stages executed in order
- **Stage**: A group of one or more actions (e.g., Source, Build, Test, Deploy)
- **Action**: A specific task (e.g., pull source from GitHub, run CodeBuild, deploy with CodeDeploy)
- **Artifacts**: Outputs from one stage passed as inputs to the next (stored in S3)
- **Manual approval**: A stage can require manual approval before proceeding (useful for production deployments)

Pricing: $1/month per active pipeline. First pipeline is free. Action executions are included.

### CodePipeline vs GitHub Actions vs CDK Pipelines

**CodePipeline**: AWS-native orchestration. Best when all your build and deploy tooling is AWS-native (CodeBuild, CodeDeploy, CloudFormation). Tight IAM integration. Visual pipeline editor. Less flexible than GitHub Actions for complex workflows.

**GitHub Actions**: More flexible workflow engine. Richer marketplace of community actions. Better for teams that use GitHub as their primary development platform. Can deploy to AWS using OIDC federation (no long-lived access keys). More natural for teams already comfortable with YAML-based CI/CD.

**CDK Pipelines**: A CDK construct that creates a self-mutating CodePipeline for CDK applications. It automatically adds stages for synthesis, self-mutation (updating the pipeline itself when you change the pipeline code), and deployment to multiple accounts/regions. Best for CDK-heavy teams that want the pipeline to be defined in the same code as the infrastructure.

Decision: Use CDK Pipelines for CDK applications. Use GitHub Actions if your team already uses GitHub and wants maximum flexibility. Use CodePipeline for pure AWS-native CI/CD or when you cannot use third-party tools.

### AWS CodeArtifact

CodeArtifact is a managed artifact repository for software packages. It supports npm, PyPI, Maven, NuGet, and generic formats. It acts as a proxy for public registries (npmjs.com, pypi.org) and stores private packages.

Use cases: Cache public packages for faster builds and protection against supply chain attacks (upstream registry outages or compromised packages). Host private packages shared across teams. Enforce package approval policies.

Pricing: $0.05/GB/month for storage + $0.09/GB for data transfer.

### AWS CodeGuru

CodeGuru uses ML to provide automated code reviews and application performance recommendations.

**CodeGuru Reviewer**: Analyzes pull requests for code quality issues, security vulnerabilities, and AWS SDK best practices. Supports Java and Python. Integrates with GitHub, Bitbucket, and CodeCommit.

**CodeGuru Profiler**: Identifies the most expensive lines of code at runtime (CPU and memory). Runs in production with minimal overhead (~5%). Helps optimize Lambda function costs and application performance.

### Amazon ECR (Elastic Container Registry)

ECR is a managed Docker container registry. It stores, manages, and deploys container images.

Key features:
- **Private and public repositories**: Private for internal images, public (ECR Public Gallery) for open-source images
- **Image scanning**: On-push or continuous scanning for OS and programming language vulnerabilities using Amazon Inspector
- **Lifecycle policies**: Automatically clean up old/untagged images (critical for cost control)
- **Cross-region replication**: Replicate images to other regions for multi-region deployments
- **Immutable tags**: Prevent image tags from being overwritten (important for production — ensures `v1.2.3` always refers to the same image)

Pricing: $0.10/GB/month for storage. Data transfer to ECS/EKS within the same region is free. Cross-region pull is charged.

Gotchas: ECR images have a default scan-on-push that only detects OS vulnerabilities. Enable Amazon Inspector for deeper scanning (language library vulnerabilities). Without lifecycle policies, old images accumulate and storage costs grow silently. ECR login tokens expire after 12 hours — automate token refresh in CI/CD.

### CI/CD Patterns on AWS

**Pattern 1: GitHub Actions + CDK**
Source on GitHub -> GitHub Actions runs on push -> builds with npm/docker, runs tests -> deploys with `cdk deploy` using OIDC-federated AWS credentials. Simple, flexible, no AWS CI/CD services needed.

**Pattern 2: CDK Pipelines (self-mutating)**
Source on GitHub/CodeCommit -> CDK Pipeline (CodePipeline under the hood) -> synthesizes CDK app -> self-mutates pipeline if pipeline code changed -> deploys to dev -> manual approval -> deploys to prod. Best for multi-stage, multi-account CDK deployments.

**Pattern 3: Full AWS-native**
CodeCommit/GitHub -> CodePipeline -> CodeBuild (build + test) -> CodeDeploy (blue/green to ECS) with CloudWatch alarm rollback. Everything managed by AWS, tight IAM integration, visual pipeline.

**Pattern 4: Container image pipeline**
Source push -> CodeBuild builds Docker image -> pushes to ECR -> CodePipeline detects new image -> CodeDeploy blue/green deploys to ECS/EKS with canary traffic shifting.

### CI/CD Interview Questions

**Q: How do you implement zero-downtime deployments on AWS?**
A: For ECS: Use CodeDeploy blue/green deployment — create a new task set, shift traffic gradually (canary or linear), roll back automatically if CloudWatch alarms fire. For Lambda: Use CodeDeploy with canary or linear traffic shifting between alias versions. For EC2: Use CodeDeploy blue/green with a new Auto Scaling group. In all cases, define health check alarms that trigger automatic rollback.

**Q: How do you secure a CI/CD pipeline on AWS?**
A: Use OIDC federation for GitHub Actions (no long-lived access keys). Use IAM roles with least-privilege for each pipeline stage. Store secrets in Secrets Manager, not in pipeline environment variables. Enable CodeBuild VPC mode for builds that need private resource access. Use ECR image scanning and immutable tags. Add manual approval gates before production deployments. Encrypt artifacts in S3 with KMS. Enable CloudTrail for audit logging of all pipeline actions.

**Q: What is the difference between CodeDeploy deployment strategies?**
A: All-at-once is fastest but riskiest (no gradual rollout). Canary shifts a small percentage of traffic first (e.g., 10% for 5 minutes) to catch issues with minimal blast radius, then shifts the rest. Linear shifts traffic in equal increments over time (e.g., 10% every 2 minutes) for smoother transitions. Blue/green creates an entirely new environment and switches traffic. All support automatic rollback on alarm. Choose canary for catching issues early, linear for smooth transitions, blue/green for complete environment isolation.

**Q: When would you use CodePipeline vs GitHub Actions?**
A: Use CodePipeline when you want a pure AWS-native solution, need tight integration with CodeDeploy blue/green deployments, or have compliance requirements that keep CI/CD within AWS. Use GitHub Actions when your team is already on GitHub, needs complex workflow logic (matrix builds, reusable workflows), or wants access to the broader GitHub Actions marketplace. CDK Pipelines is the best choice for CDK applications regardless of source provider.
