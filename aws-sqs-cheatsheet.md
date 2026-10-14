# Amazon SQS Cheat Sheet

**Interview preparation: durable queues, retries, visibility, FIFO, dead letters, and Lambda consumers.**

**Verified:** 2026-10-13. Throughput quotas depend on Region, queue mode, batching, and approved increases.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Standard versus FIFO](#2-standard-versus-fifo)
- [3. Current limits worth remembering](#3-current-limits-worth-remembering)
- [4. Visibility, delay, polling, and retention](#4-visibility-delay-polling-and-retention)
- [5. Idempotent processing pattern](#5-idempotent-processing-pattern)
- [6. FIFO grouping and head-of-line blocking](#6-fifo-grouping-and-head-of-line-blocking)
- [7. DLQ and replay](#7-dlq-and-replay)
- [8. Lambda integration](#8-lambda-integration)
- [9. Security, metrics, and cost](#9-security-metrics-and-cost)
- [10. Common patterns](#10-common-patterns)
- [11. Troubleshooting and practical inspection](#11-troubleshooting-and-practical-inspection)
- [12. Quick reference card](#12-quick-reference-card)
- [13. Exam tips summary](#13-exam-tips-summary)
- [14. Interview questions and detailed answers](#14-interview-questions-and-detailed-answers)
- [15. Message formats and Python and Java examples](#15-message-formats-and-python-and-java-examples)
- [16. AWS sources](#16-aws-sources)

---

## 1. Fundamentals and mental model

SQS decouples producers from consumers with a durable queue. Consumers pull messages and explicitly delete them after successful processing.

### Message Lifecycle with Visibility Timeout

```text
Producer → SQS Queue → Consumer Workflow

Detailed Flow:
┌─────────────────────────────────────────────────────────────────────────┐
│ Message States:                                                         │
│                                                                         │
│  1. SENT (Available)                                                    │
│     ↓ Producer sends message                                           │
│     Queue stores message (encrypted at rest)                           │
│     Message is VISIBLE to consumers                                    │
│                                                                         │
│  2. RECEIVED (In-Flight)                                                │
│     ↓ Consumer calls ReceiveMessage                                    │
│     Message becomes INVISIBLE to other consumers                       │
│     Visibility timeout starts (e.g., 30 seconds)                       │
│     Consumer processes message                                         │
│     ↓                                                                   │
│     ├─→ SUCCESS PATH                                                    │
│     │   Consumer calls DeleteMessage(ReceiptHandle)                    │
│     │   Message permanently removed                                    │
│     │   END ✓                                                          │
│     │                                                                   │
│     └─→ FAILURE PATH                                                    │
│         ├─ Consumer crashes / processing fails                         │
│         ├─ Consumer doesn't delete before timeout                      │
│         └─ Visibility timeout expires                                  │
│                                                                         │
│  3. VISIBLE AGAIN (Retry)                                               │
│     ↓ After visibility timeout expiry                                  │
│     Message becomes VISIBLE again                                      │
│     ApproximateReceiveCount += 1                                       │
│     Available for retry by same/different consumer                     │
│     ↓                                                                   │
│     If ReceiveCount > maxReceiveCount                                  │
│     → Move to Dead Letter Queue (DLQ)                                  │
│                                                                         │
│  4. DEAD LETTER (Poison Message)                                        │
│     Message in DLQ for manual inspection                               │
│     Retention period applies (up to 14 days)                           │
│     Requires manual intervention or redrive                            │
│                                                                         │
│  5. EXPIRED                                                             │
│     Message exceeds retention period (1 min - 14 days)                 │
│     Automatically deleted                                              │
└─────────────────────────────────────────────────────────────────────────┘

Visibility Timeout Lifecycle:
┌──────────┬────────────────────┬──────────────────────────────────┐
│ t=0      │ t=30s              │ t=60s                            │
│ Receive  │ Processing...      │ Timeout expires                  │
│ Message  │ Message INVISIBLE  │ → Message VISIBLE again          │
│          │                    │ → Available for retry            │
├──────────┴────────────────────┴──────────────────────────────────┤
│ If DeleteMessage called before t=30s → Removed permanently      │
│ If ChangeMessageVisibility called → Extend timeout to t+30s     │
└─────────────────────────────────────────────────────────────────┘

Key Concepts:
├─ ReceiptHandle: Unique identifier for THIS receive operation
│  └─ Changes on each receive, required for delete/visibility change
├─ ApproximateReceiveCount: How many times message was received
├─ MessageId: Persistent identifier (doesn't change)
└─ Visibility Timeout: Protection period for processing

Critical Rules:
1. Receiving ≠ Deleting (explicit delete required)
2. Receipt handle is unique per receive (use current handle)
3. Visibility timeout is per-message, per-receive
4. Multiple consumers can receive same message after timeout
5. Deleted messages cannot be recovered
```

**Mental Model Summary:**
- **Visible** = Available for consumers to receive
- **In-Flight** = Being processed (invisible to others)
- **Delete** = Permanent removal (success acknowledgment)
- **Timeout** = Automatic retry mechanism (failure recovery)

Receiving is not deleting. The receipt handle identifies a particular receipt and is used for deletion or visibility changes; use the current handle.

## 2. Standard versus FIFO

| Property | Standard | FIFO |
|---|---|---|
| Delivery | At-least-once | Deduplicated sends within documented window; retries still need correct consumers |
| Order | Best effort | Ordered within each message group |
| Parallelism | Many independent consumers | Different groups can run concurrently |
| Name | Ordinary queue name | Ends in `.fifo` |
| Typical use | Independent jobs, bulk work | Per-order/account state transitions |

FIFO's broker-level guarantees do not atomically commit your payment/database side effect with deletion. A crash after the side effect but before acknowledgement can cause processing to run again. [FIFO deduplication](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)

## 3. Current limits worth remembering

| Setting | Current value | Meaning |
|---|---|---|
| Message size | Up to **1 MiB** | Older 256 KiB maximum notes are outdated |
| Retention | 1 minute–14 days; default 4 days | Lifetime of undeleted messages |
| Visibility timeout | 0–12 hours; default 30 seconds | Hides a received message during work |
| Delay | Up to 15 minutes | Initial delivery postponement |
| Long-poll wait | Up to 20 seconds | Wait for messages instead of immediate empty response |
| Batch API entries | Up to 10 | Inspect each entry's result |
| FIFO deduplication window | 5 minutes | Does not cover arbitrary later replays |

[Message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html), [queue parameters](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-queue-parameters.html), [FIFO terms](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-key-terms.html)

The 12-hour visibility ceiling is measured from receipt; repeatedly extending visibility is not an unlimited lease. For very large payloads, store content in S3 and send a reference with coordinated retention/permissions.

## 4. Visibility, delay, polling, and retention

These settings solve different problems:

- **Delay:** make new work unavailable initially.
- **Visibility:** protect work currently being processed from normal redelivery.
- **Retention:** expire old work whether or not processing finishes.
- **Long polling:** reduce empty receives and response churn.

Set visibility above typical processing time plus reasonable margin, and extend it with a heartbeat for variable-duration jobs. Excessively short visibility causes concurrent duplicates; excessively long visibility delays recovery after a crash.

Long polling reduces empty receive calls but does not change ordering, guarantee exactly-once side effects, or make consumers push-based.

## 5. Idempotent processing pattern

```text
Receive(eventId, business operation)
  → validate
  → atomically apply effect + record eventId/outcome where possible
  → delete only after durable success
On retry: return recorded outcome, then acknowledge
```

For an external payment API, propagate a stable idempotency key and reconcile uncertain results. A standalone “mark processed before doing work” step can lose work if the process crashes in between.

Choose deduplication-record retention for the maximum replay horizon, not just the queue's five-minute producer deduplication window.

## 6. FIFO grouping and head-of-line blocking

Use `MessageGroupId` for the smallest business entity that needs ordering. Distinct groups run independently, while a failing message can block later messages in its group.

Content-based deduplication hashes the body, not message attributes. Two events with identical bodies but different meaningful attributes can be wrongly treated as duplicates unless you choose explicit IDs appropriately.

Strict order and dead-lettering have a tradeoff: moving a failed event out lets subsequent events progress, which may violate the intended business sequence. Decide whether to stop the group, compensate, or tolerate a gap.

## 7. DLQ and replay

Configure a redrive policy with a reasonable `maxReceiveCount` and monitor the DLQ. Do not use a very low count to classify transient throttling as poison data.

For standard queues, original enqueue time matters to expiry when a message reaches a DLQ; give the DLQ longer retention than the source. FIFO enqueue time resets on movement to a DLQ. Redrive only after correcting the cause and limit the replay rate. [SQS dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)

A DLQ is an operational workflow: alert, inspect, classify, fix, replay or deliberately discard with accountability. It is not successful processing.

## 8. Lambda integration

Lambda's event source mapping polls SQS and invokes the function with batches. This is different from Lambda's asynchronous invocation queue.

- Set visibility to at least **six times the function timeout**, plus any batching window, following AWS guidance.
- Enable `ReportBatchItemFailures` and return failed message IDs to avoid retrying successful records unnecessarily.
- For FIFO, preserve order by stopping after a failure and reporting failed/unprocessed records appropriately.
- Put processing-failure redrive on the **SQS source queue**; a function's async DLQ is not the same mechanism.
- Bound mapping/function concurrency to protect the database and other dependencies.

[SQS event source configuration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)

## 9. Security, metrics, and cost

Use IAM plus queue policies, encryption, TLS, and source restrictions for service publishers. Include KMS permissions where customer managed keys are used. Cross-account access requires correctly scoped permissions on both sides.

Key metrics: approximate visible/in-flight counts, oldest-message age, sent/received/deleted counts, and DLQ depth. Counts are approximate; a rising oldest age is often more actionable than raw depth.

Batching and long polling can lower request cost. Empty polling, repeated failures, large payload units, KMS requests, and downstream compute all influence the bill.

## 10. Common patterns

### Pattern 1: Decoupled microservices with SQS standard queue

**Scenario:** Order service writes orders, multiple workers process them independently.

```python
# Producer (Order Service)
import boto3
import json
from datetime import datetime

sqs = boto3.client('sqs', region_name='us-east-1')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/orders-queue'

def publish_order_event(order_id, customer_id, total):
    """Publish order created event to SQS"""
    message = {
        'eventId': f'order-{order_id}-{datetime.utcnow().isoformat()}',
        'eventType': 'OrderCreated',
        'orderId': order_id,
        'customerId': customer_id,
        'total': total,
        'timestamp': datetime.utcnow().isoformat()
    }
    
    response = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(message),
        MessageAttributes={
            'eventType': {
                'DataType': 'String',
                'StringValue': 'OrderCreated'
            }
        }
    )
    print(f"Published order {order_id}: {response['MessageId']}")
    return response['MessageId']

# Consumer (Worker Service)
import time

def process_orders():
    """Long-polling worker that processes orders"""
    while True:
        response = sqs.receive_message(
            QueueUrl=queue_url,
            MaxNumberOfMessages=10,
            WaitTimeSeconds=20,  # Long polling
            VisibilityTimeout=300,  # 5 minutes to process batch
            MessageAttributeNames=['All']
        )
        
        messages = response.get('Messages', [])
        if not messages:
            continue
            
        for message in messages:
            try:
                # Parse and process
                order = json.loads(message['Body'])
                process_order_idempotently(order)
                
                # Success: delete message
                sqs.delete_message(
                    QueueUrl=queue_url,
                    ReceiptHandle=message['ReceiptHandle']
                )
                print(f"Processed order {order['orderId']}")
                
            except Exception as e:
                # Failure: message becomes visible again after timeout
                print(f"Error processing {message['MessageId']}: {e}")
                # Optionally extend visibility if retryable
                # sqs.change_message_visibility(...)

def process_order_idempotently(order):
    """Process with idempotency check using eventId"""
    event_id = order['eventId']
    # Check if already processed (DynamoDB, database, etc.)
    # If yes: return success without reprocessing
    # If no: process and record event_id atomically
    pass
```

**Key points:**
- Long polling (20s) reduces empty responses
- Batch processing (10 messages) improves throughput
- Visibility timeout (300s) sized for batch processing time
- Explicit delete only after successful processing
- Idempotency using stable `eventId`

### Pattern 2: FIFO queue for order state transitions

**Scenario:** Ensure order events are processed in sequence per order.

```python
sqs = boto3.client('sqs', region_name='us-east-1')
fifo_queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/orders.fifo'

def publish_order_state_change(order_id, event_type, payload):
    """Publish ordered events per order"""
    event_id = f'{order_id}-{event_type}-{datetime.utcnow().isoformat()}'
    
    response = sqs.send_message(
        QueueUrl=fifo_queue_url,
        MessageBody=json.dumps({
            'eventId': event_id,
            'eventType': event_type,
            'orderId': order_id,
            'payload': payload,
            'timestamp': datetime.utcnow().isoformat()
        }),
        MessageGroupId=order_id,  # Group by order for ordering
        MessageDeduplicationId=event_id  # Dedup within 5-min window
    )
    return response['MessageId']

# Usage: events for same order processed in order
publish_order_state_change('order-123', 'OrderCreated', {...})
publish_order_state_change('order-123', 'PaymentReceived', {...})
publish_order_state_change('order-123', 'OrderShipped', {...})

# Different orders processed in parallel
publish_order_state_change('order-456', 'OrderCreated', {...})
```

**FIFO guarantees:**
- ✅ Events for `order-123` processed in publish order
- ✅ Events for `order-456` processed in parallel (different group)
- ✅ Duplicate sends within 5 minutes deduplicated
- ⚠️ Processing failure blocks later events in that group
- ⚠️ Still need idempotent processing for retries

### Pattern 3: Dead letter queue with alarm and replay

**Architecture:**
```
Source Queue (orders-queue)
  ├─ maxReceiveCount: 3
  ├─ Visibility: 5 minutes
  └─ DLQ: orders-dlq
        ├─ Retention: 14 days
        ├─ CloudWatch Alarm on ApproximateNumberOfMessagesVisible > 0
        └─ Manual inspection + replay workflow
```

**Setup via CLI:**

```bash
# Create DLQ
aws sqs create-queue \
  --queue-name orders-dlq \
  --attributes MessageRetentionPeriod=1209600  # 14 days

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders-dlq \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

# Create source queue with redrive policy
aws sqs create-queue \
  --queue-name orders-queue \
  --attributes '{
    "MessageRetentionPeriod": "345600",
    "VisibilityTimeout": "300",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\",\"maxReceiveCount\":\"3\"}"
  }'

# Create CloudWatch alarm
aws cloudwatch put-metric-alarm \
  --alarm-name orders-dlq-not-empty \
  --alarm-description "Alert when DLQ has messages" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=QueueName,Value=orders-dlq \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

**Replay from DLQ:**

```python
def inspect_dlq_messages(limit=10):
    """Inspect DLQ without deleting (for diagnosis)"""
    response = sqs.receive_message(
        QueueUrl=dlq_url,
        MaxNumberOfMessages=limit,
        WaitTimeSeconds=1,
        VisibilityTimeout=30,  # Short timeout for inspection
        MessageAttributeNames=['All']
    )
    
    for message in response.get('Messages', []):
        body = json.loads(message['Body'])
        receive_count = message['Attributes']['ApproximateReceiveCount']
        print(f"DLQ Message: {body['orderId']}, receiveCount={receive_count}")
        
        # Make visible again (inspection only)
        sqs.change_message_visibility(
            QueueUrl=dlq_url,
            ReceiptHandle=message['ReceiptHandle'],
            VisibilityTimeout=0  # Immediate visibility
        )

def redrive_from_dlq_to_source():
    """After fixing root cause, replay DLQ to source queue"""
    # Use console "Start DLQ redrive" or CLI:
    source_queue_arn = 'arn:aws:sqs:us-east-1:123456789012:orders-queue'
    
    response = sqs.start_message_move_task(
        SourceArn=f'{source_queue_arn}',  # Actually the DLQ ARN
        DestinationArn=source_queue_arn,
        MaxNumberOfMessagesPerSecond=10  # Throttle replay
    )
    print(f"Redrive task started: {response['TaskHandle']}")
```

### Pattern 4: Lambda event source mapping with partial batch failure

**Setup:**

```yaml
# Lambda function with SQS trigger (SAM template)
OrderProcessor:
  Type: AWS::Serverless::Function
  Properties:
    Runtime: python3.12
    Handler: app.lambda_handler
    Timeout: 300  # 5 minutes
    ReservedConcurrentExecutions: 10  # Limit concurrency
    Events:
      SQSEvent:
        Type: SQS
        Properties:
          Queue: !GetAtt OrdersQueue.Arn
          BatchSize: 10
          MaximumBatchingWindowInSeconds: 5
          FunctionResponseTypes:
            - ReportBatchItemFailures  # Enable partial batch responses
```

**Lambda handler with partial failures:**

```python
import json

def lambda_handler(event, context):
    """
    Process SQS batch with partial failure reporting.
    Lambda will only delete successfully processed messages.
    """
    failures = []
    
    for record in event['Records']:
        message_id = record['messageId']
        try:
            # Parse message body
            body = json.loads(record['body'])
            
            # Check receive count
            receive_count = int(record['attributes']['ApproximateReceiveCount'])
            if receive_count > 5:
                print(f"Message {message_id} received {receive_count} times, logging and skipping")
                # Could send to alternative DLQ or log to S3
                continue
            
            # Process order idempotently
            process_order(body)
            print(f"Successfully processed {body['orderId']}")
            
        except RetryableError as e:
            # Transient error: report failure, will retry
            print(f"Retryable error for {message_id}: {e}")
            failures.append({'itemIdentifier': message_id})
            
        except PermanentError as e:
            # Permanent error: log and skip (don't report failure)
            print(f"Permanent error for {message_id}: {e}")
            # Could send to alternative DLQ
            
        except Exception as e:
            # Unknown error: report failure to be safe
            print(f"Unknown error for {message_id}: {e}")
            failures.append({'itemIdentifier': message_id})
    
    # Return failures - Lambda will not delete these messages
    return {'batchItemFailures': failures}

class RetryableError(Exception):
    """Transient errors like throttling, temporary network issues"""
    pass

class PermanentError(Exception):
    """Permanent errors like validation failures, malformed data"""
    pass

def process_order(order):
    # Idempotent processing logic
    event_id = order['eventId']
    # Check if already processed, etc.
    pass
```

**Visibility timeout sizing:**
- Function timeout: 300s (5 min)
- Batch size: 10
- Batching window: 5s
- **Queue visibility timeout: 1800s (30 min)** = 6 × function timeout per AWS guidance

### Pattern 5: Cross-account queue access

**Scenario:** Account A (producer) sends to queue in Account B (consumer).

**Account B (123456789012) - Queue policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountASend",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::999988887777:root"
      },
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:shared-events"
    },
    {
      "Sid": "AllowLocalAccountAll",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:root"
      },
      "Action": "sqs:*",
      "Resource": "arn:aws:sqs:us-east-1:123456789012:shared-events"
    }
  ]
}
```

**Account A (999988887777) - IAM policy for producer role:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:SendMessage",
        "sqs:GetQueueUrl"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:shared-events"
    }
  ]
}
```

**Both policies required:** Queue policy (Account B) + IAM policy (Account A)

### Pattern 6: Delayed message for scheduled processing

**Use case:** Send reminder 15 minutes after cart abandonment.

```python
def send_delayed_reminder(cart_id, customer_email):
    """
    Send message with 15-minute delay (maximum SQS delay).
    For longer delays, use EventBridge Scheduler instead.
    """
    message = {
        'cartId': cart_id,
        'email': customer_email,
        'reminderType': 'cart-abandoned'
    }
    
    response = sqs.send_message(
        QueueUrl=reminders_queue_url,
        MessageBody=json.dumps(message),
        DelaySeconds=900,  # 15 minutes (max)
        MessageAttributes={
            'reminderType': {
                'DataType': 'String',
                'StringValue': 'cart-abandoned'
            }
        }
    )
    return response['MessageId']

# For delays > 15 minutes, use EventBridge Scheduler:
def schedule_reminder_long_delay(cart_id, delay_hours):
    """For delays beyond 15 minutes"""
    scheduler = boto3.client('scheduler')
    
    # Schedule event to publish to SQS at future time
    scheduler.create_schedule(
        Name=f'cart-reminder-{cart_id}',
        ScheduleExpression=f'at({delay_hours} hours from now)',  # Simplified
        Target={
            'Arn': reminders_queue_arn,
            'RoleArn': scheduler_role_arn,
            'Input': json.dumps({'cartId': cart_id})
        },
        FlexibleTimeWindow={'Mode': 'OFF'}
    )
```

**Limits:**
- SQS DelaySeconds: 0-900 seconds (15 minutes)
- For longer delays: EventBridge Scheduler → SQS

### Pattern 7: Priority queue simulation

**SQS does not have native priority queues.** Workaround: Use separate queues.

```python
# Architecture: 3 queues with different worker polling priorities
HIGH_PRIORITY_QUEUE = 'https://sqs...../orders-high'
MEDIUM_PRIORITY_QUEUE = 'https://sqs...../orders-medium'
LOW_PRIORITY_QUEUE = 'https://sqs...../orders-low'

def publish_with_priority(order, priority):
    """Route to appropriate queue based on priority"""
    queue_map = {
        'high': HIGH_PRIORITY_QUEUE,
        'medium': MEDIUM_PRIORITY_QUEUE,
        'low': LOW_PRIORITY_QUEUE
    }
    queue_url = queue_map.get(priority, MEDIUM_PRIORITY_QUEUE)
    
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order)
    )

def worker_with_priority_polling():
    """Worker polls high-priority queue more frequently"""
    while True:
        # Poll high priority 60% of the time
        for _ in range(6):
            process_queue(HIGH_PRIORITY_QUEUE, max_messages=10)
        
        # Poll medium priority 30% of the time
        for _ in range(3):
            process_queue(MEDIUM_PRIORITY_QUEUE, max_messages=10)
        
        # Poll low priority 10% of the time
        process_queue(LOW_PRIORITY_QUEUE, max_messages=5)

def process_queue(queue_url, max_messages):
    response = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=max_messages,
        WaitTimeSeconds=5
    )
    for message in response.get('Messages', []):
        # Process and delete
        pass
```

**Alternative:** Use message attributes + message filtering at consumer side (less efficient).

---

## 11. Troubleshooting and practical inspection

| Symptom | Investigate |
|---|---|
| Duplicates | Visibility, delete failures, crash timing, producer retries |
| Backlog growing | Consumer capacity, downstream bottleneck, throttling, poison messages |
| FIFO throughput low | Too few groups, blocked group, throughput mode/quotas |
| Empty receives despite work | In-flight messages, delay, short polling, group blocking |
| Messages disappear | Deletion, retention expiry, DLQ movement, wrong queue |

Read-only metadata inspection:

```bash
aws sqs get-queue-attributes --queue-url QUEUE_URL --attribute-names All
aws sqs list-dead-letter-source-queues --queue-url DLQ_URL
```

`ReceiveMessage` changes visibility state, so it is not a passive inspection command for production messages.

---

## 12. Quick reference card

### Queue type decision matrix

| Requirement | Standard | FIFO |
|-------------|----------|------|
| **Throughput** | Nearly unlimited (300 msgs/sec per API action, higher with batching) | 300 TPS (3,000 with batching), 300 TPS per group |
| **Ordering** | Best-effort | Strict order within message group |
| **Exactly-once** | No (at-least-once) | Deduplication within 5-min window |
| **Use case** | Independent tasks, high throughput, order doesn't matter | State machines, transactions, audit logs per entity |
| **Naming** | Any name | Must end in `.fifo` |
| **Delivery guarantee** | At-least-once (duplicates possible) | Exactly-once processing (within dedup window) |
| **Consumer model** | Competing consumers (any can process any message) | Per-group sequential processing |

### Message lifecycle states

```
┌────────────┬───────────────────────────────────────────────────────┐
│ State      │ Meaning                                               │
├────────────┼───────────────────────────────────────────────────────┤
│ AVAILABLE  │ Visible to consumers; can be received                 │
│ IN-FLIGHT  │ Received but not deleted; invisible during timeout    │
│ DELETED    │ Permanently removed; successful processing            │
│ VISIBLE    │ Timeout expired; available for retry                  │
│ DLQ        │ Moved to dead letter queue after max receives         │
│ EXPIRED    │ Retention period exceeded; automatically deleted      │
└────────────┴───────────────────────────────────────────────────────┘

State Transitions:
  AVAILABLE → (Receive) → IN-FLIGHT
  IN-FLIGHT → (Delete) → DELETED ✓
  IN-FLIGHT → (Timeout) → VISIBLE (retry)
  VISIBLE → (maxReceiveCount) → DLQ
  ANY → (Retention exceeded) → EXPIRED
```

### Key parameters and ranges

| Parameter | Range | Default | When to Change |
|-----------|-------|---------|----------------|
| **Message retention** | 1 min – 14 days | 4 days | Increase for async processing, decrease for ephemeral work |
| **Visibility timeout** | 0 – 12 hours | 30 seconds | Set to processing time + margin; too low = duplicates, too high = slow retry |
| **Delivery delay** | 0 – 15 minutes | 0 | Postpone initial delivery (scheduled work, rate limiting) |
| **Max message size** | 1 byte – 256 KB (1 MiB since 2024) | N/A | Use S3 for large payloads with SQS Extended Client |
| **Receive wait time** | 0 – 20 seconds | 0 (short polling) | Set 10-20s for long polling (reduce empty receives) |
| **Batch size** | 1 – 10 messages | 1 | Increase for throughput; decrease for granular error handling |
| **maxReceiveCount** | 1 – 1000 | N/A | Set based on retry tolerance (3-5 typical) |

### Common API operations

| Operation | Purpose | Key Parameters | Returns |
|-----------|---------|----------------|---------|
| **SendMessage** | Send single message | QueueUrl, MessageBody, (MessageGroupId/DeduplicationId for FIFO) | MessageId, MD5OfBody |
| **SendMessageBatch** | Send up to 10 messages | QueueUrl, Entries (list of messages) | Successful + Failed lists |
| **ReceiveMessage** | Retrieve 1-10 messages | QueueUrl, MaxNumberOfMessages, WaitTimeSeconds, VisibilityTimeout | Messages (with ReceiptHandle) |
| **DeleteMessage** | Acknowledge success | QueueUrl, ReceiptHandle | Status |
| **DeleteMessageBatch** | Acknowledge up to 10 | QueueUrl, Entries (list of handles) | Successful + Failed lists |
| **ChangeMessageVisibility** | Extend processing time | QueueUrl, ReceiptHandle, VisibilityTimeout | Status |
| **PurgeQueue** | Delete all messages | QueueUrl | Status (60s cooldown) |
| **GetQueueAttributes** | Read metadata | QueueUrl, AttributeNames | Attributes (depth, age, etc.) |

### Critical attributes to monitor

| CloudWatch Metric | What It Means | Alert Threshold |
|-------------------|---------------|-----------------|
| **ApproximateNumberOfMessagesVisible** | Messages waiting to be processed | > 1000 (backlog growing) |
| **ApproximateNumberOfMessagesNotVisible** | Messages in-flight | > 2× normal (stuck processing) |
| **ApproximateAgeOfOldestMessage** | How long oldest message has waited | > 5 minutes (consumer lag) |
| **NumberOfMessagesSent** | Producer throughput | Sudden drop (producer failure) |
| **NumberOfMessagesReceived** | Consumer polling rate | Sudden drop (consumer failure) |
| **NumberOfMessagesDeleted** | Successful processing rate | < Received (processing failures) |
| **ApproximateNumberOfMessagesVisible (DLQ)** | Poison messages | > 0 (needs investigation) |

### Visibility timeout sizing guide

```
Formula: Visibility Timeout = Processing Time + Safety Margin

Examples:
┌──────────────────────────────┬────────────────┬──────────────────┐
│ Scenario                     │ Processing     │ Visibility       │
├──────────────────────────────┼────────────────┼──────────────────┤
│ Fast API call (1-5s)         │ 5s             │ 30s (default)    │
│ Database write (10-20s)      │ 20s            │ 60s              │
│ Batch processing (1-2 min)   │ 120s           │ 300s (5 min)     │
│ Lambda with batching         │ Function TO    │ 6 × Function TO  │
│ Variable duration work       │ Varies         │ Use heartbeat +  │
│                              │                │ ChangeVisibility │
└──────────────────────────────┴────────────────┴──────────────────┘

Too Short → Duplicate processing (message visible before completion)
Too Long → Slow retry (must wait for timeout before retry)
```

### Idempotency patterns

| Pattern | Implementation | Pros | Cons |
|---------|----------------|------|------|
| **Database unique constraint** | INSERT with unique eventId | Atomic, durable | Requires database |
| **DynamoDB conditional write** | PutItem with ConditionExpression on eventId | Atomic, scalable | Eventual consistency concerns |
| **Redis SET NX** | SET key NX (set if not exists) | Fast | Not durable, TTL management |
| **Application state check** | Query before process, skip if exists | Simple | Race conditions, not atomic |
| **Idempotent operation** | Design operation to be naturally idempotent | No state needed | Not always possible |

**Best practice:** Atomic operation + state recording in single transaction.

### Error codes and causes

| Error | Cause | Resolution |
|-------|-------|------------|
| **ReceiptHandleIsInvalid** | Expired receipt handle, already deleted, or wrong queue | Use current receipt handle from latest receive |
| **MessageNotInflight** | Message visibility timeout expired | Expected for retry; process and delete again |
| **AWS.SimpleQueueService.NonExistentQueue** | Queue doesn't exist or wrong region | Verify queue URL and region |
| **AccessDenied** | Missing IAM/queue policy permissions | Check IAM policy + queue policy (both required for cross-account) |
| **OverLimit** | Exceeded API rate limit | Implement backoff, batch operations, request limit increase |
| **InvalidParameterValue** | Invalid parameter (e.g., negative timeout) | Validate parameters against documented ranges |
| **QueueDeletedRecently** | Queue deleted within 60-second cooldown | Wait 60s before recreating with same name |

### FIFO-specific quick reference

| Concept | Purpose | Notes |
|---------|---------|-------|
| **MessageGroupId** | Ordering scope | Messages with same group ID processed in order |
| **MessageDeduplicationId** | Deduplication scope | Prevents duplicate sends within 5-min window |
| **Content-based dedup** | Auto-generate dedup ID from body hash | Enable on queue; body must be unique |
| **Group parallelism** | Different groups processed concurrently | Use entity ID (orderId, userId) as group ID |
| **Head-of-line blocking** | Failed message blocks group | Move to DLQ or fix to unblock subsequent messages |
| **Throughput** | 300 TPS per group, 3,000 with batching | Use more groups for higher throughput |

### Cost optimization checklist

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| ✅ **Enable long polling** | 50-90% fewer requests | Set ReceiveMessageWaitTimeSeconds=20 |
| ✅ **Use batch operations** | 10× fewer API calls | SendMessageBatch, DeleteMessageBatch (10 messages) |
| ✅ **Right-size visibility timeout** | Reduce retries/duplicates | Match processing time, extend with heartbeat |
| ✅ **Set appropriate retention** | Reduce storage cost | Lower retention for ephemeral data (default 4 days) |
| ✅ **Monitor and purge DLQ** | Avoid unnecessary storage | Alert on DLQ depth, investigate and purge |
| ✅ **Use Standard over FIFO when possible** | Lower per-request cost | FIFO costs more, use only when ordering required |
| ✅ **Optimize maxReceiveCount** | Balance retries vs DLQ cost | 3-5 typical; too low = premature DLQ |

### SQS vs other messaging services

| Use Case | SQS Standard | SQS FIFO | SNS | EventBridge | Kinesis |
|----------|-------------|----------|-----|-------------|---------|
| **Decouple microservices** | ✅ Best | ⚠️ If order needed | ❌ Push model | ⚠️ Event routing | ❌ Stream |
| **Ordered processing** | ❌ No | ✅ Yes | ❌ No | ❌ No | ✅ Yes |
| **Fan-out (1→N)** | ❌ | ❌ | ✅ Best | ✅ With rules | ⚠️ Multiple consumers |
| **Exactly-once** | ❌ | ⚠️ 5-min window | ❌ | ❌ | ❌ |
| **Durability** | ✅ | ✅ | ⚠️ Retry only | ⚠️ Event archive | ✅ 1-365 days |
| **Replay** | ❌ One-time | ❌ One-time | ❌ | ✅ Archive | ✅ Yes |
| **Throughput** | Unlimited | 300/group | Unlimited | High | Very high |
| **Pull model** | ✅ | ✅ | ❌ Push | ❌ Push | ✅ |

---

## 13. Exam tips summary

### Must-know concepts

1. **Visibility timeout ≠ message retention.** Visibility hides an in-flight message temporarily (0-12 hours); retention determines total message lifetime (1 min - 14 days).

2. **Receiving ≠ Deleting.** Receiving makes a message invisible (in-flight). You must explicitly delete it to acknowledge successful processing.

3. **Receipt handle changes per receive.** The receipt handle identifies THIS receive operation. Use the current handle for delete/visibility changes.

4. **Long polling reduces costs.** Setting WaitTimeSeconds=20 reduces empty responses and API call costs by 50-90%.

5. **FIFO throughput = 300 TPS per group** (3,000 with batching). Use multiple MessageGroupIds for higher throughput.

6. **FIFO deduplication window = 5 minutes.** Not suitable for preventing arbitrary replays hours/days later.

7. **DLQ requires redrive policy on source queue.** Set maxReceiveCount (typical: 3-5). DLQ should have longer retention than source.

8. **Batch operations = 10× efficiency.** SendMessageBatch, DeleteMessageBatch, ChangeMessageVisibilityBatch reduce API calls.

9. **Lambda visibility timeout = 6 × function timeout.** Per AWS guidance for event source mapping.

10. **Message size limit = 256 KB (extended to 1 MiB in 2024).** For larger payloads, use SQS Extended Client Library (stores in S3).

11. **Delay queue postpones initial delivery** (0-15 min). Not a scheduling service—use EventBridge Scheduler for longer delays.

12. **Standard = at-least-once, FIFO = exactly-once dedup** (within 5-min window). Both still need idempotent consumers.

13. **Cross-account requires both policies.** Queue policy (resource side) + IAM policy (caller side).

14. **PurgeQueue has 60-second cooldown.** Cannot create, delete, or purge the same queue name again within 60s.

15. **CloudWatch metrics are approximate.** Exact ordering of operations not guaranteed in metrics.

### Common exam scenarios

| Scenario | Correct Answer | Wrong Answers to Avoid |
|----------|----------------|------------------------|
| Decouple order processing from web tier | SQS Standard queue | ❌ SNS (push model), ❌ Direct API call |
| Process bank transactions in order per account | SQS FIFO with MessageGroupId=accountId | ❌ Standard queue, ❌ SNS |
| Handle temporary DynamoDB throttling | Increase visibility timeout, implement exponential backoff | ❌ Immediate DLQ, ❌ Reduce batch size |
| Reduce SQS costs with many empty polls | Enable long polling (WaitTimeSeconds=20) | ❌ Increase batch size, ❌ Use FIFO |
| Process messages exactly once | FIFO + idempotent consumer | ❌ Standard queue alone, ❌ FIFO alone |
| Fan out to 10 independent services | SNS topic → 10 SQS queues | ❌ One SQS queue with 10 consumers |
| Handle poison messages | DLQ with CloudWatch alarm | ❌ Increase maxReceiveCount to 100, ❌ Delete on error |
| Cross-account queue access | Queue policy + IAM policy | ❌ IAM policy alone, ❌ Queue policy alone |
| Schedule message for 2 hours later | EventBridge Scheduler → SQS | ❌ SQS DelaySeconds (max 15 min) |
| Lambda processing SQS batch fails partially | Enable ReportBatchItemFailures | ❌ Throw exception (retries all), ❌ Delete all |
| Reduce duplicate processing | Increase visibility timeout to match processing time | ❌ Use FIFO, ❌ Reduce retention |
| Messages disappearing without processing | Check if being deleted by another consumer, DLQ, or retention expired | ❌ Increase visibility timeout |

### Red flags (wrong answer indicators)

- ❌ **"Use SNS when consumers should pull messages"** → SNS is push; use SQS for pull
- ❌ **"FIFO guarantees exactly-once side effects"** → FIFO prevents duplicate sends, not processing
- ❌ **"Increase maxReceiveCount to prevent messages going to DLQ"** → Hides poison messages
- ❌ **"Set very high visibility timeout to prevent duplicates"** → Delays retry on failure
- ❌ **"Standard queue guarantees order"** → Best-effort only; use FIFO for guaranteed order
- ❌ **"Delete message before processing"** → Loses message if processing fails
- ❌ **"Use single FIFO queue for high throughput"** → Limited to 300 TPS per group
- ❌ **"SQS guarantees exactly-once delivery"** → Only FIFO dedup within 5-min window
- ❌ **"Visibility timeout controls retention"** → Retention controls lifetime; visibility controls retry
- ❌ **"One queue for fan-out to multiple consumers"** → Use SNS topic → multiple queues

### Quick decision flowchart

```
Need messaging service?
  │
  ├─ Need strict ordering per entity?
  │  └─ YES → SQS FIFO (MessageGroupId per entity)
  │
  ├─ Need fan-out to multiple services?
  │  └─ YES → SNS topic → SQS queues (one per service)
  │
  ├─ Need event routing based on content?
  │  └─ YES → EventBridge
  │
  ├─ Need stream processing / replay?
  │  └─ YES → Kinesis Data Streams
  │
  └─ Default: Task queue, decouple producers/consumers
     └─ SQS Standard queue

SQS Standard vs FIFO?
  │
  ├─ Order critical per entity? → FIFO
  ├─ Need highest throughput? → Standard
  ├─ Duplicate messages acceptable? → Standard
  └─ Need producer deduplication? → FIFO
```

---

## 14. Interview questions and detailed answers

## 14. Interview questions and detailed answers

**Q1: Why use SQS instead of directly calling the worker service?**

SQS provides three key benefits that direct calls cannot:

1. **Buffering and burst handling:** If 1,000 orders arrive in 10 seconds but workers can only process 100/second, direct calls would fail or timeout. SQS queues the burst and workers process at their sustainable rate.

2. **Failure isolation:** If the worker crashes or is being deployed, producers continue publishing successfully. Messages wait until workers recover. With direct calls, producers must handle retries, circuit breaking, and request queuing themselves.

3. **Independent scaling:** Producers and consumers scale independently. You can scale workers based on queue depth without coordinating with producers. With direct calls, scaling requires service discovery updates, connection pool resizing, and load balancer reconfiguration.

**Tradeoff:** Eventual consistency. SQS delivery is asynchronous—you don't know when processing completes. If you need synchronous responses or sub-second latency, use direct calls with SQS as a backup for failed/slow requests.

---

**Q2: How do you achieve exactly-once processing with SQS?**

**Short answer:** You cannot rely on SQS alone—you must implement idempotent consumers.

**Detailed explanation:**

FIFO queues prevent duplicate *sends* within a 5-minute deduplication window, but they cannot prevent duplicate *processing*. Three failure scenarios cause duplicates:

1. **Consumer crashes after processing but before deleting:** Message becomes visible again after timeout; another consumer processes it.
2. **Network timeout during delete:** Processing succeeded but delete call failed/timed out; message retried.
3. **Processing takes longer than visibility timeout:** Message becomes visible before first consumer finishes.

**Solution:** Idempotent processing pattern:

```python
def process_message_idempotently(message):
    event_id = message['eventId']  # Stable business identifier
    
    # Atomic check-and-set with database
    try:
        # Option 1: Database unique constraint
        db.execute(
            "INSERT INTO processed_events (event_id, result, processed_at) VALUES (?, ?, NOW())",
            [event_id, "success"]
        )
        # First time: insert succeeds, process order
        process_order(message)
        
    except UniqueViolationException:
        # Already processed: skip processing, still return success
        logger.info(f"Event {event_id} already processed, skipping")
    
    # Delete message either way (processed now or previously)
    sqs.delete_message(ReceiptHandle=message['ReceiptHandle'])
```

**Key principles:**
- Use a stable business identifier (eventId), not SQS MessageId
- Record processing outcome atomically with business transaction
- Always delete message after idempotent check (whether first or duplicate processing)
- DynamoDB conditional writes or database unique constraints provide atomicity

---

**Q3: Should you use one queue or one queue per subscriber?**

**Depends on subscription pattern:**

**One queue, multiple consumers (competing consumers):**
- Use case: Independent workers processing the same task type
- Example: 10 EC2 instances all processing order fulfillment
- Behavior: Each message consumed by exactly one worker
- Scaling: Add more consumers; they compete for messages

```
Order Queue → [Worker 1] processes message A
           ├→ [Worker 2] processes message B
           └→ [Worker 3] processes message C
```

**One queue per subscriber (fan-out):**
- Use case: Multiple services need the same event independently
- Example: Order created → send email, update analytics, trigger shipping
- Implementation: SNS topic → 3 SQS queues (one per service)
- Behavior: Each service processes every message independently

```
SNS Topic (OrderCreated) 
  ├→ Email Queue → Email Service processes all orders
  ├→ Analytics Queue → Analytics Service processes all orders
  └→ Shipping Queue → Shipping Service processes all orders
```

**Wrong approach:** One SQS queue with 3 consumers expecting each to see all messages → Only one consumer receives each message (competing consumers pattern).

---

**Q4: How do you handle messages that need to be processed tomorrow?**

**Short answer:** SQS DelaySeconds supports maximum 15 minutes. For longer delays, use EventBridge Scheduler.

**Solutions by delay duration:**

| Delay | Solution |
|-------|----------|
| < 15 minutes | SQS DelaySeconds parameter |
| 15 min – 1 year | EventBridge Scheduler → SQS |
| Dynamic/complex scheduling | Step Functions with Wait state → SQS |

**Example: 24-hour reminder**

```python
import boto3
from datetime import datetime, timedelta

scheduler = boto3.client('scheduler')
sqs = boto3.client('sqs')

def schedule_reminder_for_tomorrow(user_id, cart_id):
    """Schedule reminder to be sent to SQS in 24 hours"""
    target_time = datetime.utcnow() + timedelta(hours=24)
    
    # Create one-time schedule
    scheduler.create_schedule(
        Name=f'cart-reminder-{cart_id}',
        ScheduleExpression=f'at({target_time.strftime("%Y-%m-%dT%H:%M:%S")})',
        FlexibleTimeWindow={'Mode': 'OFF'},
        Target={
            'Arn': 'arn:aws:sqs:us-east-1:123456789012:reminders-queue',
            'RoleArn': 'arn:aws:iam::123456789012:role/EventBridgeToSQS',
            'SqsParameters': {
                'MessageGroupId': user_id  # If FIFO queue
            },
            'Input': json.dumps({
                'userId': user_id,
                'cartId': cart_id,
                'reminderType': 'cart-abandoned'
            })
        },
        ActionAfterCompletion='DELETE'  # Auto-delete after execution
    )
```

**For recurring schedules:** Use EventBridge Rules with cron expressions instead of Scheduler.

---

**Q5: What if processing a message takes longer than 12 hours (maximum visibility timeout)?**

**Short answer:** SQS is not designed for long-running jobs. Break the work into stages or use a workflow service.

**Solutions:**

**Option 1: Checkpointed stages**
```python
def process_large_job(message):
    job_id = message['jobId']
    
    # Stage 1: Extract (publish stage 2 message)
    data = extract_data(job_id)
    sqs.send_message(QueueUrl=transform_queue, MessageBody=json.dumps({
        'jobId': job_id,
        'stage': 'transform',
        'dataLocation': save_to_s3(data)
    }))
    
# Separate consumer for stage 2
def transform_stage(message):
    # Stage 2: Transform (publish stage 3 message)
    pass
```

**Option 2: Heartbeat with visibility extension**
```python
import threading

def process_with_heartbeat(message, queue_url):
    receipt_handle = message['ReceiptHandle']
    processing_complete = threading.Event()
    
    def extend_visibility():
        while not processing_complete.is_set():
            time.sleep(300)  # Every 5 minutes
            if not processing_complete.is_set():
                sqs.change_message_visibility(
                    QueueUrl=queue_url,
                    ReceiptHandle=receipt_handle,
                    VisibilityTimeout=600  # Extend by 10 minutes
                )
    
    heartbeat_thread = threading.Thread(target=extend_visibility)
    heartbeat_thread.start()
    
    try:
        # Long-running processing
        process_job(message)
        sqs.delete_message(QueueUrl=queue_url, ReceiptHandle=receipt_handle)
    finally:
        processing_complete.set()
        heartbeat_thread.join()
```

**Option 3: Step Functions (recommended for complex workflows)**
```yaml
# Step Functions definition
StartAt: StartJob
States:
  StartJob:
    Type: Task
    Resource: arn:aws:lambda:...:start-job
    Next: WaitForCompletion
  
  WaitForCompletion:
    Type: Wait
    Seconds: 3600  # Wait 1 hour
    Next: CheckStatus
  
  CheckStatus:
    Type: Task
    Resource: arn:aws:lambda:...:check-status
    Next: IsComplete
  
  IsComplete:
    Type: Choice
    Choices:
      - Variable: $.status
        StringEquals: "COMPLETE"
        Next: Success
    Default: WaitForCompletion
```

**Maximum visibility extension limit:** While you can extend repeatedly, the receipt handle becomes invalid after 12 hours from initial receive.

---

**Q6: Explain the relationship between visibility timeout, message retention, and maxReceiveCount.**

These three parameters control different aspects of message lifecycle:

**1. Visibility Timeout (0 – 12 hours, default 30s)**
- **What:** How long a message is hidden after being received
- **Purpose:** Prevent other consumers from processing the same message while it's being worked on
- **Lifecycle impact:** Per-receive; resets each time message is received

```
Receive → Invisible for 30s → (timeout) → Visible again → Receive → Invisible for 30s
```

**2. Message Retention (1 min – 14 days, default 4 days)**
- **What:** Total lifetime of a message from send to automatic deletion
- **Purpose:** Prevent unbounded queue growth for unprocessed messages
- **Lifecycle impact:** Starts at SendMessage; message deleted after retention regardless of state

```
Day 0: Message sent
Day 4 (default): Message deleted automatically (whether processed or not)
```

**3. maxReceiveCount (1 – 1000, set on redrive policy)**
- **What:** Maximum times a message can be received before moving to DLQ
- **Purpose:** Detect poison messages that repeatedly fail processing
- **Lifecycle impact:** Increments ApproximateReceiveCount on each receive; triggers DLQ move

```
Receive #1 → Fail → Timeout → Visible
Receive #2 → Fail → Timeout → Visible
Receive #3 → Fail → Timeout → Move to DLQ
```

**Common mistake:** Setting visibility timeout = processing time but forgetting about retries.

**Example scenario:**
- Processing time: 60 seconds
- Visibility timeout: 60 seconds
- maxReceiveCount: 3
- Message retention: 4 days

**Timeline:**
```
00:00 → Receive #1, invisible for 60s
01:00 → Processing fails, visible again
01:05 → Receive #2, invisible for 60s
02:05 → Processing fails, visible again
02:10 → Receive #3, invisible for 60s
03:10 → Processing fails, ApproximateReceiveCount = 3
03:10 → Moved to DLQ (not visible in source queue anymore)

DLQ retention starts fresh (not inherited from source)
```

**Best practices:**
- Visibility timeout: processing time × 1.5 to 2
- maxReceiveCount: 3-5 (balance between retries and poison message detection)
- DLQ retention: ≥ source retention (for standard queue, due to enqueue time inheritance)

---

**Q7: How does FIFO MessageGroupId affect throughput and ordering?**

**MessageGroupId creates independent ordering boundaries:**

```python
# Same group ID → strict ordering, sequential processing
sqs.send_message(
    QueueUrl=fifo_queue,
    MessageBody=json.dumps({'action': 'create_order', 'orderId': '123'}),
    MessageGroupId='order-123',  # All order-123 events use same group
    MessageDeduplicationId='evt-1'
)

sqs.send_message(
    QueueUrl=fifo_queue,
    MessageBody=json.dumps({'action': 'confirm_payment', 'orderId': '123'}),
    MessageGroupId='order-123',  # Must be processed AFTER create_order
    MessageDeduplicationId='evt-2'
)
```

**Throughput impact:**

| Scenario | Throughput | Explanation |
|----------|------------|-------------|
| **Single MessageGroupId** | 300 TPS (3,000 with batching) | All messages sequential; one-at-a-time processing |
| **10 different MessageGroupIds** | 3,000 TPS (30,000 with batching) | 10 independent streams; parallel processing |
| **1,000 different MessageGroupIds** | 300,000 TPS (with batching) | Nearly unlimited parallelism |

**Example: E-commerce order processing**

```python
# WRONG: One group for all orders (bottleneck)
MessageGroupId = "all-orders"  # Max 300 TPS

# RIGHT: Group per order (parallel processing)
MessageGroupId = order_id  # Each order processed independently
```

**Head-of-line blocking:**

```
Messages in group "order-123":
  1. CreateOrder (processing...)
  2. ConfirmPayment (waiting)
  3. ShipOrder (waiting)
  4. DeliverOrder (waiting)

If CreateOrder fails and goes to DLQ:
  → Messages 2, 3, 4 are BLOCKED until #1 moves to DLQ
  → OR: Leave #1 in queue, blocking forever

Solution: Set appropriate maxReceiveCount to unblock group
```

**Group ID selection strategy:**

| Entity Type | Group ID | Parallelism |
|-------------|----------|-------------|
| **Account transactions** | `account-{accountId}` | Per account |
| **User sessions** | `session-{sessionId}` | Per session |
| **IoT device updates** | `device-{deviceId}` | Per device |
| **Multi-tenant processing** | `tenant-{tenantId}` | Per tenant |
| **Workflow steps** | `workflow-{workflowId}` | Per workflow |

**Anti-pattern:** Using MessageGroupId for load balancing or routing (use Standard queue instead).

---

**Q8: Walk through debugging: "Messages are being processed multiple times."**

**Debugging checklist:**

**1. Check visibility timeout vs processing time**
```bash
# Get queue attributes
aws sqs get-queue-attributes \
  --queue-url $QUEUE_URL \
  --attribute-names VisibilityTimeout

# If visibility timeout < processing time → duplicates expected
# Solution: Increase visibility timeout
aws sqs set-queue-attributes \
  --queue-url $QUEUE_URL \
  --attributes VisibilityTimeout=300  # 5 minutes
```

**2. Check if message is being deleted**
```python
# Add logging to confirm delete is called
def process_message(message):
    message_id = message['MessageId']
    try:
        do_work(message)
        sqs.delete_message(
            QueueUrl=queue_url,
            ReceiptHandle=message['ReceiptHandle']
        )
        logger.info(f"✓ Deleted message {message_id}")  # Check logs
    except Exception as e:
        logger.error(f"✗ Failed to process {message_id}: {e}")
        # Message NOT deleted → will be retried
```

**3. Check ApproximateReceiveCount**
```python
# High receive count indicates repeated failures
for message in messages:
    receive_count = int(message['Attributes']['ApproximateReceiveCount'])
    if receive_count > 3:
        logger.warning(f"Message {message['MessageId']} received {receive_count} times")
```

**4. Check for multiple consumers with different configurations**
```bash
# Lambda event source mapping visibility timeout
aws lambda get-event-source-mapping \
  --uuid <mapping-uuid> \
  --query 'FunctionResponseTypes'

# Should include 'ReportBatchItemFailures'
# Without it: partial failures cause entire batch retry
```

**5. Check idempotency implementation**
```python
# Missing idempotency check
def process_order_WRONG(order):
    # Processes every time, even if duplicate
    db.insert_order(order)  # Duplicate inserts!
    
# With idempotency
def process_order_RIGHT(order):
    event_id = order['eventId']
    if db.event_already_processed(event_id):
        logger.info(f"Skipping duplicate {event_id}")
        return  # Don't reprocess
    
    db.insert_order_and_mark_processed(order, event_id)
```

**6. Check for consumer crashes**
```bash
# Monitor CloudWatch Logs for:
# - Exceptions during processing
# - Container/Lambda OOM errors
# - Timeout errors

# Check consumer health
aws ecs describe-services --cluster <cluster> --services <service>
```

**7. Use message tracing**
```python
# Add message attribute for tracking
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps(message),
    MessageAttributes={
        'trace_id': {
            'DataType': 'String',
            'StringValue': str(uuid.uuid4())
        }
    }
)

# Log on receive
trace_id = message['MessageAttributes']['trace_id']['StringValue']
logger.info(f"Processing message trace_id={trace_id}, receiveCount={receive_count}")
```

**Common causes ranked by frequency:**
1. ✅ Visibility timeout too short (most common)
2. ✅ Consumer crashes before delete
3. ✅ Delete call fails/times out
4. ✅ Missing idempotency check
5. ✅ Lambda batch failure handling incorrect

---

**Q9: What's the difference between SQS delay queue and visibility timeout?**

| Aspect | Delay Queue | Visibility Timeout |
|--------|-------------|-------------------|
| **Purpose** | Postpone initial delivery | Hide message during processing |
| **When set** | At send time (per message) or queue default | At receive time or queue default |
| **Duration** | 0-15 minutes | 0-12 hours |
| **Applies to** | New messages only | Already-received messages |
| **Can extend** | No (fixed at send) | Yes (ChangeMessageVisibility) |
| **Use case** | Scheduled work, rate limiting | Processing protection, retry delay |

**Delay Queue example:**
```python
# Scenario: Rate limit API calls (one every 10 seconds)
for request in api_requests:
    sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(request),
        DelaySeconds=10  # Won't be visible for 10 seconds
    )
    # Continues immediately; doesn't block
```

**Visibility Timeout example:**
```python
# Scenario: Protect in-flight message from duplicate processing
response = sqs.receive_message(
    QueueUrl=queue_url,
    VisibilityTimeout=300  # Message invisible to others for 5 minutes
)

# Process for 4 minutes
process_message(response['Messages'][0])

# Delete before timeout expires
sqs.delete_message(...)
```

**Timeline comparison:**
```
DELAY QUEUE:
t=0s  : SendMessage(DelaySeconds=60)
t=0-60s : Message NOT VISIBLE (cannot be received)
t=60s : Message becomes AVAILABLE
t=60s+: Consumer can receive message

VISIBILITY TIMEOUT:
t=0s  : Message AVAILABLE
t=0s  : Consumer receives message (VisibilityTimeout=300)
t=0-300s: Message INVISIBLE to other consumers
t=300s: If not deleted, becomes VISIBLE again (retry)
```

**Can you combine them?** Yes:
```python
# Send with delay, then visibility timeout applies on receive
sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps(message),
    DelaySeconds=300  # Wait 5 minutes before available
)

# When eventually received (after 5-min delay)
response = sqs.receive_message(
    QueueUrl=queue_url,
    VisibilityTimeout=600  # Then invisible for 10 minutes during processing
)
```

---

**Q10: How do you implement cross-account SQS access?**

**Requirement:** Account A (producer 111111111111) needs to send to queue in Account B (consumer 222222222222).

**Both policies required:**

**Step 1: Account B - Add queue policy (resource-based)**
```bash
# In Account B
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/222222222222/shared-queue \
  --attributes '{
    "Policy": "{
      \"Version\": \"2012-10-17\",
      \"Statement\": [{
        \"Effect\": \"Allow\",
        \"Principal\": {\"AWS\": \"arn:aws:iam::111111111111:root\"},
        \"Action\": [\"sqs:SendMessage\", \"sqs:GetQueueUrl\"],
        \"Resource\": \"arn:aws:sqs:us-east-1:222222222222:shared-queue\"
      }]
    }"
  }'
```

**Step 2: Account A - Add IAM policy (identity-based)**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "sqs:SendMessage",
      "sqs:GetQueueUrl"
    ],
    "Resource": "arn:aws:sqs:us-east-1:222222222222:shared-queue"
  }]
}
```

**Step 3: Send from Account A**
```python
# In Account A application
import boto3

sqs = boto3.client('sqs', region_name='us-east-1')

# Use full ARN or URL (cross-account requires explicit queue URL)
cross_account_queue_url = 'https://sqs.us-east-1.amazonaws.com/222222222222/shared-queue'

response = sqs.send_message(
    QueueUrl=cross_account_queue_url,
    MessageBody=json.dumps({'event': 'OrderCreated', 'orderId': '12345'})
)
```

**Common mistakes:**
- ❌ Only setting queue policy (IAM policy also required)
- ❌ Only setting IAM policy (queue policy also required)
- ❌ Principal: `"AWS": "111111111111"` (needs full ARN or `:root`)
- ❌ Wrong queue URL (use full URL with account ID, not queue name)

**Encryption consideration:**
If queue uses KMS encryption, Account A also needs:
```json
{
  "Effect": "Allow",
  "Action": [
    "kms:Decrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "arn:aws:kms:us-east-1:222222222222:key/<key-id>"
}
```

And KMS key policy in Account B must allow Account A.

---

**Q12: When would you choose SQS Standard vs FIFO vs SNS vs EventBridge?**

**Decision matrix:**

| Requirement | Choose | Why |
|-------------|--------|-----|
| **Task queue, order doesn't matter** | SQS Standard | Simple, high throughput, at-least-once |
| **Strict ordering per entity** | SQS FIFO | Guarantees order within MessageGroupId |
| **Fan-out to multiple services** | SNS → SQS queues | Each subscriber gets copy independently |
| **Event routing based on content** | EventBridge | Rules filter events to targets |
| **Replay historical events** | EventBridge Archive or Kinesis | SQS deletes on consume |
| **Real-time stream processing** | Kinesis Data Streams | Ordered, replay able, multiple consumers |
| **Exactly-once processing** | None (all need idempotent consumers) | Or use database transactions |

**Example scenarios:**

**Scenario 1: Decouple order processing**
```
Web API → SQS Standard → Worker Fleet
✓ Use Standard: order doesn't matter, high throughput
```

**Scenario 2: Process account transactions in order**
```
Transaction Service → SQS FIFO (groupId=accountId) → Processing Service
✓ Use FIFO: strict ordering per account required
```

**Scenario 3: Notify 5 services when order created**
```
Order Service → SNS Topic → [Email Queue, Analytics Queue, Shipping Queue, Inventory Queue, Audit Queue]
✓ Use SNS: fan-out, each service processes independently
```

**Scenario 4: Route events to different services based on region**
```
Producer → EventBridge → Rule(region=US) → US Processing
                      → Rule(region=EU) → EU Processing
✓ Use EventBridge: content-based routing
```

**Scenario 5: Process IoT sensor data in order with replay**
```
IoT Devices → Kinesis Data Streams → [Real-time Lambda, S3 Archive, Analytics]
✓ Use Kinesis: ordered, real-time, replay, multiple consumers
```

**Anti-patterns:**
- ❌ SQS for fan-out (use SNS → SQS)
- ❌ SNS for task queues (push model, no retry control)
- ❌ SQS for event replay (messages deleted on consume)
- ❌ EventBridge for high-throughput task queues (use SQS)
- ❌ FIFO when order doesn't matter (unnecessary cost/complexity)

---

## 15. Message formats and Python and Java examples

### Setup and payload contract

Use Python 3 with Boto3, or Java 17+ with AWS SDK 2.x module `software.amazon.awssdk:sqs` and Jackson 2.x `com.fasterxml.jackson.core:jackson-databind`. The optional Java Lambda handler also needs `com.amazonaws:aws-lambda-java-core`. Pin versions through your build, using an approved AWS SDK 2.x BOM for SDK modules.

Set `AWS_REGION` and `SQS_QUEUE_URL` and use a profile/workload role through the default credential chain. The publisher needs `sqs:SendMessage`; a polling worker needs `sqs:ReceiveMessage` and `sqs:DeleteMessage`, plus applicable KMS access. These examples assume an existing **standard queue**. They send real messages and the polling worker deletes successfully processed messages when run.

The same order event used in [SNS](aws-sns-cheatsheet.md#12-message-formats-and-python-and-java-examples) is serialized into `MessageBody`, a string. SQS can transport text that is not JSON; JSON here is the application's chosen contract.

**SendMessage request shape** (SDK configuration/authentication omitted):

```json
{
  "QueueUrl": "https://sqs.ap-south-1.amazonaws.com/123456789012/orders",
  "MessageBody": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}",
  "MessageAttributes": {
    "eventType": {"DataType": "String", "StringValue": "OrderCreated"}
  }
}
```

**Reduced ReceiveMessage response** for a direct send:

```json
{
  "Messages": [
    {
      "MessageId": "sqs-generated-id",
      "ReceiptHandle": "opaque-handle-for-this-receive",
      "Body": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}",
      "Attributes": {"ApproximateReceiveCount": "1"},
      "MessageAttributes": {
        "eventType": {"DataType": "String", "StringValue": "OrderCreated"}
      }
    }
  ]
}
```

Request message attributes explicitly when receiving. `Attributes` contains SQS system metadata; `MessageAttributes` contains publisher metadata. `MessageId`, `ReceiptHandle`, and the application's `eventId` have different jobs. Only the current receipt handle acknowledges this delivery. An empty poll may omit `Messages`. [SendMessage](https://docs.aws.amazon.com/boto3/latest/reference/services/sqs/client/send_message.html), [ReceiveMessage](https://docs.aws.amazon.com/boto3/latest/reference/services/sqs/client/receive_message.html)

### Body parsing depends on the producer

| Configured `BODY_MODE` below | Producer/target setup | Extract business event |
|---|---|---|
| `direct` | Direct SQS send or SNS raw delivery | Parse `Body` once |
| `sns-envelope` | SNS subscription with raw delivery disabled | Parse `Body`, then its `Message` string |
| `eventbridge` | Classic EventBridge target with the original whole event | Parse `Body`, then take the already-decoded `detail` object |

Configure the known contract; do not guess from the presence of a generic `Message` or `detail` key. EventBridge target transformations can deliberately change this format.

### Python: send, long-poll, parse, and delete

Save as `sqs_example.py`. Run with argument `send` to publish, or `receive` for one long-poll cycle. Set `BODY_MODE` for the queue's contract; it defaults to `direct`.

```python
import json
import os
import sys
import boto3


def decode_body(body, mode):
    value = json.loads(body)
    if mode == "sns-envelope":
        value = json.loads(value["Message"])
    elif mode == "eventbridge":
        value = value["detail"]
    elif mode != "direct":
        raise ValueError("Unknown BODY_MODE")
    return value


def process_order(order):
    if order["schemaVersion"] != 1 or order["eventType"] != "OrderCreated":
        raise ValueError("Unsupported order event")
    # Demo only: success means printing. Replace with durable, idempotent work.
    print("Processed order:", order["orderId"], "event:", order["eventId"])


def send(sqs, queue_url):
    order = {
        "schemaVersion": 1, "eventId": "evt-order-456-created",
        "eventType": "OrderCreated", "orderId": "456",
        "country": "IN", "total": 1200,
    }
    result = sqs.send_message(
        QueueUrl=queue_url,
        MessageBody=json.dumps(order),
        MessageAttributes={
            "eventType": {"DataType": "String", "StringValue": "OrderCreated"}
        },
    )
    print("SQS accepted:", result["MessageId"])


def receive_once(sqs, queue_url, mode):
    result = sqs.receive_message(
        QueueUrl=queue_url,
        MaxNumberOfMessages=10,
        WaitTimeSeconds=20,
        VisibilityTimeout=60,
        MessageAttributeNames=["All"],
        MessageSystemAttributeNames=["ApproximateReceiveCount"],
    )
    for message in result.get("Messages", []):
        try:
            process_order(decode_body(message["Body"], mode))
            sqs.delete_message(
                QueueUrl=queue_url, ReceiptHandle=message["ReceiptHandle"]
            )
        except Exception as error:
            # No delete on failure; visibility expiry permits another attempt.
            print("Not acknowledged:", message["MessageId"], error, file=sys.stderr)


if __name__ == "__main__":
    if len(sys.argv) != 2 or sys.argv[1] not in {"send", "receive"}:
        raise SystemExit("Usage: python sqs_example.py send|receive")
    client = boto3.client("sqs", region_name=os.environ["AWS_REGION"])
    url = os.environ["SQS_QUEUE_URL"]
    if sys.argv[1] == "send":
        send(client, url)
    else:
        receive_once(client, url, os.environ.get("BODY_MODE", "direct"))
```

The 60-second visibility value is a demo setting. Size it for the whole batch's processing time or extend visibility while working. A production worker needs repeated polling, shutdown handling, schema validation, metrics, bounded error handling, and durable idempotency. SDK request retries do not make the application side effect atomic with deletion.

### Java: the same send and receive flow

Save as `SqsExample.java`; use argument `send` or `receive`. Each SDK client is closed after this short demonstration; a long-lived worker normally reuses its client.

```java
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sqs.SqsClient;
import software.amazon.awssdk.services.sqs.model.*;
import java.util.Map;

public class SqsExample {
    private static final ObjectMapper JSON = new ObjectMapper();

    static JsonNode decodeBody(String body, String mode) throws Exception {
        JsonNode value = JSON.readTree(body);
        return switch (mode) {
            case "direct" -> value;
            case "sns-envelope" -> JSON.readTree(value.required("Message").asText());
            case "eventbridge" -> value.required("detail");
            default -> throw new IllegalArgumentException("Unknown BODY_MODE");
        };
    }

    static void processOrder(JsonNode order) {
        if (order.required("schemaVersion").asInt() != 1
            || !"OrderCreated".equals(order.required("eventType").asText())) {
            throw new IllegalArgumentException("Unsupported order event");
        }
        // Demo only: replace with durable, idempotent business processing.
        System.out.println("Processed order: " + order.required("orderId").asText()
            + ", event: " + order.required("eventId").asText());
    }

    public static void main(String[] args) throws Exception {
        if (args.length != 1 || !(args[0].equals("send") || args[0].equals("receive"))) {
            throw new IllegalArgumentException("Usage: SqsExample send|receive");
        }
        String queueUrl = System.getenv("SQS_QUEUE_URL");
        String mode = System.getenv().getOrDefault("BODY_MODE", "direct");
        try (SqsClient sqs = SqsClient.builder()
                .region(Region.of(System.getenv("AWS_REGION"))).build()) {
            if (args[0].equals("send")) {
                String body = JSON.writeValueAsString(Map.of(
                    "schemaVersion", 1, "eventId", "evt-order-456-created",
                    "eventType", "OrderCreated", "orderId", "456",
                    "country", "IN", "total", 1200));
                var result = sqs.sendMessage(SendMessageRequest.builder()
                    .queueUrl(queueUrl).messageBody(body)
                    .messageAttributes(Map.of("eventType",
                        MessageAttributeValue.builder().dataType("String")
                            .stringValue("OrderCreated").build()))
                    .build());
                System.out.println("SQS accepted: " + result.messageId());
            } else {
                var result = sqs.receiveMessage(ReceiveMessageRequest.builder()
                    .queueUrl(queueUrl).maxNumberOfMessages(10)
                    .waitTimeSeconds(20).visibilityTimeout(60)
                    .messageAttributeNames("All")
                    .messageSystemAttributeNamesWithStrings("ApproximateReceiveCount")
                    .build());
                for (Message message : result.messages()) {
                    try {
                        processOrder(decodeBody(message.body(), mode));
                        sqs.deleteMessage(DeleteMessageRequest.builder()
                            .queueUrl(queueUrl).receiptHandle(message.receiptHandle())
                            .build());
                    } catch (Exception e) {
                        System.err.println("Not acknowledged: " + message.messageId()
                            + " " + e.getMessage());
                    }
                }
            }
        }
    }
}
```

[AWS Java send/receive/delete API examples](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/examples-sqs-messages.html)

### Receiving through Lambda is a different envelope

An SQS event source mapping sends **`Records` with lowercase `body` and `messageId`**, rather than the SDK's `Messages`/`Body`/`MessageId`. Below is a reduced Lambda input:

```json
{
  "Records": [
    {
      "messageId": "sqs-generated-id",
      "receiptHandle": "opaque-handle-for-this-receive",
      "eventSource": "aws:sqs",
      "body": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}",
      "attributes": {"ApproximateReceiveCount": "1"},
      "messageAttributes": {
        "eventType": {"dataType": "String", "stringValue": "OrderCreated"}
      }
    }
  ]
}
```

Python handler: place this beside `decode_body` and `process_order` from the example above. Enable **`ReportBatchItemFailures`** on the mapping; the handler alone does not enable it.

```python
def lambda_handler(event, context):
    failures = []
    for record in event["Records"]:
        try:
            process_order(decode_body(record["body"], os.environ.get("BODY_MODE", "direct")))
        except Exception:
            failures.append({"itemIdentifier": record["messageId"]})
    return {"batchItemFailures": failures}
```

Java handler: package `SqsExample` and this `SqsLambdaHandler.java` together; configure `SqsLambdaHandler::handleRequest`. The helper methods are reused without running `main`.

```java
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

public class SqsLambdaHandler
        implements RequestHandler<Map<String, Object>, Map<String, Object>> {
    private static final ObjectMapper JSON = new ObjectMapper();

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> event, Context context) {
        JsonNode root = JSON.valueToTree(event);
        List<Map<String, String>> failures = new ArrayList<>();
        String mode = System.getenv().getOrDefault("BODY_MODE", "direct");
        for (JsonNode record : root.required("Records")) {
            try {
                SqsExample.processOrder(SqsExample.decodeBody(
                    record.required("body").asText(), mode));
            } catch (Exception e) {
                failures.add(Map.of("itemIdentifier", record.required("messageId").asText()));
            }
        }
        return Map.of("batchItemFailures", failures);
    }
}
```

Example result if one record fails:

```json
{"batchItemFailures": [{"itemIdentifier": "sqs-generated-id"}]}
```

The mapping handles deletion of successful records; these Lambda handlers do not call `DeleteMessage` themselves. The snippets continue after individual failures and are for **standard queues**. For FIFO, stop after the first failure and report failed/unprocessed messages to preserve ordering. [SQS Lambda input and batching](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)

### FIFO send changes and interview checks

For a FIFO queue URL, add `MessageGroupId=orderId` and `MessageDeduplicationId=eventId` in Python; Java uses `.messageGroupId(orderId)` and `.messageDeduplicationId(eventId)`. Do not generate a new deduplication ID merely because a send is retried.

- **Why parse JSON?** The transport body is text; the application chooses its schema.
- **Why not delete by message ID?** Deletion acknowledges a receipt, using its current receipt handle.
- **Why request attributes?** They are optional receive output and distinct from body fields.
- **Why preserve `eventId`?** Broker IDs can change across hops; business idempotency needs a stable logical identity.
- **Why no delete on a processing exception?** The message must remain available for retry/redrive. Only acknowledge after the actual operation succeeds durably.

---

## 16. AWS sources

- [Message quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html)
- [Queue parameters](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-configure-queue-parameters.html)
- [FIFO terms](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-key-terms.html)
- [FIFO deduplication semantics](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html)
- [Dead-letter queues](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html)
- [Lambda integration](https://docs.aws.amazon.com/lambda/latest/dg/services-sqs-configure.html)
- [Boto3 SendMessage](https://docs.aws.amazon.com/boto3/latest/reference/services/sqs/client/send_message.html)
- [Boto3 ReceiveMessage](https://docs.aws.amazon.com/boto3/latest/reference/services/sqs/client/receive_message.html)
- [Java send, receive, and delete](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/examples-sqs-messages.html)
- [SQS Lambda event shape](https://docs.aws.amazon.com/lambda/latest/dg/with-sqs.html)

**Related:** [SNS](aws-sns-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md), [EventBridge](aws-eventbridge-cheatsheet.md).
