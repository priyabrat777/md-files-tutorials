# Amazon SNS Cheat Sheet

**Interview preparation: publish/subscribe, fanout, filtering, FIFO, delivery failures, and payload limits.**

**Verified:** 2026-10-13. Topic type, configured size, and subscription protocol affect behavior.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Standard versus FIFO](#2-standard-versus-fifo)
- [3. Current correction: message size](#3-current-correction-message-size)
- [4. Subscription filters and message shape](#4-subscription-filters-and-message-shape)
- [5. Delivery, retries, and dead-letter queues](#5-delivery-retries-and-dead-letter-queues)
- [6. FIFO, deduplication, and idempotency](#6-fifo-deduplication-and-idempotency)
- [7. Security and cross-account fanout](#7-security-and-cross-account-fanout)
- [8. Observability and cost](#8-observability-and-cost)
- [9. Common patterns](#9-common-patterns)
- [10. Troubleshooting and practical inspection](#10-troubleshooting-and-practical-inspection)
- [11. Quick reference card](#11-quick-reference-card)
- [12. Exam tips summary](#12-exam-tips-summary)
- [13. Interview questions and detailed answers](#13-interview-questions-and-detailed-answers)
- [14. Message formats and Python and Java examples](#14-message-formats-and-python-and-java-examples)
- [15. AWS sources](#15-aws-sources)

---

## 1. Fundamentals and mental model

SNS distributes a published message to matching subscriptions. It is useful when several independent consumers need their own copy of an event.

### Mental Model: Pub/Sub vs Queue

```text
SNS = Publisher/Subscriber (Fanout)
├── One message → Multiple independent copies
├── Each subscriber gets its own copy
├── Asynchronous push delivery
└── No message persistence in SNS (delivered immediately)

SQS = Queue (Competing Consumers)
├── One message → One consumer
├── Multiple workers compete for messages
├── Pull-based polling
└── Message persistence until processed
```

### Core Architecture: Fanout Pattern

```text
                                    ┌─────────────────────────────────┐
                                    │     Publisher Application       │
                                    │  (Order Service, Event Source)  │
                                    └────────────┬────────────────────┘
                                                 │
                                                 │ Publish (256KB-1MB)
                                                 ▼
                                    ┌────────────────────────────────┐
                                    │        SNS Topic               │
                                    │  (orders-topic)                │
                                    │  • Message filtering           │
                                    │  • Delivery retries            │
                                    │  • Encryption (KMS)            │
                                    └────┬──────┬──────┬─────────────┘
                                         │      │      │
                      ┌──────────────────┘      │      └─────────────────┐
                      │                         │                        │
                      │ Filter:                 │ Filter:                │ Filter:
                      │ eventType=              │ eventType=             │ (none - all
                      │ "OrderCreated"          │ "OrderCreated"         │  messages)
                      ▼                         ▼                        ▼
         ┌──────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐
         │  SQS: billing-queue  │  │ SQS: shipping-queue │  │ SQS: analytics-queue │
         │  (Durable Buffer)    │  │  (Durable Buffer)   │  │  (Durable Buffer)    │
         └──────────┬───────────┘  └──────────┬──────────┘  └──────────┬───────────┘
                    │                         │                         │
                    │ Poll                    │ Poll                    │ Poll
                    ▼                         ▼                         ▼
         ┌──────────────────────┐  ┌─────────────────────┐  ┌──────────────────────┐
         │   Billing Workers    │  │  Shipping Workers   │  │  Analytics Workers   │
         │   (3 instances)      │  │   (5 instances)     │  │   (2 instances)      │
         │   • Invoice          │  │   • Ship order      │  │   • Store metrics    │
         │   • Charge card      │  │   • Track package   │  │   • Update dashboard │
         └──────────────────────┘  └─────────────────────┘  └──────────────────────┘

Key Points:
├── SNS pushes to SQS (no polling at SNS level)
├── Each SQS queue gets independent copy
├── Workers poll their queue (competing consumers within team)
├── One subscriber failure doesn't affect others
└── Filters reduce unnecessary queue traffic
```

### Why This Pattern?

| Scenario | Solution | Benefit |
|----------|----------|---------|
| **Multiple teams need same event** | SNS fanout to team-specific SQS queues | Independent processing, no coupling |
| **Different processing rates** | Each queue has own workers/scaling | Billing slow doesn't block shipping |
| **Isolated failures** | Queue + DLQ per team | One team's bug doesn't break others |
| **Selective routing** | Subscription filters | Analytics gets all, billing gets only paid orders |

### One Queue vs Fanout

```text
❌ Wrong: Single SQS Queue with Multiple Consumers
OrderService → SQS queue → [Billing, Shipping, Analytics workers all polling]
Problem: First consumer deletes message; others never see it

✅ Right: SNS Fanout to Multiple Queues
OrderService → SNS → [Billing SQS, Shipping SQS, Analytics SQS]
Result: Each team gets independent copy and buffer
```

One queue with many workers is **competing consumption** (load distribution). One topic with several queues is **fanout** plus **independent buffering**.

## 2. Standard versus FIFO

| Dimension | Standard topic | FIFO topic |
|---|---|---|
| Delivery/order model | At-least-once, best-effort ordering | Ordering and deduplication under documented conditions |
| Typical endpoints | SQS, Lambda, HTTPS, email, SMS and others subject to configuration | SQS FIFO or standard queues |
| Ordering scope | No strict ordering | Message group |
| Lambda integration | Direct asynchronous invocation | Through a subscribed SQS queue |
| Best fit | General fanout and notifications | Ordered business event streams |

A FIFO topic feeding an SQS **standard** queue does not make the downstream queue ordered. FIFO topics cannot deliver directly to customer endpoints such as email/SMS/HTTPS. [FIFO delivery](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-delivery.html)

## 3. Current correction: message size

**256 KiB is the default, not the only supported configuration.** SNS now supports up to **1 MiB (1,048,576 bytes)** when the topic's `MaximumMessageSize` is raised.

| Configuration | Constraints |
|---|---|
| Default 256 KiB or below | Broader endpoint types subject to topic-type rules |
| Above 256 KiB, up to 1 MiB | Supported endpoint types: SQS, Lambda, Data Firehose; at most 100 subscriptions |

The topic validates the body plus attributes. Downstream limits and envelope overhead still matter, especially Firehose's smaller record limit. Larger-than-supported data can live in S3 with a reference in the message. These endpoint rules do not override FIFO topic restrictions. [Large-message configuration](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html)

Older quota pages can still show only 256 KiB. For implementation, consult the feature guide and the live `MaximumMessageSize` attribute instead of quoting a single number without context.

## 4. Subscription filters and message shape

Each subscription can filter on message attributes or the JSON message body via `FilterPolicyScope`. No filter means all messages are eligible. Filtering reduces unnecessary deliveries and consumer-side branching. [Message filtering](https://docs.aws.amazon.com/sns/latest/dg/sns-message-filtering.html)

Example filter policy for `FilterPolicyScope=MessageBody`:

```json
{
  "eventType": ["OrderCreated"],
  "region": ["IN", "SG"]
}
```

The publisher's body must have matching fields/types. Filtering is not authorization: a subscriber allowed to change its filter might request other events. Use account/topic boundaries and IAM for confidentiality.

Raw message delivery changes how SNS wraps a message for supported subscriptions. Verify the consumer's expected envelope and attribute limitations rather than toggling it without a contract test.

## 5. Delivery, retries, and dead-letter queues

SNS retry behavior depends on endpoint type and error category. Some failures are non-retryable; transient failures follow the delivery policy. A successful publish does not mean every subscriber has processed the message.

**SNS DLQ attaches to the subscription.** It captures messages SNS could not deliver. It is different from an SQS consumer DLQ that captures messages repeatedly failing application processing. [SNS dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html)

```text
SNS → delivery fails → subscription DLQ
SNS → SQS succeeds → worker fails repeatedly → queue processing DLQ
SNS → Lambda accepted → handler fails → Lambda async failure handling
```

For direct Lambda subscriptions, inspect both SNS delivery metrics and Lambda asynchronous error/destination metrics. Do not expect SNS to retry a business error after Lambda has accepted the event.

## 6. FIFO, deduplication, and idempotency

Use `MessageGroupId` for the business ordering unit, such as order or account. A single global group serializes unrelated work.

SNS FIFO deduplication uses an explicit ID or content-based configuration, with a five-minute deduplication window. Delivery guarantees have conditions, including consumer behavior and filtering. [FIFO deduplication](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-dedup.html)

**Reasoning example:** a worker charges a card, then crashes before acknowledging its SQS message. A later attempt must recognize the existing charge. Broker deduplication cannot make a database/payment side effect and queue acknowledgement one atomic operation.

Use a stable business event ID and durable idempotency records; keep the effect and recorded outcome atomic where possible.

## 7. Security and cross-account fanout

- Publisher requires topic access; topic policy controls supported resource access.
- An SQS queue policy must allow `sns.amazonaws.com` to send from the intended topic ARN, usually constrained with `aws:SourceArn` and where applicable `aws:SourceAccount`.
- KMS permissions are separate from topic/queue permissions when customer managed keys are involved.
- Subscription ownership and confirmation differ by endpoint/account workflow. A pending subscription does not receive normal deliveries.
- Use TLS, least privilege, and controlled topic administration; avoid broad wildcard publishing.

For encrypted DLQs, verify the SNS service can use the key and send to the queue. A broken DLQ path can hide the very failures it is meant to preserve.

## 8. Observability and cost

Monitor publishes, deliveries, failed notifications, filtered-out messages, DLQ arrivals, and subscriber backlog. Rising filtered counts may be intentional routing or a schema regression.

Cost is driven by published data, deliveries/protocols, payload units, transfer, encryption, and downstream services. Fanout multiplies downstream work; filtering can reduce it.

---

## 9. Common patterns

### Pattern 1: SNS fanout to multiple SQS queues (decoupled microservices)

**Scenario:** Order created event needs to trigger independent actions by different teams.

**Architecture:**
```
Order Service → SNS Topic (orders-topic)
                   ├→ SQS Queue (billing-queue) → Billing Service
                   ├→ SQS Queue (shipping-queue) → Shipping Service
                   ├→ SQS Queue (inventory-queue) → Inventory Service
                   └→ SQS Queue (analytics-queue) → Analytics Service
```

**Setup via CLI:**

```bash
# Create SNS topic
aws sns create-topic --name orders-topic
TOPIC_ARN=$(aws sns create-topic --name orders-topic --query 'TopicArn' --output text)

# Create SQS queues for each service
aws sqs create-queue --queue-name billing-queue
aws sqs create-queue --queue-name shipping-queue
aws sqs create-queue --queue-name inventory-queue
aws sqs create-queue --queue-name analytics-queue

# Get queue ARNs and URLs
BILLING_QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/billing-queue \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# Add queue policy to allow SNS to send messages
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/billing-queue \
  --attributes '{
    "Policy": "{
      \"Version\": \"2012-10-17\",
      \"Statement\": [{
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"sns.amazonaws.com\"},
        \"Action\": \"sqs:SendMessage\",
        \"Resource\": \"'"$BILLING_QUEUE_ARN"'\",
        \"Condition\": {
          \"ArnEquals\": {\"aws:SourceArn\": \"'"$TOPIC_ARN"'\"}
        }
      }]
    }"
  }'

# Subscribe each queue to the topic
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $BILLING_QUEUE_ARN \
  --attributes RawMessageDelivery=false

# Repeat for other queues...
```

**Publish event:**

```python
import boto3
import json

sns = boto3.client('sns')
topic_arn = 'arn:aws:sns:us-east-1:123456789012:orders-topic'

def publish_order_created(order_id, customer_id, total):
    message = {
        'eventId': f'order-{order_id}-created',
        'eventType': 'OrderCreated',
        'orderId': order_id,
        'customerId': customer_id,
        'total': total
    }
    
    response = sns.publish(
        TopicArn=topic_arn,
        Message=json.dumps(message),
        MessageAttributes={
            'eventType': {'DataType': 'String', 'StringValue': 'OrderCreated'}
        }
    )
    print(f"Published to SNS: {response['MessageId']}")
```

**Key benefits:**
- ✅ Each service gets independent copy (billing doesn't block shipping)
- ✅ Independent scaling per service
- ✅ Failure isolation (one service down doesn't affect others)
- ✅ SQS provides buffering and retry for each service

### Pattern 2: SNS with subscription filters (selective routing)

**Scenario:** Analytics needs all orders, but billing only needs paid orders.

```bash
# Subscribe analytics queue without filter (receives all messages)
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $ANALYTICS_QUEUE_ARN

# Subscribe billing queue with filter (only paid orders)
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $BILLING_QUEUE_ARN \
  --attributes '{
    "FilterPolicy": "{\"paymentStatus\":[\"paid\",\"authorized\"]}",
    "FilterPolicyScope": "MessageAttributes"
  }'
```

**Publisher sends with message attributes:**

```python
def publish_order_with_payment_status(order_id, payment_status):
    message = {
        'eventId': f'order-{order_id}-created',
        'eventType': 'OrderCreated',
        'orderId': order_id,
        'total': 150.00
    }
    
    response = sns.publish(
        TopicArn=topic_arn,
        Message=json.dumps(message),
        MessageAttributes={
            'eventType': {'DataType': 'String', 'StringValue': 'OrderCreated'},
            'paymentStatus': {'DataType': 'String', 'StringValue': payment_status}
        }
    )
    return response['MessageId']

# This goes to both analytics and billing
publish_order_with_payment_status('order-123', 'paid')

# This goes only to analytics (billing filter blocks it)
publish_order_with_payment_status('order-456', 'pending')
```

**Filter policy on message body (alternative):**

```bash
# Filter on JSON body fields
aws sns set-subscription-attributes \
  --subscription-arn $SUBSCRIPTION_ARN \
  --attribute-name FilterPolicyScope \
  --attribute-value MessageBody

aws sns set-subscription-attributes \
  --subscription-arn $SUBSCRIPTION_ARN \
  --attribute-name FilterPolicy \
  --attribute-value '{
    "eventType": ["OrderCreated"],
    "total": [{"numeric": [">=", 100]}]
  }'
```

**Result:** Billing queue only receives orders ≥ $100.

### Pattern 3: SNS to Lambda for immediate processing

**Scenario:** Send email notification immediately when order created (no queue buffering needed).

**Setup:**

```bash
# Create Lambda function (assume exists: send-email-lambda)
LAMBDA_ARN="arn:aws:lambda:us-east-1:123456789012:function:send-email-lambda"

# Grant SNS permission to invoke Lambda
aws lambda add-permission \
  --function-name send-email-lambda \
  --statement-id sns-invoke-permission \
  --action lambda:InvokeFunction \
  --principal sns.amazonaws.com \
  --source-arn $TOPIC_ARN

# Subscribe Lambda to SNS topic
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol lambda \
  --notification-endpoint $LAMBDA_ARN
```

**Lambda handler:**

```python
import json
import boto3

ses = boto3.client('ses')

def lambda_handler(event, context):
    """Process SNS notification and send email"""
    for record in event['Records']:
        # Parse SNS message
        message = json.loads(record['Sns']['Message'])
        order_id = message['orderId']
        customer_email = message.get('customerEmail')
        
        if customer_email:
            # Send email via SES
            ses.send_email(
                Source='orders@example.com',
                Destination={'ToAddresses': [customer_email]},
                Message={
                    'Subject': {'Data': f'Order {order_id} Confirmed'},
                    'Body': {'Text': {'Data': f'Your order {order_id} has been confirmed.'}}
                }
            )
            print(f"Email sent for order {order_id}")
    
    return {'statusCode': 200}
```

**Key points:**
- ✅ Lambda invoked asynchronously by SNS
- ✅ No SQS queue needed for immediate processing
- ✅ Configure Lambda DLQ for failed invocations
- ⚠️ Lambda errors don't cause SNS retry (use Lambda destinations)

### Pattern 4: Cross-account SNS fanout

**Scenario:** Account A (central events) publishes to topic; Account B (data team) subscribes with their own SQS queue.

**Account A (111111111111) - Topic policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAccountAPublish",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
      "Action": ["sns:Publish", "sns:Subscribe"],
      "Resource": "arn:aws:sns:us-east-1:111111111111:shared-events"
    },
    {
      "Sid": "AllowAccountBSubscribe",
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::222222222222:root"},
      "Action": "sns:Subscribe",
      "Resource": "arn:aws:sns:us-east-1:111111111111:shared-events"
    }
  ]
}
```

**Account B (222222222222) - Queue policy:**

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "sns.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:us-east-1:222222222222:data-team-queue",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:us-east-1:111111111111:shared-events"
      }
    }
  }]
}
```

**Account B - Subscribe to Account A topic:**

```bash
# In Account B
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:111111111111:shared-events \
  --protocol sqs \
  --notification-endpoint arn:aws:sqs:us-east-1:222222222222:data-team-queue \
  --region us-east-1
```

**Both policies required:** Topic policy (Account A) + Queue policy (Account B).

### Pattern 5: SNS FIFO topic with SQS FIFO queue (strict ordering)

**Scenario:** Process account transactions in order per account.

```bash
# Create FIFO topic (must end in .fifo)
aws sns create-topic \
  --name transactions.fifo \
  --attributes FifoTopic=true,ContentBasedDeduplication=false

FIFO_TOPIC_ARN=$(aws sns create-topic --name transactions.fifo \
  --attributes FifoTopic=true --query 'TopicArn' --output text)

# Create FIFO queue
aws sqs create-queue \
  --queue-name transactions.fifo \
  --attributes FifoQueue=true,ContentBasedDeduplication=false

FIFO_QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/transactions.fifo \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# Add queue policy
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/transactions.fifo \
  --attributes '{
    "Policy": "{...}"  # Same as Pattern 1
  }'

# Subscribe FIFO queue to FIFO topic
aws sns subscribe \
  --topic-arn $FIFO_TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $FIFO_QUEUE_ARN
```

**Publish with MessageGroupId:**

```python
def publish_transaction(account_id, transaction_type, amount):
    """Publish transaction with ordering per account"""
    event_id = f'txn-{account_id}-{transaction_type}-{int(time.time())}'
    
    message = {
        'eventId': event_id,
        'eventType': transaction_type,
        'accountId': account_id,
        'amount': amount
    }
    
    response = sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:transactions.fifo',
        Message=json.dumps(message),
        MessageGroupId=account_id,  # Group by account for ordering
        MessageDeduplicationId=event_id  # Prevent duplicates
    )
    return response['MessageId']

# These are processed in order for account-123
publish_transaction('account-123', 'Deposit', 100.00)
publish_transaction('account-123', 'Withdrawal', 50.00)
publish_transaction('account-123', 'Deposit', 25.00)

# These are processed in parallel (different account)
publish_transaction('account-456', 'Deposit', 200.00)
```

**Guarantees:**
- ✅ Transactions for same account processed in order
- ✅ Different accounts processed in parallel
- ✅ Duplicate sends (same MessageDeduplicationId) rejected within 5 minutes
- ⚠️ Still need idempotent processing for retries

### Pattern 6: SNS with DLQ for failed deliveries

**Scenario:** Capture messages that SNS fails to deliver to a subscription.

```bash
# Create DLQ for failed SNS deliveries
aws sqs create-queue --queue-name sns-delivery-failures-dlq

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/sns-delivery-failures-dlq \
  --attribute-names QueueArn --query 'Attributes.QueueArn' --output text)

# Configure subscription with DLQ
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol sqs \
  --notification-endpoint $TARGET_QUEUE_ARN \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"'"$DLQ_ARN"'\"}"
  }'

# Grant SNS permission to send to DLQ
aws sqs set-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/sns-delivery-failures-dlq \
  --attributes '{
    "Policy": "{
      \"Version\": \"2012-10-17\",
      \"Statement\": [{
        \"Effect\": \"Allow\",
        \"Principal\": {\"Service\": \"sns.amazonaws.com\"},
        \"Action\": \"sqs:SendMessage\",
        \"Resource\": \"'"$DLQ_ARN"'\",
        \"Condition\": {
          \"ArnEquals\": {\"aws:SourceArn\": \"'"$TOPIC_ARN"'\"}
        }
      }]
    }"
  }'

# Create CloudWatch alarm for DLQ
aws cloudwatch put-metric-alarm \
  --alarm-name sns-delivery-failures \
  --alarm-description "Alert when SNS delivery DLQ has messages" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --statistic Average \
  --period 60 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --dimensions Name=QueueName,Value=sns-delivery-failures-dlq \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

**When messages go to SNS DLQ:**
- Target queue doesn't exist
- Queue policy denies SNS
- KMS key policy denies SNS (encrypted queue)
- Target endpoint unavailable (Lambda, HTTPS)
- Message exceeds target queue size limit

**Note:** SNS DLQ captures SNS delivery failures, not application processing failures.

### Pattern 7: Mobile push notifications via SNS

**Scenario:** Send push notifications to iOS/Android devices.

```bash
# Create platform application (iOS example with APNs)
aws sns create-platform-application \
  --name MyiOSApp \
  --platform APNS \
  --attributes \
    PlatformCredential=<APNs-certificate>,\
    PlatformPrincipal=<APNs-private-key>

PLATFORM_APP_ARN=$(aws sns create-platform-application --name MyiOSApp \
  --platform APNS --attributes PlatformCredential=<cert> \
  --query 'PlatformApplicationArn' --output text)

# Register device token
aws sns create-platform-endpoint \
  --platform-application-arn $PLATFORM_APP_ARN \
  --token <device-token-from-app>

ENDPOINT_ARN=$(aws sns create-platform-endpoint \
  --platform-application-arn $PLATFORM_APP_ARN \
  --token <device-token> --query 'EndpointArn' --output text)
```

**Send push notification:**

```python
import boto3
import json

sns = boto3.client('sns')

def send_push_notification(endpoint_arn, title, body):
    """Send push notification to specific device"""
    # iOS-specific message format
    ios_payload = {
        'aps': {
            'alert': {
                'title': title,
                'body': body
            },
            'sound': 'default',
            'badge': 1
        }
    }
    
    # Android-specific message format
    android_payload = {
        'notification': {
            'title': title,
            'body': body
        }
    }
    
    # Combined message for multiple platforms
    message = {
        'default': body,  # Fallback
        'APNS': json.dumps(ios_payload),
        'GCM': json.dumps(android_payload)
    }
    
    response = sns.publish(
        TargetArn=endpoint_arn,
        Message=json.dumps(message),
        MessageStructure='json'  # Enable platform-specific messages
    )
    print(f"Push notification sent: {response['MessageId']}")

# Usage
send_push_notification(
    endpoint_arn='arn:aws:sns:us-east-1:123456789012:endpoint/APNS/MyiOSApp/abc-123',
    title='Order Shipped',
    body='Your order #12345 has been shipped!'
)
```

**Broadcast to all devices (topic-based):**

```bash
# Create application topic
aws sns create-topic --name order-notifications

# Subscribe all iOS devices to topic
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:123456789012:order-notifications \
  --protocol application \
  --notification-endpoint $ENDPOINT_ARN
```

---

## 10. Troubleshooting and practical inspection

### Common Issues and Root Causes

| Symptom | Root Cause | Solution |
|---|---|---|
| **Publish accepted but no consumer event** | • Subscription not confirmed<br>• Filter policy blocks message<br>• Wrong `FilterPolicyScope` (body vs attributes) | • Check subscription status<br>• Validate filter matches message<br>• Test without filter first |
| **SQS not receiving** | • Queue policy missing `sns.amazonaws.com` principal<br>• Missing/wrong `aws:SourceArn` condition<br>• KMS key policy denies SNS | • Add queue policy allowing topic ARN<br>• Verify condition matches topic<br>• Grant SNS decrypt/GenerateDataKey |
| **Lambda errors but SNS delivery succeeds** | • Async invocation vs handler execution<br>• SNS only tracks push acceptance<br>• Lambda failures go to Lambda DLQ, not SNS DLQ | • Check Lambda CloudWatch Logs<br>• Configure Lambda DLQ/destinations<br>• Monitor Lambda error metrics |
| **Duplicates in standard topic** | • At-least-once delivery guarantee<br>• Publisher retry on timeout<br>• Network issues cause resend | • Design idempotent consumers<br>• Use stable event IDs<br>• Consider FIFO for strict dedup |
| **Order lost** | • Standard topic/queue (no ordering)<br>• Different message groups (FIFO)<br>• Parallel workers processing same group | • Use FIFO topic + FIFO queue<br>• Ensure correct `MessageGroupId`<br>• Process groups serially |
| **Large publish rejected** | • Default 256KB limit active<br>• Attributes + body exceed limit<br>• Endpoint incompatible with >256KB | • Set `MaximumMessageSize` to 1MB<br>• Use S3 for large payloads<br>• Verify all subscriptions support size |
| **Messages disappear (no error)** | • Subscription filter silently drops<br>• Subscription pending confirmation<br>• DLQ misconfigured (permissions) | • Check `NumberOfMessagesPublished` vs `NumberOfNotificationsDelivered`<br>• Confirm subscription<br>• Test DLQ permissions |
| **Cross-account delivery fails** | • Queue policy in target account missing<br>• Both accounts need KMS permissions<br>• `aws:SourceAccount` condition too strict | • Add queue policy in target account<br>• Grant SNS in source, queue in target KMS access<br>• Use `aws:SourceArn` not `aws:SourceAccount` |
| **FIFO topic rejects message** | • Missing `MessageGroupId`<br>• Missing `MessageDeduplicationId` (if content-based disabled)<br>• Trying to subscribe email/SMS | • Add both required attributes<br>• Use business IDs<br>• FIFO supports only SQS endpoints |
| **High filtered message count** | • Filter too restrictive<br>• Schema mismatch (type/field name)<br>• Publisher sending unexpected format | • Review filter policy vs actual messages<br>• Test with sample payload<br>• Add CloudWatch alarm on filtered metrics |

### Delivery Failure Root Causes by Protocol

| Protocol | Common Failure | Root Cause | Fix |
|----------|----------------|------------|-----|
| **SQS** | Permission denied | Queue policy missing SNS principal | Add queue policy with `aws:SourceArn` condition |
| **Lambda** | Function error | Handler exception after SNS accepted | Check Lambda logs, configure Lambda DLQ |
| **HTTPS** | Timeout | Endpoint slow or down | Reduce endpoint latency, increase timeout in delivery policy |
| **Email** | Bounce | Invalid address, mailbox full, spam filter | Verify email, check SNS bounce notifications |
| **SMS** | Delivery failure | Invalid number, carrier block, opt-out | Validate E.164 format, check opt-out list |

### Diagnostic Commands

Read-only inspection:

```bash
# Topic configuration
aws sns get-topic-attributes --topic-arn arn:aws:sns:us-east-1:123456789012:orders-topic

# List all subscriptions for topic
aws sns list-subscriptions-by-topic --topic-arn arn:aws:sns:us-east-1:123456789012:orders-topic

# Subscription details (filter policy, DLQ, raw delivery)
aws sns get-subscription-attributes --subscription-arn arn:aws:sns:us-east-1:123456789012:orders-topic:abc-123

# Check effective delivery policy
aws sns get-subscription-attributes \
  --subscription-arn arn:aws:sns:us-east-1:123456789012:orders-topic:abc-123 \
  --query 'Attributes.EffectiveDeliveryPolicy'

# List pending confirmations
aws sns list-subscriptions --query 'Subscriptions[?SubscriptionArn==`PendingConfirmation`]'
```

### Key Metrics to Monitor

| Metric | What It Means | Alert Threshold |
|--------|---------------|-----------------|
| `NumberOfMessagesPublished` | Messages accepted by SNS | Baseline for comparison |
| `NumberOfNotificationsDelivered` | Successful deliveries | < 95% of published |
| `NumberOfNotificationsFailed` | Delivery failures | > 1% of published |
| `NumberOfNotificationsFilteredOut` | Blocked by filters | Unexpected increase |
| `NumberOfNotificationsFilteredOut-InvalidAttributes` | Filter type mismatch | Any occurrence |

---

## 11. Quick reference card

### Topic type decision matrix

| Requirement | Standard | FIFO |
|-------------|----------|------|
| **Ordering** | Best-effort | Strict order within MessageGroupId |
| **Deduplication** | No | Yes (5-minute window) |
| **Throughput** | ~100,000 publishes/sec | 300 TPS (3,000 with batching) per group |
| **Supported endpoints** | SQS, Lambda, HTTP/S, Email, SMS, Mobile push, Kinesis Firehose | SQS only (standard or FIFO) |
| **Direct Lambda** | Yes (async invocation) | No (must use SQS FIFO) |
| **Use case** | General fanout, notifications, alerts | Ordered events per entity |
| **Naming** | Any name | Must end in `.fifo` |

### Subscription protocols

| Protocol | Use Case | Delivery Format | Notes |
|----------|----------|-----------------|-------|
| **SQS** | Buffered, asynchronous processing | JSON envelope or raw | Most common for microservices |
| **Lambda** | Immediate, serverless processing | SNS event with Records[] | Direct async invocation |
| **HTTP/HTTPS** | External webhooks, APIs | POST with JSON body | Requires endpoint confirmation |
| **Email** | Human notifications | Plain text or JSON | Not for high-volume automation |
| **Email-JSON** | Developers/debugging | Raw JSON | Unformatted JSON in email |
| **SMS** | Mobile alerts | Plain text only | Country/carrier restrictions apply |
| **Mobile Push** | iOS/Android notifications | Platform-specific JSON | Requires platform application |
| **Kinesis Firehose** | Data streaming, analytics | Batch delivery | For data lake integration |

### Message size limits

| Configuration | Size Limit | Supported Endpoints | Notes |
|---------------|------------|---------------------|-------|
| **Default** | 256 KB | All protocols | Standard configuration |
| **Extended (requires `MaximumMessageSize` attribute)** | 1 MB | SQS, Lambda, Firehose only | Max 100 subscriptions |
| **Large payloads** | Use S3 reference | All protocols | Store in S3, send reference |

### Filter policy scope

| Scope | Filters On | Use Case | Example |
|-------|-----------|----------|---------|
| **MessageAttributes** | Message attributes only | Simple routing based on metadata | `{"eventType": ["OrderCreated"]}` |
| **MessageBody** | JSON message body | Content-based routing | `{"total": [{"numeric": [">=", 100]}]}` |

### Filter policy operators

| Operator | Example | Matches |
|----------|---------|---------|
| **Exact match** | `{"color": ["red"]}` | color = "red" |
| **Multiple values (OR)** | `{"color": ["red", "blue"]}` | color = "red" OR "blue" |
| **Anything-but** | `{"color": [{"anything-but": "red"}]}` | color ≠ "red" |
| **Numeric range** | `{"price": [{"numeric": [">=", 100]}]}` | price ≥ 100 |
| **Numeric between** | `{"price": [{"numeric": [">=", 100, "<=", 200]}]}` | 100 ≤ price ≤ 200 |
| **Prefix** | `{"region": [{"prefix": "us-"}]}` | region starts with "us-" |
| **Suffix** | `{"file": [{"suffix": ".jpg"}]}` | file ends with ".jpg" |
| **Exists** | `{"priority": [{"exists": true}]}` | priority attribute exists |
| **Does not exist** | `{"debug": [{"exists": false}]}` | debug attribute missing |

### Key attributes to set

| Attribute | Level | Purpose | Values |
|-----------|-------|---------|--------|
| **FifoTopic** | Topic | Enable FIFO ordering | `true` / `false` |
| **ContentBasedDeduplication** | Topic | Auto-generate dedup ID from content | `true` / `false` |
| **MaximumMessageSize** | Topic | Maximum message size | `1024` - `1048576` bytes |
| **DisplayName** | Topic | Friendly name for email subscriptions | Any string |
| **FilterPolicy** | Subscription | Message filtering rules | JSON policy |
| **FilterPolicyScope** | Subscription | Where to apply filter | `MessageAttributes` / `MessageBody` |
| **RawMessageDelivery** | Subscription | Skip SNS envelope | `true` / `false` |
| **RedrivePolicy** | Subscription | Dead letter queue for failures | `{"deadLetterTargetArn":"..."}` |

### SNS envelope structure

**Without RawMessageDelivery (default):**
```json
{
  "Type": "Notification",
  "MessageId": "sns-generated-uuid",
  "TopicArn": "arn:aws:sns:us-east-1:123456789012:topic",
  "Subject": "Optional subject",
  "Message": "{\"your\":\"json-string\"}",
  "Timestamp": "2026-10-13T10:00:00.000Z",
  "SignatureVersion": "1",
  "Signature": "...",
  "SigningCertURL": "https://...",
  "UnsubscribeURL": "https://...",
  "MessageAttributes": {
    "key": {"Type": "String", "Value": "value"}
  }
}
```

**With RawMessageDelivery=true:**
```
{"your":"json-string"}
```

### Message attributes data types

| DataType | Example Value | Use Case |
|----------|---------------|----------|
| **String** | `"OrderCreated"` | Event types, IDs |
| **String.Array** | `["red", "blue"]` | Multiple values |
| **Number** | `123.45` | Prices, quantities |
| **Binary** | `base64-encoded` | Small binary data |

**Note:** Attribute names, types, and values count toward message size limit.

### Delivery retry behavior by protocol

| Protocol | Initial Retry | Backoff | Max Duration | Success Criteria |
|----------|---------------|---------|--------------|------------------|
| **SQS** | Immediate | N/A | Until queue accepts | 200 OK from SQS |
| **Lambda** | 3 retries | Exponential (1s, 2s, 4s) | ~5 minutes | 2xx from Lambda |
| **HTTP/S** | 3 retries (healthy), 100 (throttled) | Exponential | 1 hour | 200, 201, 202, 204, 102 |
| **Email** | Varies by provider | Provider-specific | Hours | Accepted by mail server |
| **SMS** | 3 retries | Linear | ~20 seconds | Carrier accepts |

### CloudWatch metrics

| Metric | Dimension | What It Measures | Alerted Threshold |
|--------|-----------|------------------|-------------------|
| **NumberOfMessagesPublished** | Topic | Messages published to topic | Baseline for comparison |
| **NumberOfNotificationsDelivered** | Topic, Protocol | Successful deliveries | < 95% of published |
| **NumberOfNotificationsFailed** | Topic, Protocol | Delivery failures | > 1% of published |
| **NumberOfNotificationsFilteredOut** | Topic | Messages blocked by filters | Unexpected spike |
| **NumberOfNotificationsFilteredOut-InvalidAttributes** | Topic | Filter type/value mismatch | > 0 (indicates schema error) |
| **NumberOfNotificationsRedrivenToDlq** | Topic | Messages sent to subscription DLQ | > 0 (delivery failure) |
| **SMSSuccessRate** | None | SMS delivery success rate | < 90% |
| **PublishSize** | Topic | Average message size | Approaching limit |

### Cost optimization

| Strategy | Savings | Implementation |
|----------|---------|----------------|
| ✅ **Use subscription filters** | Reduce unnecessary deliveries | Filter at SNS, not in consumer |
| ✅ **Enable RawMessageDelivery** | Reduce SQS message size | Save on SQS storage/transfer |
| ✅ **Batch publish when possible** | Reduce API calls | Use PublishBatch API |
| ✅ **Right-size message payloads** | Reduce transfer costs | Use S3 for >256KB data |
| ✅ **Monitor filtered messages** | Identify over-broad subscriptions | Alert on high filter rate |
| ✅ **Use SQS over HTTP/S** | Lower per-delivery cost | SQS cheaper than HTTP retries |
| ✅ **Consolidate topics** | Reduce fixed costs | Use filters instead of multiple topics |

### Common API operations

| Operation | Purpose | Key Parameters | Returns |
|-----------|---------|----------------|---------|
| **CreateTopic** | Create new topic | Name, Attributes (FifoTopic, etc.) | TopicArn |
| **Subscribe** | Add subscription | TopicArn, Protocol, Endpoint, Attributes | SubscriptionArn |
| **Publish** | Send message | TopicArn/TargetArn, Message, MessageAttributes | MessageId |
| **PublishBatch** | Send up to 10 messages | TopicArn, PublishBatchRequestEntries | Successful/Failed lists |
| **SetSubscriptionAttributes** | Update subscription | SubscriptionArn, AttributeName, AttributeValue | Status |
| **GetTopicAttributes** | Read topic config | TopicArn | Attributes map |
| **ConfirmSubscription** | Confirm endpoint | TopicArn, Token | SubscriptionArn |
| **Unsubscribe** | Remove subscription | SubscriptionArn | Status |

### SNS vs other messaging services

| Use Case | SNS | SQS | EventBridge | Kinesis |
|----------|-----|-----|-------------|---------|
| **Fan-out (1→N)** | ✅ Best | ❌ | ✅ With rules | ⚠️ Multiple consumers |
| **Pull model** | ❌ Push | ✅ | ❌ Push | ✅ |
| **Ordering** | ⚠️ FIFO only | ⚠️ FIFO only | ❌ | ✅ |
| **Durability** | ⚠️ Retry only | ✅ Queue | ⚠️ Archive | ✅ 1-365 days |
| **Replay** | ❌ | ❌ | ✅ Archive | ✅ |
| **Filtering** | ✅ Per subscription | ❌ | ✅ Rules | ❌ |
| **Direct notifications** | ✅ Email, SMS, Push | ❌ | ⚠️ Limited | ❌ |
| **Throughput** | Very high | High | Very high | Very high |
| **Latency** | <100ms | Polling delay | <1s | <1s |

---

## 12. Exam tips summary

### Must-know concepts

1. **SNS is pub/sub (fanout), SQS is queue (competing consumers).** One SNS message → multiple independent copies. One SQS message → one consumer.

2. **SNS pushes, SQS pulls.** SNS delivers immediately to subscriptions. SQS waits for consumers to poll.

3. **SNS + SQS = fan-out with buffering.** Best practice for microservices: SNS topic → multiple SQS queues (one per service).

4. **FIFO topics can only deliver to SQS.** No direct Lambda, HTTP, Email, SMS, or mobile push for FIFO topics.

5. **Subscription filters reduce cost and complexity.** Filter at SNS level, not in every consumer.

6. **RawMessageDelivery changes the envelope.** Without it: SNS wraps message in JSON envelope. With it: delivers original Message string only.

7. **SNS DLQ ≠ SQS consumer DLQ.** SNS DLQ captures SNS delivery failures (queue unreachable). SQS DLQ captures processing failures (handler errors).

8. **Message size: 256 KB default, 1 MB configurable.** Extended size (>256KB) only supports SQS, Lambda, Firehose endpoints.

9. **Cross-account requires both policies.** Topic policy (source account) + Queue/Lambda policy (target account).

10. **FIFO deduplication window = 5 minutes.** Prevents duplicate publishes within window, but consumers still need idempotency for retries.

11. **FilterPolicyScope determines filter location.** MessageAttributes (default) filters on attributes. MessageBody filters on JSON body content.

12. **Mobile push requires platform application.** Create platform app (APNS, FCM) before creating endpoints.

13. **Email subscriptions require confirmation.** HTTP/S also requires confirmation. SQS/Lambda don't need confirmation.

14. **SNS retries depend on protocol.** HTTP: 100 retries over 1 hour. Lambda: 3 retries over ~5 min. SQS: until successful.

15. **One subscription per endpoint per topic.** Same queue can't be subscribed twice to same topic (update filter instead).

### Common exam scenarios

| Scenario | Correct Answer | Wrong Answers to Avoid |
|----------|----------------|------------------------|
| Multiple services need same event | SNS topic → SQS queues (one per service) | ❌ One SQS queue with multiple consumers |
| Send email when order created | SNS with Email subscription | ❌ SES directly from app, ❌ SQS |
| Process events in strict order | SNS FIFO → SQS FIFO | ❌ Standard SNS, ❌ SNS FIFO → Lambda |
| Filter messages by attribute | Subscription filter policy | ❌ Filter in every consumer |
| Cross-account event delivery | Topic policy + Queue policy | ❌ Topic policy alone, ❌ IAM policy alone |
| Capture SNS delivery failures | Subscription DLQ (RedrivePolicy) | ❌ SQS consumer DLQ |
| Large 500KB messages | Set MaximumMessageSize=1MB, use SQS/Lambda | ❌ Split message, ❌ Default (fails) |
| Reduce fanout costs | Subscription filters | ❌ Multiple topics, ❌ Filter in consumers |
| Mobile push to iOS/Android | SNS Platform Application + endpoints | ❌ SNS topic alone, ❌ SES |
| Guaranteed exactly-once processing | SNS FIFO + idempotent consumers | ❌ SNS FIFO alone (still need idempotency) |
| Lambda gets duplicate invocations | Design idempotent handler | ❌ Use FIFO (wrong protocol), ❌ Prevent retries |
| Message order lost | Use FIFO topic + FIFO queue + MessageGroupId | ❌ Standard topic, ❌ Multiple groups |

### Red flags (wrong answer indicators)

- ❌ **"Use SQS for fanout to multiple services"** → SQS is queue, use SNS for fanout
- ❌ **"One SQS queue with all consumers"** → Each service needs its own queue
- ❌ **"SNS FIFO with direct Lambda"** → FIFO only supports SQS endpoints
- ❌ **"SNS guarantees exactly-once processing"** → Only deduplication; consumers need idempotency
- ❌ **"Filter in consumer application"** → Filter at subscription level (cheaper, simpler)
- ❌ **"RawMessageDelivery for all cases"** → Removes SNS envelope (may break consumers)
- ❌ **"Send 500KB messages with default config"** → Exceeds 256KB default limit
- ❌ **"SNS stores messages"** → SNS delivers immediately, no storage (use SQS for buffering)
- ❌ **"Topic policy alone for cross-account"** → Need both topic and queue/Lambda policies
- ❌ **"SNS retry handles processing errors"** → Retries delivery failures only, not logic errors

### Quick decision flowchart

```
Need messaging?
  │
  ├─ Multiple independent services need same event?
  │  └─ YES → SNS fanout to multiple SQS queues
  │
  ├─ Need buffering and retry?
  │  └─ YES → SNS → SQS → Workers
  │
  ├─ Immediate notification (email, SMS, mobile)?
  │  └─ YES → SNS with Email/SMS/Mobile subscription
  │
  ├─ Single consumer, task queue?
  │  └─ YES → SQS only (no SNS needed)
  │
  ├─ Need strict ordering per entity?
  │  └─ YES → SNS FIFO → SQS FIFO (MessageGroupId)
  │
  └─ Need selective routing?
     └─ YES → SNS with subscription filters

SNS Standard vs FIFO?
  │
  ├─ Need strict ordering? → FIFO
  ├─ Need highest throughput? → Standard
  ├─ Need Email/SMS/HTTP/Mobile? → Standard (FIFO doesn't support)
  └─ Need deduplication? → FIFO
```

---

## 13. Interview questions and detailed answers

### Setup and the business event

These examples use **Python 3 with Boto3** and **Java 17+ with AWS SDK for Java 2.x**. Java dependencies are `software.amazon.awssdk:sns`, Jackson 2.x `com.fasterxml.jackson.core:jackson-databind`, and `com.amazonaws:aws-lambda-java-core` for the handler. Manage SDK module versions with an approved AWS SDK 2.x BOM and pin compatible Jackson/Lambda library versions in your build.

Configure `AWS_REGION` and `SNS_TOPIC_ARN`; use a local AWS profile or a workload role through the default credential chain. Publishing requires `sns:Publish`. The example assumes an existing **standard topic** and, for receiving, an existing SQS subscription or authorized Lambda subscription. Publishing does not create subscribers.

Use the same business event across the SNS, SQS, and EventBridge examples:

```json
{
  "schemaVersion": 1,
  "eventId": "evt-order-456-created",
  "eventType": "OrderCreated",
  "orderId": "456",
  "country": "IN",
  "total": 1200
}
```

The sample ID represents one logical event. Generate a new ID for a new event and preserve it on retries. `total` is an illustrative numeric value; real payment contracts should define currency and units explicitly.

### Publish request versus delivered message

SNS `Message` is a **string**. Serialize your JSON object into that string; SNS does not require every application message to be JSON. `MessageAttributes` are separate typed routing metadata. Do **not** set `MessageStructure="json"` just because the business payload is JSON: that option means protocol-specific messages with a `default` entry. [Boto3 Publish](https://docs.aws.amazon.com/boto3/latest/reference/services/sns/client/publish.html)

SDK request shape, shown as JSON for readability:

```json
{
  "TopicArn": "arn:aws:sns:ap-south-1:123456789012:orders",
  "Message": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}",
  "MessageAttributes": {
    "eventType": {"DataType": "String", "StringValue": "OrderCreated"}
  }
}
```

| Subscription path | What the receiver sees | Parsing |
|---|---|---|
| SNS → SQS, default wrapping | SQS `Body` is a JSON SNS envelope; its `Message` is the original string | Parse `Body`, then parse `Message` |
| SNS → SQS, raw delivery | SQS `Body` is the original published string | Parse `Body` once |
| SNS → Lambda directly | Lambda event has `Records[].Sns.Message` | Parse each `Message` string |

The following is a **reduced SNS envelope after parsing SQS `Body`**, not the complete `ReceiveMessage` response. Signature/certificate/unsubscribe metadata is omitted:

```json
{
  "Type": "Notification",
  "MessageId": "sns-generated-id",
  "TopicArn": "arn:aws:sns:ap-south-1:123456789012:orders",
  "Message": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}",
  "Timestamp": "2026-10-13T10:00:00Z",
  "MessageAttributes": {
    "eventType": {"Type": "String", "Value": "OrderCreated"}
  }
}
```

Notice the attribute representation: the publish API uses `DataType`/`StringValue`, while the SNS JSON envelope uses `Type`/`Value`. Raw delivery removes the SNS envelope; for SQS it supports at most ten message attributes. Configure `RawMessageDelivery` on the **subscription**, not on each publish request. [Raw delivery](https://docs.aws.amazon.com/sns/latest/dg/sns-large-payload-raw-message-delivery.html)

### Python: publish

```python
import json
import os
import boto3


def publish_order():
    order = {
        "schemaVersion": 1,
        "eventId": "evt-order-456-created",
        "eventType": "OrderCreated",
        "orderId": "456",
        "country": "IN",
        "total": 1200,
    }
    sns = boto3.client("sns", region_name=os.environ["AWS_REGION"])
    result = sns.publish(
        TopicArn=os.environ["SNS_TOPIC_ARN"],
        Message=json.dumps(order),
        MessageAttributes={
            "eventType": {"DataType": "String", "StringValue": "OrderCreated"}
        },
    )
    print("SNS accepted:", result["MessageId"])


if __name__ == "__main__":
    publish_order()
```

`MessageId` identifies SNS acceptance, not the business event and not completion by all subscribers.

### Python: receive through a Lambda subscription

Reduced Lambda input:

```json
{
  "Records": [
    {
      "EventSource": "aws:sns",
      "Sns": {
        "MessageId": "sns-generated-id",
        "TopicArn": "arn:aws:sns:ap-south-1:123456789012:orders",
        "Message": "{\"schemaVersion\":1,\"eventId\":\"evt-order-456-created\",\"eventType\":\"OrderCreated\",\"orderId\":\"456\",\"country\":\"IN\",\"total\":1200}"
      }
    }
  ]
}
```

```python
import json


def lambda_handler(event, context):
    for record in event["Records"]:
        order = json.loads(record["Sns"]["Message"])
        if order["schemaVersion"] != 1 or order["eventType"] != "OrderCreated":
            raise ValueError("Unsupported order event")
        # Demo processing only. Replace with a durable, idempotent operation.
        print("Received order:", order["orderId"], "event:", order["eventId"])
```

SNS pushes; there is no `sns.receive_message()` polling API. Returning successfully acknowledges the Lambda invocation. Propagate processing failures so Lambda can apply its asynchronous failure policy. Returning an object containing an error string is still a successful handler return. [SNS Lambda event format](https://docs.aws.amazon.com/lambda/latest/dg/with-sns.html)

### Java: publish and receive through Lambda

Save as `SnsExample.java`. Run `main` to publish; for a Lambda deployment configure handler `SnsExample::handleRequest`. Publishing and receiving are separate invocations of the example, not an automatically wired subscription.

```java
import com.amazonaws.services.lambda.runtime.Context;
import com.amazonaws.services.lambda.runtime.RequestHandler;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import software.amazon.awssdk.regions.Region;
import software.amazon.awssdk.services.sns.SnsClient;
import software.amazon.awssdk.services.sns.model.MessageAttributeValue;
import software.amazon.awssdk.services.sns.model.PublishRequest;
import java.util.Map;

public class SnsExample implements RequestHandler<Map<String, Object>, Void> {
    private static final ObjectMapper JSON = new ObjectMapper();

    public static void main(String[] args) throws Exception {
        String body = JSON.writeValueAsString(Map.of(
            "schemaVersion", 1, "eventId", "evt-order-456-created",
            "eventType", "OrderCreated", "orderId", "456",
            "country", "IN", "total", 1200));
        try (SnsClient sns = SnsClient.builder()
                .region(Region.of(System.getenv("AWS_REGION"))).build()) {
            var result = sns.publish(PublishRequest.builder()
                .topicArn(System.getenv("SNS_TOPIC_ARN"))
                .message(body)
                .messageAttributes(Map.of("eventType",
                    MessageAttributeValue.builder().dataType("String")
                        .stringValue("OrderCreated").build()))
                .build());
            System.out.println("SNS accepted: " + result.messageId());
        }
    }

    @Override
    public Void handleRequest(Map<String, Object> event, Context context) {
        JsonNode root = JSON.valueToTree(event);
        try {
            for (JsonNode record : root.required("Records")) {
                JsonNode order = JSON.readTree(
                    record.required("Sns").required("Message").asText());
                if (order.required("schemaVersion").asInt() != 1
                    || !"OrderCreated".equals(order.required("eventType").asText())) {
                    throw new IllegalArgumentException("Unsupported order event");
                }
                // Demo only: replace with durable, idempotent processing.
                context.getLogger().log("Order " + order.required("orderId").asText()
                    + ", event " + order.required("eventId").asText() + "\n");
            }
            return null;
        } catch (Exception e) {
            throw new RuntimeException("SNS event processing failed", e);
        }
    }
}
```

### FIFO and SQS consumer variations

For a FIFO topic, add `MessageGroupId=orderId` and `MessageDeduplicationId=eventId` to Python's publish arguments; Java's builder equivalents are `.messageGroupId(orderId)` and `.messageDeduplicationId(eventId)`. Reuse the same deduplication ID for the same logical event. FIFO Lambda delivery goes through SQS; the direct Lambda handler above is for a standard-topic subscription.

For SNS → SQS, use the [SQS polling examples](aws-sqs-cheatsheet.md#12-message-formats-and-python-and-java-examples). Select the known body mode explicitly: `sns-envelope` for wrapped delivery, `direct` for raw delivery. The consumer deletes the **SQS receipt handle** after processing; there is no SNS delete/ack call.

---

## 15. AWS sources

- [Large-message support](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html)
- [Publish API](https://docs.aws.amazon.com/sns/latest/api/API_Publish.html)
- [FIFO delivery](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-delivery.html)
- [FIFO deduplication](https://docs.aws.amazon.com/sns/latest/dg/fifo-message-dedup.html)
- [Filtering](https://docs.aws.amazon.com/sns/latest/dg/sns-message-filtering.html)
- [Subscription dead-letter queues](https://docs.aws.amazon.com/sns/latest/dg/sns-dead-letter-queues.html)
- [Boto3 publish request](https://docs.aws.amazon.com/boto3/latest/reference/services/sns/client/publish.html)
- [Raw message delivery](https://docs.aws.amazon.com/sns/latest/dg/sns-large-payload-raw-message-delivery.html)
- [SNS Lambda input](https://docs.aws.amazon.com/lambda/latest/dg/with-sns.html)
- [Java Lambda handler contracts](https://docs.aws.amazon.com/lambda/latest/dg/java-handler.html)
- [SNS Java SDK examples](https://docs.aws.amazon.com/code-library/latest/ug/java_2_sns_code_examples.html)

**Related:** [SQS](aws-sqs-cheatsheet.md), [EventBridge](aws-eventbridge-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md).
