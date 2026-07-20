# AWS SAA-C03 Tricky Scenario Practice Questions

> **Important**
>
> These are original practice questions for architecture reasoning. They are not exam dumps, do not reproduce real exam questions, and should be used to practice service selection and tradeoff analysis for the AWS Certified Solutions Architect - Associate (SAA-C03).

## How To Use This File

- Read the scenario first.
- Identify the dominant requirement: security, resilience, performance, or cost.
- Eliminate answers that violate a hard constraint.
- Click **Show answer and explanation** only after choosing.

## Domain Coverage

| Domain | Official focus | Questions |
|---|---|---:|
| Domain 1 | Design Secure Architectures | 8 |
| Domain 2 | Design Resilient Architectures | 8 |
| Domain 3 | Design High-Performing Architectures | 8 |
| Domain 4 | Design Cost-Optimized Architectures | 8 |

Primary syllabus references:

- [Domain 1: Design Secure Architectures](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03-domain1.html)
- [Domain 2: Design Resilient Architectures](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03-domain2.html)
- [Domain 3: Design High-Performing Architectures](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03-domain3.html)
- [Domain 4: Design Cost-Optimized Architectures](https://docs.aws.amazon.com/aws-certification/latest/solutions-architect-associate-03/solutions-architect-associate-03-domain4.html)

---

# Domain 1: Design Secure Architectures

## Question 1: Cross-Account S3 Access Without Users

A company has a centralized logging account with an S3 bucket. Application accounts must write logs to the bucket. The security team does not want long-term access keys in application accounts and wants access limited to only the logging bucket prefix for each account.

Which solution best meets these requirements?

A. Create IAM users in the logging account and store their access keys in each application account.  
B. Create an IAM role in the logging account and allow application accounts to assume it; restrict the role and bucket policy to account-specific prefixes.  
C. Make the S3 bucket public and use random prefix names for each application account.  
D. Use a single AWS account root user access key for all logging writes.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

Use cross-account IAM role assumption and S3 bucket/resource policies. This avoids long-term credentials, supports least privilege, and can restrict writes to specific prefixes.

Why the others are wrong:

- **A** uses long-term credentials and increases operational risk.
- **C** violates basic data security controls.
- **D** is never appropriate; root access keys should not be used for workloads.

</details>

## Question 2: Private Subnet Access To S3

EC2 instances in private subnets need to download objects from Amazon S3. The company wants to avoid public internet paths and reduce NAT Gateway processing charges.

What should the solutions architect recommend?

A. Add an internet gateway to the private subnets.  
B. Use a NAT Gateway in each Availability Zone.  
C. Create a gateway VPC endpoint for Amazon S3 and update route tables.  
D. Create a Client VPN endpoint for the EC2 instances.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: C**

An S3 gateway VPC endpoint keeps S3 traffic on the AWS network and avoids NAT Gateway data processing for S3 access.

Why the others are wrong:

- **A** would make the subnet public if routes point to the internet gateway.
- **B** works for outbound internet, but it is not the lowest-cost private S3 path.
- **D** is for remote users connecting to a VPC, not EC2-to-S3 service access.

</details>

## Question 3: Secrets Rotation For RDS

An application running on Amazon ECS needs database credentials for Amazon RDS. The security team requires automatic credential rotation and auditability. Developers should not store database passwords in container images or environment files.

Which service should be used?

A. AWS Systems Manager Parameter Store standard parameter with no rotation.  
B. AWS Secrets Manager with automatic rotation using a Lambda rotation function.  
C. Amazon S3 encrypted object containing the password.  
D. Amazon CloudWatch Logs encrypted log group.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

Secrets Manager is designed for secrets storage, retrieval, auditability, and rotation. It integrates well with RDS and ECS task definitions.

Why the others are wrong:

- **A** can store configuration, but standard parameters do not provide managed secret rotation.
- **C** is not a secret-management service.
- **D** is for logs, not credential storage.

</details>

## Question 4: Web Attack Protection

A public web application behind an Application Load Balancer has recently received SQL injection and cross-site scripting attempts. The team wants managed protection with minimal application changes.

What is the best solution?

A. Attach AWS WAF to the Application Load Balancer with managed rule groups.  
B. Use AWS Shield Standard only.  
C. Replace the ALB with a NAT Gateway.  
D. Enable Amazon GuardDuty and make no traffic filtering changes.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

AWS WAF protects web applications from common layer 7 attacks such as SQL injection and XSS. It can attach to ALB, CloudFront, API Gateway, and AppSync.

Why the others are wrong:

- **B** helps with DDoS, not application-layer rule filtering.
- **C** NAT Gateway is unrelated to inbound web protection.
- **D** GuardDuty detects suspicious activity but does not block these HTTP requests.

</details>

## Question 5: S3 Data Classification

A company stores millions of documents in Amazon S3. Compliance requires identifying buckets that contain personally identifiable information and generating findings for the security team.

Which service is most appropriate?

A. Amazon Inspector  
B. Amazon Macie  
C. AWS Artifact  
D. AWS X-Ray

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

Amazon Macie discovers and reports sensitive data such as PII in S3.

Why the others are wrong:

- **A** scans workloads/images/functions for vulnerabilities.
- **C** provides AWS compliance reports and agreements.
- **D** traces distributed application requests.

</details>

## Question 6: Workforce Access Across Many Accounts

A company uses AWS Organizations with separate accounts for dev, test, and production. Employees authenticate with an external identity provider. The company wants centralized workforce access and permission sets across accounts.

Which service should be used?

A. Amazon Cognito user pools  
B. AWS IAM Identity Center  
C. AWS Secrets Manager  
D. AWS Resource Access Manager

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

IAM Identity Center is the preferred service for centralized workforce access to multiple AWS accounts and cloud applications.

Why the others are wrong:

- **A** is for application customer identity, not workforce AWS account access.
- **C** stores and rotates secrets.
- **D** shares supported resources across accounts, but does not provide workforce SSO.

</details>

## Question 7: Enforcing Maximum Permissions

A security team wants to ensure that no account inside an AWS Organizations OU can disable CloudTrail, even if an administrator in that account has an IAM policy allowing it.

What should be used?

A. An IAM group policy in each member account  
B. A service control policy attached to the OU  
C. A security group rule  
D. A resource tag on CloudTrail

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

Service control policies define maximum available permissions for accounts in an organization or OU. They can prevent actions even when local IAM policies allow them.

Key trap:

SCPs do not grant permissions. They only restrict what permissions can be used.

</details>

## Question 8: KMS Key Access Trap

An application role has an IAM policy allowing `kms:Decrypt` on a customer managed KMS key. The application still receives access denied errors when decrypting data.

What is the most likely cause?

A. KMS keys cannot be used by IAM roles.  
B. The KMS key policy does not allow the role or account to use the key.  
C. KMS only works with S3 objects.  
D. The role needs AmazonS3FullAccess.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

KMS authorization depends on key policy and IAM permissions. If the key policy does not allow access directly or delegate access to the account, IAM permission alone is not enough.

Why the others are wrong:

- **A** IAM roles can use KMS keys.
- **C** KMS integrates with many AWS services.
- **D** S3 permissions do not solve KMS key policy denial.

</details>

---

# Domain 2: Design Resilient Architectures

## Question 9: Stateless Web Tier Resilience

A company runs a web application on two EC2 instances in one Availability Zone. Users lose access whenever that AZ has problems. The application stores session state on the instances.

Which design improves resilience the most?

A. Move both EC2 instances to a larger instance type in the same AZ.  
B. Use an Auto Scaling group across multiple AZs behind an ALB and move session state to ElastiCache or DynamoDB.  
C. Add a second internet gateway.  
D. Store session state in EBS volumes attached to each instance.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

A resilient web tier should be stateless and spread across multiple AZs. Externalizing session state allows instances to be replaced or scaled without losing sessions.

Why the others are wrong:

- **A** improves capacity, not AZ resilience.
- **C** a VPC has one internet gateway attachment and this does not solve compute failure.
- **D** EBS is AZ-scoped and attached to instances, so it does not make the web tier stateless.

</details>

## Question 10: Queue-Based Decoupling

An image-processing application receives unpredictable upload spikes. The upload API must respond quickly, and image processing can happen asynchronously. Failed jobs should be retried and eventually isolated for review.

Which design is best?

A. API Gateway invokes Lambda synchronously and waits for all image processing to finish.  
B. The upload service sends messages to SQS; workers process messages; configure a dead-letter queue.  
C. Store jobs in CloudWatch Logs and scan logs every hour.  
D. Use a single EC2 instance with a cron job.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

SQS decouples producers from workers, buffers spikes, supports retries, and integrates with DLQs for failed messages.

Why the others are wrong:

- **A** couples user latency to backend processing.
- **C** logs are not a work queue.
- **D** creates a single point of failure and poor scaling.

</details>

## Question 11: DR Strategy With Low Cost

A workload must be recoverable in another Region. The business accepts several hours of downtime and data loss up to the last nightly backup. Cost must be minimized.

Which disaster recovery strategy is most appropriate?

A. Active-active multi-Region  
B. Warm standby  
C. Backup and restore  
D. Multi-AZ only in the primary Region

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: C**

Backup and restore is the lowest-cost DR strategy when RTO and RPO can be measured in hours and data can be restored from backups.

Why the others are wrong:

- **A** and **B** reduce recovery time but cost more.
- **D** protects against AZ failure, not regional failure.

</details>

## Question 12: RDS HA vs Read Scaling

A production relational database must automatically fail over if the primary Availability Zone fails. The application does not need additional read capacity.

What should be configured?

A. RDS Multi-AZ deployment  
B. RDS read replica in the same Region only  
C. DynamoDB global table  
D. ElastiCache read replica

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

RDS Multi-AZ is the standard high-availability configuration for automatic failover. Read replicas are primarily for read scaling and may require promotion.

</details>

## Question 13: Ordered Processing With Retry

A payment processing system must process events in order for each customer account. The system must support retries and prevent duplicate processing as much as possible.

Which service is the best fit?

A. Amazon SQS FIFO queue with message group ID per customer account  
B. Amazon SQS Standard queue  
C. Amazon SNS Standard topic only  
D. Amazon EventBridge default bus only

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

SQS FIFO supports ordering within a message group and deduplication. A customer account ID as the message group ID preserves per-customer ordering while allowing parallelism across customers.

Why the others are wrong:

- **B** provides best-effort ordering only.
- **C** is pub/sub, not a durable ordered worker queue by itself.
- **D** is event routing, not strict ordered processing.

</details>

## Question 14: Multi-Region Read App

A read-heavy application serves users in North America and Europe. Users need low-latency reads in both Regions. Writes can be routed to one primary Region, but read data should be available globally with minimal operational overhead.

Which database feature is most appropriate for a relational Aurora workload?

A. Aurora Global Database  
B. EBS snapshots copied hourly  
C. S3 Cross-Region Replication only  
D. One larger Aurora writer instance

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Aurora Global Database supports low-latency global reads with replication to secondary Regions. It is built for cross-Region read scaling and disaster recovery.

Why the others are wrong:

- **B** is backup/restore, not low-latency global reads.
- **C** replicates objects, not relational database transactions.
- **D** does not reduce latency for European users.

</details>

## Question 15: Legacy App Cannot Change

A legacy application writes uploaded files to a local file system path on multiple Linux web servers. The company wants the app to survive instance replacement with minimal code changes.

Which storage option best supports this?

A. Instance store on each EC2 instance  
B. Amazon EFS mounted by all web servers  
C. Separate EBS volume for each web server  
D. Amazon S3 mounted through custom scripts as a POSIX file system

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

EFS provides a managed shared NFS file system for Linux workloads across multiple clients and AZs. It is a strong fit when legacy apps need shared file paths.

Why the others are wrong:

- **A** is ephemeral.
- **C** does not provide shared multi-instance file semantics.
- **D** S3 is object storage, not a native POSIX file system.

</details>

## Question 16: Health-Based Regional Failover

A company hosts the same public application in two Regions. It wants users routed to the primary Region unless health checks fail, then routed to the secondary Region.

Which Route 53 routing policy should be used?

A. Weighted routing  
B. Failover routing  
C. Geolocation routing  
D. Simple routing

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

Failover routing uses health checks to route traffic from a primary endpoint to a secondary endpoint when the primary is unhealthy.

Why the others are wrong:

- **A** splits traffic by weight.
- **C** routes by user geography.
- **D** has no health-based failover logic.

</details>

---

# Domain 3: Design High-Performing Architectures

## Question 17: S3 Analytics Performance

A team queries 20 TB of CSV logs in S3 using Athena. Queries are slow and expensive. The access pattern filters heavily by date and application ID.

Which changes should improve performance and reduce cost? Choose TWO.

A. Convert data to Apache Parquet.  
B. Partition the data by date and application ID.  
C. Move all data to S3 Glacier Deep Archive.  
D. Increase the size of the Athena server.  
E. Store each log line as a separate S3 object.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answers: A and B**

Athena charges by data scanned. Columnar formats such as Parquet and partitioning reduce scanned data and improve query performance.

Why the others are wrong:

- **C** makes data slow to retrieve and unsuitable for frequent Athena queries.
- **D** Athena is serverless; you do not resize servers.
- **E** too many tiny objects can hurt performance and operations.

</details>

## Question 18: Compute For Long Batch Jobs

A company runs containerized financial simulations. Jobs run for 2 to 6 hours, can be retried, and need to scale based on queue depth. The company wants managed scheduling.

Which service is most appropriate?

A. AWS Lambda  
B. AWS Batch  
C. Amazon API Gateway  
D. AWS Device Farm

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

AWS Batch is designed for managed batch scheduling and can run long containerized jobs using EC2, Spot, or Fargate compute environments.

Why the others are wrong:

- **A** has a 15-minute maximum timeout.
- **C** is an API front door.
- **D** is for application testing on devices/browsers.

</details>

## Question 19: Read-Heavy Database

An application uses Amazon RDS for PostgreSQL. CPU on the writer is low, but read query latency is high during peak reporting hours. The workload can tolerate slightly stale reads for reports.

What should be recommended?

A. Add one or more read replicas and route reporting queries to them.  
B. Convert the database to S3 Standard-IA.  
C. Enable Multi-AZ only and expect read performance to improve.  
D. Replace all SQL queries with CloudTrail queries.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Read replicas offload read traffic from the primary database. They are appropriate when read traffic dominates and slightly stale reads are acceptable.

Key trap:

Multi-AZ is for high availability. It is not primarily a read-scaling feature for standard RDS deployments.

</details>

## Question 20: Cache Placement

A product catalog service reads the same product metadata thousands of times per minute from a relational database. The data changes a few times per day. The goal is sub-millisecond reads and reduced database load.

Which approach best fits?

A. Add ElastiCache using a cache-aside pattern.  
B. Add more NAT Gateways.  
C. Use AWS Backup to restore the database every hour.  
D. Move the database to EBS Cold HDD.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

ElastiCache is appropriate for sub-millisecond reads and reducing database load when cached data can be refreshed or invalidated safely.

</details>

## Question 21: Global Static Content

A media site serves images and CSS from an S3 bucket to users around the world. Users far from the Region report slow page loads.

What should be added?

A. Amazon CloudFront with the S3 bucket as origin  
B. AWS Direct Connect  
C. Amazon EBS Provisioned IOPS  
D. AWS Snowball Edge

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

CloudFront caches content at edge locations to reduce latency for global users and can securely access S3 origins.

Why the others are wrong:

- **B** is for private hybrid connectivity.
- **C** is block storage for EC2.
- **D** is for offline transfer or edge compute in disconnected locations.

</details>

## Question 22: Streaming Click Events

A web application emits clickstream events. Several analytics applications need to consume the same ordered stream independently and replay data for debugging within the retention period.

Which service should be selected?

A. Amazon Kinesis Data Streams  
B. Amazon SQS Standard queue  
C. AWS DataSync  
D. Amazon EFS

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Kinesis Data Streams supports ordered records per partition key, multiple consumers, and replay within the configured retention period.

Why the others are wrong:

- **B** is a queue, not a replayable multi-consumer stream.
- **C** transfers files.
- **D** is shared file storage.

</details>

## Question 23: Choosing ALB vs NLB

A TCP-based trading application requires very high throughput, static IP addresses per Availability Zone, and low latency. HTTP path-based routing is not needed.

Which load balancer should be used?

A. Application Load Balancer  
B. Network Load Balancer  
C. Gateway Load Balancer  
D. Classic Load Balancer

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

NLB is designed for high-performance Layer 4 traffic and provides static IP addresses per AZ.

Why the others are wrong:

- **A** is for Layer 7 HTTP/HTTPS routing.
- **C** is for deploying and scaling virtual network appliances.
- **D** is previous-generation and rarely the best answer for new designs.

</details>

## Question 24: DynamoDB Hot Partition

A DynamoDB table stores IoT events. The partition key is the current date, so all devices write to the same partition for a day. Write throttling occurs during peak ingestion.

What should be changed?

A. Use a higher-cardinality partition key such as device ID or a composite write-sharded key.  
B. Replace DynamoDB with Amazon EFS.  
C. Enable S3 Transfer Acceleration.  
D. Add an Application Load Balancer in front of DynamoDB.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

DynamoDB performance depends heavily on partition key design. A low-cardinality key such as date can create hot partitions. Higher-cardinality or write-sharded keys distribute load.

</details>

---

# Domain 4: Design Cost-Optimized Architectures

## Question 25: Unknown S3 Access Pattern

A company stores user-generated objects in S3. Some objects are accessed frequently for a few days, some are never accessed again, and access patterns are hard to predict. The company wants automatic cost optimization without operational analysis.

Which storage class is best?

A. S3 Standard only  
B. S3 Intelligent-Tiering  
C. S3 Glacier Deep Archive for all objects immediately  
D. EBS Throughput Optimized HDD

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: B**

S3 Intelligent-Tiering automatically moves objects between access tiers when access patterns are unknown or changing.

Why the others are wrong:

- **A** can be more expensive for rarely accessed objects.
- **C** is too slow and restrictive for objects that may be accessed soon.
- **D** is EC2 block storage, not object storage.

</details>

## Question 26: Compute Purchasing Option

A workload runs continuously at a predictable baseline of 40 EC2 instances for the next three years. The company wants to reduce cost while retaining flexibility across instance families and Regions as much as possible.

Which option is usually the best fit?

A. Compute Savings Plans  
B. Spot Instances only  
C. On-Demand Instances only  
D. Dedicated Hosts only

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Compute Savings Plans provide discounts for committed compute spend and are more flexible than many instance-specific commitments.

Why the others are wrong:

- **B** is interruptible and unsuitable as the only option for required steady baseline capacity.
- **C** has maximum flexibility but no commitment discount.
- **D** is mainly for licensing/compliance requirements, not general cost optimization.

</details>

## Question 27: Fault-Tolerant Workers At Lowest Cost

A batch image-rendering fleet can tolerate interruptions and retries. Jobs are queued in SQS and checkpoints are written to S3. The goal is lowest compute cost.

What should be used?

A. Spot Instances or Spot capacity in the worker fleet  
B. Dedicated Hosts  
C. RDS Multi-AZ  
D. AWS Shield Advanced

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Spot capacity is well suited for interruption-tolerant, retryable workloads. SQS and S3 checkpoints make the design resilient to interruptions.

</details>

## Question 28: NAT Gateway Cost

Private subnet workloads download large amounts of data from S3 every day through a NAT Gateway. NAT data processing charges are increasing.

What is the most cost-effective improvement?

A. Add an S3 gateway VPC endpoint and update route tables.  
B. Add another NAT Gateway in the same AZ.  
C. Move the S3 bucket to EBS.  
D. Use AWS WAF to cache S3 data.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Gateway VPC endpoints for S3 avoid NAT Gateway data processing for supported S3 traffic from private subnets.

</details>

## Question 29: Cost Allocation

A company has multiple teams sharing one AWS account. Finance needs monthly cost reports by team and application.

What should be implemented first?

A. Cost allocation tags applied consistently to resources.  
B. Larger EC2 instances for all workloads.  
C. S3 Transfer Acceleration for all buckets.  
D. AWS Shield Advanced for every account.

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Cost allocation tags enable costs to be grouped and reported by team, application, environment, or owner.

Exam trap:

You cannot optimize what you cannot attribute. Tagging and account structure are foundational for cost visibility.

</details>

## Question 30: Database Cost For Intermittent Workload

A small internal application uses a relational database only during business hours. Traffic is unpredictable during those hours and near zero overnight. The team wants to reduce idle database cost while keeping SQL compatibility.

Which option is most appropriate?

A. Aurora Serverless v2 with appropriate minimum and maximum capacity  
B. DynamoDB with no schema redesign  
C. RDS Multi-AZ with the largest instance class  
D. Amazon Redshift provisioned cluster

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Aurora Serverless v2 keeps relational compatibility while scaling capacity based on demand, which can reduce idle cost for variable workloads.

Why the others are wrong:

- **B** may be good for non-relational patterns but requires schema/access-pattern redesign.
- **C** increases cost and focuses on HA, not idle cost.
- **D** is for analytics warehousing, not OLTP application databases.

</details>

## Question 31: Archive Retrieval Tradeoff

A company must retain compliance records for seven years. Records are rarely accessed, and retrieval within 12 hours is acceptable. Lowest storage cost is the top priority.

Which storage class is best?

A. S3 Standard  
B. S3 Standard-IA  
C. S3 Glacier Deep Archive  
D. EFS Standard

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: C**

S3 Glacier Deep Archive is designed for long-term archive data where retrieval time of hours is acceptable and lowest storage cost is important.

</details>

## Question 32: Data Transfer Into AWS

A company has 500 TB of archival data in an on-premises data center. The internet connection is slow and would take months to upload the data. The company wants a faster, cost-effective one-time transfer to Amazon S3.

Which service should be used?

A. AWS Snowball Edge  
B. AWS Client VPN  
C. Amazon CloudFront  
D. Amazon Route 53

<details>
<summary><strong>Show answer and explanation</strong></summary>

**Answer: A**

Snowball Edge is appropriate for large offline data transfers when network transfer would be too slow or expensive.

Why the others are wrong:

- **B** provides remote user VPN access.
- **C** is a CDN.
- **D** is DNS.

</details>

---

# Quick Answer Key

<details>
<summary><strong>Show quick answer key</strong></summary>

| Question | Domain | Answer |
|---:|---|---|
| 1 | Secure Architectures | B |
| 2 | Secure Architectures | C |
| 3 | Secure Architectures | B |
| 4 | Secure Architectures | A |
| 5 | Secure Architectures | B |
| 6 | Secure Architectures | B |
| 7 | Secure Architectures | B |
| 8 | Secure Architectures | B |
| 9 | Resilient Architectures | B |
| 10 | Resilient Architectures | B |
| 11 | Resilient Architectures | C |
| 12 | Resilient Architectures | A |
| 13 | Resilient Architectures | A |
| 14 | Resilient Architectures | A |
| 15 | Resilient Architectures | B |
| 16 | Resilient Architectures | B |
| 17 | High-Performing Architectures | A, B |
| 18 | High-Performing Architectures | B |
| 19 | High-Performing Architectures | A |
| 20 | High-Performing Architectures | A |
| 21 | High-Performing Architectures | A |
| 22 | High-Performing Architectures | A |
| 23 | High-Performing Architectures | B |
| 24 | High-Performing Architectures | A |
| 25 | Cost-Optimized Architectures | B |
| 26 | Cost-Optimized Architectures | A |
| 27 | Cost-Optimized Architectures | A |
| 28 | Cost-Optimized Architectures | A |
| 29 | Cost-Optimized Architectures | A |
| 30 | Cost-Optimized Architectures | A |
| 31 | Cost-Optimized Architectures | C |
| 32 | Cost-Optimized Architectures | A |

</details>

> **Study Tip**
>
> When you miss a question, write down the constraint you missed. SAA-C03 questions often become easy once you isolate the hard requirement: ordering, RTO/RPO, latency, managed operations, protocol compatibility, or lowest cost.
