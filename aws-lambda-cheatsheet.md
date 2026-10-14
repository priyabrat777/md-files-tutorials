# AWS Lambda Cheat Sheet

**Interview preparation: compute modes, invocation types, retries, concurrency, networking, and durable execution.**

**Verified:** 2026-10-13. The timeout answer depends on compute type and invocation path; the 90-minute change is scoped, not universal.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Current correction: 15 minutes versus 90 minutes](#2-current-correction-15-minutes-versus-90-minutes)
- [3. Default compute versus Managed Instances](#3-default-compute-versus-managed-instances)
- [4. Invocation modes and who retries](#4-invocation-modes-and-who-retries)
- [5. Concurrency and backpressure](#5-concurrency-and-backpressure)
- [6. Cold starts, initialization, and SnapStart](#6-cold-starts-initialization-and-snapstart)
- [7. SQS processing and idempotency](#7-sqs-processing-and-idempotency)
- [8. Networking and IAM](#8-networking-and-iam)
- [9. Limits, packaging, and cost](#9-limits-packaging-and-cost)
- [10. Durable functions and alternatives](#10-durable-functions-and-alternatives)
- [11. Troubleshooting and practical inspection](#11-troubleshooting-and-practical-inspection)
- [12. Interview questions and quick revision](#12-interview-questions-and-quick-revision)
- [13. AWS sources](#13-aws-sources)

---

## 1. Fundamentals and mental model

Lambda runs application handlers with AWS-managed infrastructure. You configure code/runtime, memory/resources, execution role, invocation sources, scaling controls, and failure handling.

### Full Lambda Invocation Flow

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                          EVENT SOURCE                                    │
│         API Gateway · S3 · DynamoDB Stream · SQS · SNS · EventBridge    │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                     INVOCATION MECHANISM                                 │
│                                                                          │
│  Synchronous (request/response)  ──► Caller waits for result            │
│  Asynchronous (fire & forget)    ──► Lambda queues, returns 202         │
│  Event Source Mapping (polling)  ──► Lambda pulls from stream/queue     │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
                                 ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                   EXECUTION ENVIRONMENT                                  │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │  Runtime (Python, Node.js, Java, Go, .NET, Ruby, Custom)       │    │
│  │  /tmp storage (512 MB – 10 GB ephemeral)                       │    │
│  │  Environment variables (including secrets from Secrets Manager) │    │
│  │  SDK clients initialized in global scope (reused across calls)  │    │
│  └────────────────────────────────────────────────────────────────┘    │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────┐    │
│  │                      HANDLER FUNCTION                           │    │
│  │  event → business logic → response                             │    │
│  │                                                                 │    │
│  │  Uses EXECUTION ROLE to call AWS APIs:                         │    │
│  │    ├─ DynamoDB GetItem/PutItem                                 │    │
│  │    ├─ S3 GetObject/PutObject                                   │    │
│  │    └─ SQS SendMessage                                          │    │
│  └────────────────────────────────────────────────────────────────┘    │
└────────────────────────────────┬─────────────────────────────────────────┘
                                 │
            ┌────────────────────┼────────────────────┐
            │                    │                    │
            ▼                    ▼                    ▼
       Success              Handler Error        System Error
         │                      │                    │
         └──────────────────────┴────────────────────┘
                                 │
                        Retry owner decision:
                        ├─ Sync: Caller retries
                        ├─ Async: Lambda retries (2×)
                        └─ Stream: Mapping retries per config
```

### Invocation Type Mental Models

```text
SYNCHRONOUS (Request/Response):
Client ──request──► Lambda ──waits──► Handler executes ──result──► Client
                    (timeout = 15 min max)
Who retries: CALLER (client must implement retry logic)
Use cases: API Gateway, direct invoke, ALB

ASYNCHRONOUS (Fire & Forget):
Client ──request──► Lambda ──202 Accepted──► Client (continues immediately)
                         │
                         └──queues──► Handler executes later
                                      (retries 2× on failure by default)
Who retries: LAMBDA (via async invocation queue)
Use cases: S3 events, SNS, EventBridge, CloudWatch Logs

EVENT SOURCE MAPPING (Pull-based):
Lambda ──polls──► SQS/Kinesis/DynamoDB ──batch──► Handler(batch)
                                                   (retries per mapping config)
Who retries: MAPPING + SOURCE (depends on visibility/iterator age)
Use cases: SQS, Kinesis, DynamoDB Streams, Kafka
```

### Execution Role vs Resource Policy

```text
EXECUTION ROLE (IAM Role assumed by Lambda):
Lambda assumes this role when handler runs
→ Grants permissions for handler to call AWS APIs

Example: Handler needs to write to DynamoDB
Execution Role Policy:
{
  "Effect": "Allow",
  "Action": ["dynamodb:PutItem", "dynamodb:GetItem"],
  "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
}

RESOURCE POLICY (Who can invoke this Lambda):
Controls which principals can invoke the function

Example: Allow S3 bucket to invoke function
Resource Policy:
{
  "Effect": "Allow",
  "Principal": {"Service": "s3.amazonaws.com"},
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessImage",
  "Condition": {
    "StringEquals": {"AWS:SourceAccount": "123456789012"},
    "ArnLike": {"AWS:SourceArn": "arn:aws:s3:::my-bucket"}
  }
}

Key Distinction:
├─ Execution role = What Lambda can DO (call AWS APIs)
└─ Resource policy = Who can INVOKE the Lambda
```

Managed infrastructure does not eliminate deadlines, connection limits, idempotency, or application security.

## 2. Current correction: 15 minutes versus 90 minutes

AWS announced the increase on **September 9, 2026**. [90-minute announcement](https://aws.amazon.com/about-aws/whats-new/2026/09/aws-lambda-90-minute-function/)

| Execution path | Maximum duration | Qualification |
|---|---|---|
| Lambda default compute, ordinary invocation | 900 seconds / 15 minutes | Standard per-invocation ceiling |
| Lambda Managed Instances, synchronous invocation | 15 minutes | Not covered by the async/ESM increase |
| Lambda Managed Instances, asynchronous invocation | 5,400 seconds / 90 minutes | Configure this compute/invocation path |
| Lambda Managed Instances, eligible event source mapping | 90 minutes | Amazon MQ and Amazon DocumentDB mappings are excluded |
| Durable function, asynchronous durable execution | Up to one year | Workflow lifetime across checkpoints/waits/invocations; not one year of one continuous invocation |

[Function quotas and exceptions](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html), [durable functions](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html), [durable execution FAQ](https://aws.amazon.com/lambda/faqs/)

**Interview-ready answer:** “The standard limit is 15 minutes. Eligible asynchronous and event-source invocations on Lambda Managed Instances can run for 90 minutes. Durable execution is a separate long-running workflow model.”

An API Gateway/client deadline remains independent. Moving a function to Managed Instances does not turn a synchronous HTTP request into a 90-minute job connection.

## 3. Default compute versus Managed Instances

| Dimension | Default compute | Lambda Managed Instances |
|---|---|---|
| Compute model | Lambda-managed execution fleet | EC2-based capacity providers in your account |
| Low/no traffic | On-demand scale-to-zero behavior | Configured baseline capacity remains relevant |
| Typical fit | Bursty/event-driven workloads | Predictable sustained traffic or specific hardware needs |
| Per-environment concurrency | One invocation at a time | Supports multiple concurrent invocations |
| Cost model | Requests and execution resource-duration | EC2-based capacity plus management charges |

Managed Instances changes thread-safety assumptions: globals, mutable caches, request-local data, and database clients must support concurrent invocations in one environment. Do not transfer default-compute isolation assumptions blindly. [Managed Instances architecture](https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances.html)

**Timeout handling also differs:** Managed Instances marks an invocation as failed at its deadline but does not forcibly stop that invocation's code. Check remaining time, stop work cooperatively, and use dependency deadlines. Otherwise a retry can overlap continuing side effects and occupied workers. [Managed Instances execution behavior](https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-execution-environment.html)

## 4. Invocation modes and who retries

| Mode | Examples | Retry owner |
|---|---|---|
| Synchronous request | Direct request/response, API integration | Caller/integration |
| Asynchronous invocation | SNS, supported service notifications | Lambda async queue after acceptance |
| Event source mapping | SQS, DynamoDB Streams, Kinesis and others | Mapping + source-specific semantics |

For ordinary asynchronous invocation, handler failures are retried twice by default. Throttling/system errors use a different retry path, up to six hours by default. Configure event age, retry count, and destinations/DLQ; duplicate delivery remains possible. [Async errors](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-error-handling.html)

For synchronous calls, Lambda does not apply the async retry policy. A caller timeout may leave execution running; retries need idempotency.

For streams, a bad record can delay progress on its shard. Configure supported partial failures, age/retry bounds, batch splitting, and failure handling according to the source.

## 5. Concurrency and backpressure

For default compute, approximate required concurrency as:

```text
concurrent invocations ≈ requests per second × average duration in seconds
Example: 200 requests/second × 0.25 seconds ≈ 50 concurrent invocations
```

Add headroom and validate tails/bursts. This is a planning approximation, not a capacity guarantee or the Managed Instances environment-count formula.

| Control | Purpose | Trap |
|---|---|---|
| Reserved concurrency | Reserves part of regional concurrency and caps the function | Does not pre-initialize environments |
| Provisioned concurrency | Keeps supported execution environments initialized | Costs money; excess traffic can use on-demand capacity |
| Source mapping concurrency | Bounds supported source consumers | Must align with function/downstream capacity |

Setting reserved concurrency to zero stops ordinary execution for that function. Protect a database by limiting incoming work, not by allowing unlimited function scale and hoping connections hold. Check compute-mode support for each control. [Reserved concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)

## 6. Cold starts, initialization, and SnapStart

Initialize reusable SDK clients outside the handler where appropriate, keep dependencies lean, and measure initialization separately from handler duration. Reuse is an optimization; environment persistence is not guaranteed.

Provisioned concurrency and SnapStart address startup differently. SnapStart restores supported published functions from initialized snapshots. Check runtime/feature compatibility and repair uniqueness, credentials, and network state after restore as required. Do not assume a random ID generated at snapshot time remains unique per execution environment. [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)

Avoid treating `/tmp`, global memory, or background threads after handler return as durable work storage. Persist important state externally.

## 7. SQS processing and idempotency

- Lambda polls the queue through an event source mapping and invokes batches.
- Set visibility to at least six times the function timeout plus the batching window, per AWS guidance.
- Return per-record failures with `ReportBatchItemFailures` where configured.
- Use the queue's redrive policy for repeated processing failures.
- Cap concurrency and make business effects idempotent before acknowledging success.

[SQS configuration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)

For a 90-minute eligible invocation, six times that timeout is nine hours before any batch-window margin, still subject to SQS's 12-hour ceiling. Long visibility also delays recovery, so assess whether smaller checkpointed jobs are better.

**Failure example:** a charge succeeds but the handler times out before returning. A retry must recover the recorded charge/outcome instead of charging again.

## 8. Networking and IAM

The execution role controls AWS API calls from code. The function resource policy controls supported invocation permissions. Being permitted to invoke a function does not give that caller the function's role credentials.

VPC attachment enables private-resource access; a public subnet does not automatically give the function a public IP or internet egress. Configure NAT or service endpoints as appropriate and review SG/DNS/routes.

Use Secrets Manager or other suitable managed configuration rather than hard-coded secrets. Reuse database connections carefully and evaluate RDS Proxy, but also bound concurrency and transaction duration.

## 9. Limits, packaging, and cost

Common default-compute limits to recognize: memory 128–10,240 MB; synchronous buffered request/response 6 MB each; asynchronous event 1 MB; streamed response up to 200 MB. Integration envelopes and upstream/downstream limits may reduce usable application payload. [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)

ZIP and container-image packaging are deployment choices; a container image does not make Lambda a continuously running ECS service. Match architecture/runtime support and deploy immutable artifacts through published versions/aliases.

Benchmark memory settings: more memory also changes CPU allocation for default compute and may reduce duration enough to improve cost. Include retries, idle provisioned capacity, Managed Instances capacity, logs, NAT, and downstream service charges.

## 10. Durable functions and alternatives

Durable functions checkpoint steps and suspend/resume workflows. Replay-aware code and supported SDK constructs are required; do not assume arbitrary side effects outside checkpointed steps are safe. Waiting workflow time differs from active compute time. [Durable programming model](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)

| Need | Consider |
|---|---|
| Short bursty event handler | Default Lambda compute |
| Longer eligible async job with sustained capacity needs | Managed Instances, within supported timeout |
| Waits, callbacks, multi-step durable code | Durable functions |
| Explicit service orchestration/state-machine visibility | Step Functions |
| Long arbitrary container/batch job | ECS/Fargate or AWS Batch |

Choose from workload and recovery requirements, not only the maximum possible timeout.

**Adjacent current capability:** Lambda MicroVMs is a separate isolated-execution primitive with its own lifecycle/API. A MicroVM can have a total lifespan of up to eight hours, including running and suspended time. That limit does not extend an ordinary Lambda function invocation. This guide's handler/retry/concurrency discussion concerns Lambda functions. [MicroVM lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/microvms-launching.html)

## 11. Troubleshooting and practical inspection

| Symptom | First checks |
|---|---|
| Throttles | Regional/function/source concurrency and downstream pressure |
| Timeouts | Dependency latency, network, connection pooling, CPU/memory, configured deadline |
| Errors after VPC attachment | NAT/endpoints, DNS, SG, routes |
| Duplicate processing | Source retry behavior and missing idempotency |
| DLQ empty despite SQS failures | Queue redrive versus function async DLQ confusion |
| Cold-start spikes | Initialization, traffic bursts, deployment/version, startup strategy |
| 90-minute setting rejected | Compute type, invocation path, unsupported mapping |

Read-only examples:

```bash
aws lambda get-function-configuration --function-name orders
aws lambda get-function-concurrency --function-name orders
aws lambda list-event-source-mappings --function-name orders
aws lambda get-function-event-invoke-config --function-name orders
aws lambda get-policy --function-name orders
```

Absent optional configuration can return a not-found response. Inspect `Duration`, `Errors`, `Throttles`, concurrency, async event age, and source backlog together.
Lambda can also run containers, provided that we have to use lambda as base image for the container.

---

## 12. Common Patterns

### Pattern 1: API Gateway → Lambda → DynamoDB

**Scenario:** RESTful API for managing orders with immediate response required.

```text
Architecture:
Client → API Gateway (REST API)
       → Lambda (synchronous invocation)
       → DynamoDB table

Invocation: Synchronous (client waits for response)
Timeout: Configure for p99 latency + buffer (e.g., 10 seconds)
Retry: Client retries on 5xx errors
```

**Execution Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Key decisions:**
- Use reserved concurrency to protect DynamoDB from throttling
- Implement idempotency using client-provided request ID
- Set API Gateway timeout < Lambda timeout to avoid ambiguous outcomes

---

### Pattern 2: S3 Event → Lambda → Image Processing → S3

**Scenario:** Automatically resize images uploaded to S3.

```text
Architecture:
User uploads image → S3 bucket (my-uploads/)
                   → S3 Event Notification
                   → Lambda (asynchronous invocation)
                   → Process image (resize)
                   → S3 bucket (my-processed/)

Invocation: Asynchronous (S3 doesn't wait)
Timeout: 5 minutes (image processing can take time)
Retry: Lambda retries 2× automatically on failure
DLQ: SQS queue for failed processing events
```

**Resource Policy (allows S3 to invoke Lambda):**
```json
{
  "Effect": "Allow",
  "Principal": {"Service": "s3.amazonaws.com"},
  "Action": "lambda:InvokeFunction",
  "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ImageProcessor",
  "Condition": {
    "StringEquals": {"AWS:SourceAccount": "123456789012"},
    "ArnLike": {"AWS:SourceArn": "arn:aws:s3:::my-uploads"}
  }
}
```

**Execution Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::my-uploads/*"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::my-processed/*"
    }
  ]
}
```

**Key decisions:**
- Set async DLQ to capture failed events for manual review
- Use `/tmp` for temporary image storage during processing
- Avoid circular trigger: processed images must go to a DIFFERENT bucket/prefix
- Configure S3 event filter to only trigger on `.jpg`, `.png` uploads

---

### Pattern 3: SQS Queue → Lambda → Process Messages

**Scenario:** Background worker processing order fulfillment messages.

```text
Architecture:
Order service → SQS queue (orders-queue)
              → Lambda polls queue (event source mapping)
              → Process order (call warehouse API, send email)
              → Delete message on success

Invocation: Event source mapping (Lambda polls)
Batch size: 10 messages per invocation
Visibility timeout: 6 × Lambda timeout (e.g., 360 seconds for 60s timeout)
Retry: Queue redrive policy → DLQ after 3 attempts
```

**Event Source Mapping Configuration:**
```bash
aws lambda create-event-source-mapping \
  --function-name OrderProcessor \
  --event-source-arn arn:aws:sqs:us-east-1:123456789012:orders-queue \
  --batch-size 10 \
  --maximum-batching-window-in-seconds 5 \
  --scaling-config MaximumConcurrency=10 \
  --function-response-types ReportBatchItemFailures
```

**Partial Batch Failure Handler (Python):**
```python
def lambda_handler(event, context):
    batch_item_failures = []
    
    for record in event['Records']:
        try:
            order = json.loads(record['body'])
            process_order(order)  # Your business logic
        except Exception as e:
            print(f"Failed to process message: {record['messageId']}, error: {e}")
            batch_item_failures.append({"itemIdentifier": record['messageId']})
    
    return {"batchItemFailures": batch_item_failures}
```

**Key decisions:**
- Enable `ReportBatchItemFailures` to avoid reprocessing successful messages
- Set reserved concurrency to prevent overwhelming downstream services
- Implement idempotency using message deduplication ID
- Monitor `ApproximateAgeOfOldestMessage` CloudWatch metric

---

### Pattern 4: EventBridge Scheduled Rule → Lambda → Cleanup Task

**Scenario:** Nightly cleanup of expired sessions from DynamoDB.

```text
Architecture:
EventBridge rule (cron: 0 2 * * ? *)
  → Lambda (asynchronous invocation)
  → Scan DynamoDB for expired items
  → BatchDeleteItem

Invocation: Asynchronous (EventBridge doesn't wait)
Timeout: 15 minutes (for large scans)
Concurrency: Reserved = 1 (prevent overlapping runs)
```

**EventBridge Rule:**
```json
{
  "ScheduleExpression": "cron(0 2 * * ? *)",
  "State": "ENABLED",
  "Targets": [{
    "Arn": "arn:aws:lambda:us-east-1:123456789012:function:SessionCleanup",
    "Id": "1",
    "RetryPolicy": {
      "MaximumRetryAttempts": 0
    },
    "DeadLetterConfig": {
      "Arn": "arn:aws:sqs:us-east-1:123456789012:cleanup-dlq"
    }
  }]
}
```

**Key decisions:**
- Set reserved concurrency = 1 to prevent concurrent cleanup runs
- Use pagination for large DynamoDB scans to avoid timeout
- Add CloudWatch alarm if function duration approaches timeout
- Consider Step Functions if cleanup needs to process millions of items

---

## 13. Quick Reference Card

### Lambda Limits by Compute Type

| Resource | Default Compute | Managed Instances | Notes |
|---|---|---|---|
| **Memory** | 128 MB – 10,240 MB | Same | CPU scales with memory |
| **Timeout (sync)** | 15 minutes | 15 minutes | Client may timeout earlier |
| **Timeout (async)** | 15 minutes | 90 minutes | Only eligible invocations |
| **Timeout (event source)** | 15 minutes | 90 minutes | Excludes MQ, DocumentDB |
| **/tmp storage** | 512 MB (default) – 10 GB | Same | Ephemeral, cleared after |
| **Environment variables** | 4 KB total | Same | Use Secrets Manager for large configs |
| **Deployment package (ZIP)** | 50 MB (zipped), 250 MB (unzipped) | Same | Use container for larger |
| **Container image** | 10 GB | Same | Must use Lambda base image |
| **Concurrent executions** | 1 per environment | Multiple per environment | Thread-safety critical for Managed |
| **Payload (sync buffered)** | 6 MB request, 6 MB response | Same | Use streaming for larger |
| **Payload (async)** | 256 KB | Same | S3 for larger payloads |
| **Layers** | 5 layers, 250 MB total unzipped | Same | Share common code |

### Concurrency Calculation Formula

```text
Required Concurrency ≈ (Requests per second) × (Average duration in seconds)

Examples:
├─ 100 req/s × 0.5s = 50 concurrent executions
├─ 1,000 req/s × 0.1s = 100 concurrent executions
└─ 10 req/s × 5s = 50 concurrent executions

Add 20-30% buffer for burst traffic and tail latencies.
```

### Invocation Type Decision Matrix

| Question | Answer → Invocation Type |
|---|---|
| Does caller need immediate response? | **Yes** → Synchronous |
| Can operation complete in < 15 minutes? | **No** → Consider Step Functions or ECS |
| Is it a user-facing API? | **Yes** → Synchronous (API Gateway) |
| Is it a background job? | **Yes** → Asynchronous or SQS event source |
| Do you need built-in retries? | **Yes** → Asynchronous or event source mapping |
| Processing S3/SNS events? | **N/A** → Asynchronous (automatic) |
| Processing SQS/Kinesis/DynamoDB Stream? | **N/A** → Event source mapping (automatic) |

### Cold Start Optimization Strategies

| Strategy | Use Case | Tradeoff |
|---|---|---|
| **Provisioned Concurrency** | Latency-sensitive APIs (< 200ms p99) | Ongoing cost for idle capacity |
| **SnapStart** | Java applications (long JVM startup) | Java-only; must handle restore uniqueness |
| **Smaller package** | All functions | Development effort to minimize dependencies |
| **ARM (Graviton2)** | General workloads | 20% faster boot + 34% better price/perf |
| **Keep warm** (scheduled ping) | Low-traffic functions | Wasteful; use provisioned concurrency instead |

### Common CLI Commands

```bash
# Invoke function synchronously with payload
aws lambda invoke \
  --function-name MyFunction \
  --payload '{"key":"value"}' \
  response.json

# Update function code from ZIP
aws lambda update-function-code \
  --function-name MyFunction \
  --zip-file fileb://function.zip

# Set reserved concurrency (limit + reserve)
aws lambda put-function-concurrency \
  --function-name MyFunction \
  --reserved-concurrent-executions 10

# Set provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name MyFunction \
  --qualifier prod \
  --provisioned-concurrent-executions 5

# Get function metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=MyFunction \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 60 \
  --statistics Average,Maximum

# Tail logs in real-time
aws logs tail /aws/lambda/MyFunction --follow
```

---

## 14. Exam Tips Summary

### Must-Know Concepts

1. **Timeout scoping:** 15 minutes is standard; 90 minutes only for Managed Instances async/event-source (not MQ/DocumentDB)
2. **Execution role vs resource policy:** Execution role = what Lambda can DO; resource policy = who can INVOKE
3. **Invocation retry ownership:** Sync = caller retries; Async = Lambda retries 2×; Event source = mapping config
4. **Concurrency types:** Reserved = cap + reserve; Provisioned = pre-initialized environments; they solve different problems
5. **Cold start mitigation:** Provisioned concurrency (costs money) vs SnapStart (Java-only, restore considerations)
6. **VPC Lambda:** Needs NAT or VPC endpoints for AWS API calls; public subnet ≠ internet access
7. **SQS visibility timeout:** Set to 6× function timeout + batching window to prevent duplicate processing
8. **Async DLQ vs SQS DLQ:** Function async DLQ captures Lambda failures; SQS redrive DLQ captures application processing failures
9. **Container image:** Must use Lambda base image; not a continuously running service like ECS
10. **Managed Instances threading:** Unlike default compute (1 invocation/environment), Managed Instances handle concurrent invocations in same environment — must be thread-safe
11. **Durable functions:** Workflow lifetime up to 1 year via checkpoints ≠ 1 year of continuous single invocation
12. **Lambda Layers:** Max 5 layers, 250 MB unzipped total; use for sharing common code (SDKs, libraries)

### Common Exam Scenarios

- **Lambda throttling at scale** → Check regional concurrency limit (1,000 default), reserved concurrency caps, downstream throttling
- **Timeout with external API** → Add connection pooling, set socket timeout, check VPC NAT capacity, validate DNS
- **Duplicate SQS processing** → Implement idempotency with message deduplication ID, check visibility timeout (6× function timeout)
- **High cost despite low traffic** → Check provisioned concurrency, long-running functions, excessive memory allocation
- **Cold starts hurting API latency** → Use provisioned concurrency (costs money) or optimize package size/dependencies
- **VPC Lambda can't reach S3** → Add S3 Gateway endpoint (free) or NAT Gateway (paid) for internet-routed paths
- **Cross-account Lambda invocation fails** → Need resource policy on target Lambda AND execution role permission in source account
- **Function can invoke but API calls fail** → Check execution role permissions, not resource policy
- **EventBridge schedule not triggering** → Verify rule is ENABLED, check Lambda resource policy allows `events.amazonaws.com`
- **90-minute timeout rejected** → Verify compute type (must be Managed Instances), invocation type (async/event-source), mapping type (not MQ/DocumentDB)

### Red Flags (Anti-patterns to avoid)

- ❌ **Hardcoding secrets in environment variables** → Use Secrets Manager with automatic rotation
- ❌ **Using `/tmp` for durable data** → Ephemeral storage cleared after execution; use S3/DynamoDB/EFS
- ❌ **No idempotency with async invocation** → Retries will duplicate side effects (payments, emails, orders)
- ❌ **Setting timeout to max "just in case"** → Wastes cost and delays error detection; set to p99 + small buffer
- ❌ **Reserved concurrency on bursty traffic** → Caps scale-out; use it only to protect downstream or isolate tenants
- ❌ **Creating new SDK client per invocation** → Initialize once in global scope, reuse across invocations
- ❌ **No batch failure reporting with SQS** → Entire batch retried even if only 1 message failed; enable `ReportBatchItemFailures`
- ❌ **VPC Lambda with no NAT/endpoints** → Can't reach AWS APIs, external internet, or even CloudWatch Logs
- ❌ **Assuming container = continuously running** → Lambda container still has invocation limits; use ECS for always-on services
- ❌ **Mixing sync and async retry expectations** → Sync invocations don't get Lambda's built-in retries; caller must retry

---

## 15. Interview Questions and Quick Revision

**Q1: What is Lambda's maximum runtime?**

The standard limit is 15 minutes for all invocation types on default compute. Lambda Managed Instances extends this to 90 minutes for eligible asynchronous invocations and event source mappings (excluding Amazon MQ and DocumentDB). Durable functions support workflow lifetimes up to 1 year through checkpointing, but that's not a single continuous invocation.

---

**Q2: Explain the difference between execution role and resource policy.**

The execution role is an IAM role that Lambda assumes when your handler runs. It grants permissions for your code to call AWS APIs (e.g., DynamoDB, S3, SQS). The resource policy controls who can invoke the Lambda function itself (e.g., allowing S3 bucket or API Gateway to trigger it). They're orthogonal: execution role = what Lambda can DO, resource policy = who can CALL Lambda.

---

**Q3: Reserved versus provisioned concurrency?**

Reserved concurrency allocates a portion of your regional concurrency limit to a function and caps its maximum concurrent executions — it does NOT pre-initialize environments. Provisioned concurrency keeps a specified number of execution environments warm and initialized to reduce cold starts. Reserved solves isolation/limiting problems; provisioned solves latency problems. They can be used together.

---

**Q4: Why does SNS success not guarantee the handler succeeded?**

When SNS invokes Lambda asynchronously, Lambda returns HTTP 202 Accepted immediately after queuing the event — before the handler runs. If the handler fails later, Lambda retries automatically (up to 2 times by default). SNS only knows that Lambda accepted the event, not whether processing ultimately succeeded. Check Lambda's async DLQ or CloudWatch metrics for actual failures.

---

**Q5: Can Lambda run a container indefinitely like ECS?**

No. Container image packaging is just a deployment format for Lambda — the function still has invocation time limits (15 or 90 minutes depending on configuration). Lambda is event-driven and stateless. For long-running always-on services, use ECS/Fargate or EC2.

---

**Q6: How do you make order creation retry-safe?**

Implement idempotency by persisting a stable business event ID (e.g., client-provided request ID) with the operation outcome in the same atomic transaction. On retry, check if that ID already exists — if so, return the stored result instead of creating a duplicate order. For external side effects like payments, use the payment provider's idempotency key mechanism and reconcile unknown outcomes (e.g., payment sent but response lost).

---

**Q7: Why is my Lambda function throttling when account concurrency limit isn't reached?**

Check these in order:
1. Function-level reserved concurrency cap (may be set lower than needed)
2. Event source mapping concurrency limit (e.g., SQS mapping MaxConcurrency setting)
3. Downstream service throttling (DynamoDB, external API) causing retries and backpressure
4. Regional concurrency burst limit (1,000 default burst, then gradual scale to account limit)
5. VPC ENI capacity (older VPC Lambda had scaling delays; current architecture mitigates this)

---

**Q8: When should I use provisioned concurrency?**

Use provisioned concurrency when:
- Your API has strict latency SLOs (e.g., p99 < 200ms) that cold starts violate
- You can predict traffic patterns and justify the ongoing cost
- The function handles user-facing synchronous requests

Don't use it for:
- Background async jobs (cold start doesn't matter)
- Functions invoked < 1/minute (cost outweighs benefit)
- When SnapStart is available and sufficient (Java functions)

---

**Q9: How does Lambda work with VPC resources?**

When you attach a Lambda to a VPC, it gets an ENI in your subnets and can access private resources (RDS, ElastiCache, internal ALBs). However, it loses direct internet and AWS API access. To restore this, add:
- **S3/DynamoDB:** VPC Gateway Endpoint (free, no internet)
- **Other AWS APIs:** VPC Interface Endpoints (hourly cost) or NAT Gateway (paid, uses internet)

Without NAT or endpoints, VPC Lambda can't reach CloudWatch Logs, Secrets Manager, or any internet service.

---

**Q10: Explain Lambda's async retry behavior.**

When a function invoked asynchronously fails (unhandled exception, timeout, out-of-memory), Lambda automatically retries twice (total of 3 attempts). Between retries, it uses exponential backoff (delay increases). Events are retained for 6 hours by default (configurable 1 minute – 6 hours). After max retries or event age is exceeded, the event can be sent to a dead-letter queue (SQS or SNS) for manual investigation.

---

**Q11: What is the Lambda SQS visibility timeout formula?**

AWS recommends: `Visibility Timeout = 6 × Function Timeout + Batching Window`

Why 6×? Lambda may retry the batch on transient errors, and each batch needs time to process. If visibility is too short, messages reappear while still being processed, causing duplicates. For a 60-second function timeout with 5-second batching window: `6 × 60 + 5 = 365 seconds`.

---

**Q12: How do cold starts differ between default compute and Managed Instances?**

Default compute cold starts occur when Lambda needs to provision a new execution environment — this includes downloading your code, starting the runtime, and running initialization code outside the handler. Each environment handles 1 invocation at a time.

Managed Instances environments can handle multiple concurrent invocations in the same environment, so cold starts are less frequent at scale (fewer total environments needed). However, your code MUST be thread-safe since globals and connections are shared.

---

**Q13: What's the difference between Lambda async DLQ and SQS redrive DLQ?**

Lambda async DLQ captures events that Lambda failed to invoke successfully (handler errors, timeouts) after exhausting retries. This is configured on the Lambda function's async invoke config.

SQS redrive DLQ captures messages that were successfully delivered to Lambda but the application failed to process them (after N ReceiveCount). This is configured on the SQS queue's redrive policy.

They're different failure boundaries: Lambda async DLQ = Lambda can't process event; SQS DLQ = application can't process message.

---

**Q14: When would you choose Lambda Layers?**

Use Lambda Layers to:
- Share common code/libraries across multiple functions (e.g., shared SDK, logging library)
- Separate application logic from dependencies (deploy code changes without re-uploading large dependencies)
- Stay under deployment package size limits (50 MB ZIP limit)

Limitations:
- Max 5 layers per function
- 250 MB total unzipped size (including all layers + function code)
- Layers are versioned and immutable (can't modify an existing layer version)

---

**Q15: How does SnapStart work and what are its gotchas?**

SnapStart takes a snapshot of an initialized Lambda execution environment (after initialization code runs but before handler invocation) and restores from that snapshot on future cold starts. This dramatically reduces Java cold start times.

Gotchas:
1. **Java-only** (not available for Python, Node.js, etc.)
2. **Uniqueness:** Random IDs, timestamps, or secrets generated during initialization are baked into the snapshot and reused — fix by regenerating after restore
3. **Network connections:** DB connections opened during init are stale after restore — reconnect in handler or use lazy initialization
4. **Published versions only:** Doesn't work with $LATEST
5. **Pricing:** Charged for snapshot storage (small cost)

---

## 16. AWS sources

- [September 2026 timeout announcement](https://aws.amazon.com/about-aws/whats-new/2026/09/aws-lambda-90-minute-function/)
- [Quotas and excluded mappings](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html)
- [Managed Instances](https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances.html)
- [Managed Instances timeout behavior](https://docs.aws.amazon.com/lambda/latest/dg/lambda-managed-instances-execution-environment.html)
- [Async retries](https://docs.aws.amazon.com/lambda/latest/dg/invocation-async-error-handling.html)
- [Reserved concurrency](https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html)
- [SnapStart](https://docs.aws.amazon.com/lambda/latest/dg/snapstart.html)
- [SQS integration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)
- [Durable functions](https://docs.aws.amazon.com/lambda/latest/dg/durable-functions.html)
- [Durable execution FAQ](https://aws.amazon.com/lambda/faqs/)
- [Separate MicroVM lifecycle](https://docs.aws.amazon.com/lambda/latest/dg/microvms-launching.html)

**Related:** [API Gateway](aws-api-gateway-cheatsheet.md), [SQS](aws-sqs-cheatsheet.md), [ECS](aws-ecs-cheatsheet.md).
