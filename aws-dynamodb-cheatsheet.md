# Amazon DynamoDB Cheat Sheet

**Interview preparation: access patterns, keys, indexes, capacity, consistency, transactions, and global tables.**

**Verified:** 2026-10-13. Global-table consistency mode and Region support materially change the guarantees.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Access-pattern-first modeling](#2-access-pattern-first-modeling)
- [3. GSI versus LSI](#3-gsi-versus-lsi)
- [4. Capacity math and hot keys](#4-capacity-math-and-hot-keys)
- [5. Consistency and transactions](#5-consistency-and-transactions)
- [6. Current correction: global tables have two modes](#6-current-correction-global-tables-have-two-modes)
- [7. Streams, TTL, caching, and recovery](#7-streams-ttl-caching-and-recovery)
- [8. Security and cost](#8-security-and-cost)
- [9. Common Patterns](#9-common-patterns)
- [10. Troubleshooting and practical inspection](#10-troubleshooting-and-practical-inspection)
- [11. Interview questions and answers](#11-interview-questions-and-answers)
- [Quick Reference Card](#quick-reference-card)
- [Exam Tips Summary](#exam-tips-summary)
- [12. Quick revision](#12-quick-revision)
- [13. AWS sources](#13-aws-sources)

---

## 1. Fundamentals and mental model

DynamoDB is a managed key-value/document database. Model known access patterns into keys and indexes rather than expecting arbitrary relational joins at request time.

```text
Access pattern → partition key + optional sort key → Query/GetItem
              → alternative access pattern → secondary index
Writes → conditional updates / transactions → change stream → consumers
```

| Concept | Purpose |
|---|---|
| Partition key | Distributes data and identifies an item collection |
| Sort key | Orders/selects items within a partition-key value |
| Item | Record with flexible non-key attributes |
| GSI | Alternate partition/sort-key access path |
| LSI | Alternate sort key under the same partition key |
| Condition expression | Makes a write depend on current item state |

## 2. Access-pattern-first modeling

Example order model:

| PK | SK | Meaning |
|---|---|---|
| `CUSTOMER#123` | `ORDER#2026-10-13#456` | Orders for one customer in time order |
| `ORDER#456` | `META` | Order details |
| `ORDER#456` | `ITEM#001` | Order line |

This is an illustrative model, not a requirement to duplicate every order. If duplicated views are used, define which write path keeps them consistent.

Ask: lookup keys, sorting, range queries, cardinality, hottest tenant/entity, growth, consistency, and transactional boundaries. Single-table design can reduce round trips for related entities; it also increases modeling complexity and is not mandatory.

`Query` selects a partition-key value and optional sort-key conditions. `Scan` reads across data. A filter expression is evaluated after reading and does not turn a scan into an efficient key lookup.

## 3. GSI versus LSI

| Property | GSI | LSI |
|---|---|---|
| Partition key | Can differ from table | Same as table |
| Sort key | Alternate, optional depending on key schema | Alternate sort key |
| Creation | Can add later | At table creation |
| Read consistency | Eventual | Eventual or strong |
| Provisioned capacity | Separate index capacity | Shares table capacity |
| Important constraint | Project needed attributes; account for write amplification | 10 GB item-collection limit with LSIs |

An overloaded GSI can affect writes to its base table. A sparse GSI indexes only items with the necessary index-key attributes, which can model work queues or exceptional states efficiently. [Secondary indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html)

Current documentation also distinguishes **vector indexes** for similarity search from secondary indexes for key queries; those use a different API/access model. Do not describe a GSI as a vector-search index.

## 4. Capacity math and hot keys

| Operation | Capacity unit basis |
|---|---|
| Standard strongly consistent read | 1 read unit per 4 KiB, rounded up |
| Eventually consistent read | Half the corresponding strong-read units |
| Standard write | 1 write unit per 1 KiB, rounded up |
| Transactional read/write | Twice the corresponding nontransactional unit requirement |

Example: 100 strongly consistent reads/second of 6 KiB items require roughly 200 RCUs. At eventual consistency, roughly 100 RCUs. Fifty writes/second of 1.5 KiB items require roughly 100 WCUs, before index/transaction effects. [Read/write operations](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/read-write-operations.html)

On-demand capacity simplifies provisioning and suits variable demand. Provisioned capacity with auto scaling can suit predictable usage. Neither makes a single hot key infinitely scalable. Spread keys, consider write sharding for suitable workloads, and include the extra read/merge complexity.

## 5. Consistency and transactions

Tables and LSIs support optional strong reads; GSIs and Streams do not. A strong read is not a lock: another writer can change the item immediately afterward. Use conditional writes for optimistic concurrency. [Read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)

| Limit | Current value |
|---|---|
| Item | 400 KiB including attribute names/values |
| Transaction | Up to 100 unique items and 4 MiB total |
| Query/Scan page | Up to 1 MiB before filtering; paginate |
| BatchGetItem | Up to 100 items and 16 MiB; partial results possible |
| BatchWriteItem | Up to 25 puts/deletes; not a transaction |

Inspect unprocessed keys/items and retry with backoff. A successful batch response can still contain unprocessed work. [Constraints](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Constraints.html)

## 6. Current correction: global tables have two modes

| Mode | Replication and behavior | Best fit |
|---|---|---|
| MREC, default | Asynchronous; concurrent cross-Region conflicts use last-writer-wins | Low local latency with tolerated replication lag |
| MRSC | Synchronous cross-Region coordination; strong reads can see latest writes globally | Global correctness requirements and zero-RPO design |

MRSC uses exactly three Regions: three replicas or two replicas plus a witness. The witness is not a readable application replica. MRSC currently does not support TTL, LSIs, or transaction APIs. MREC transactions are atomic in the originating Region, not replicated as one atomic multi-item unit. [Global-table behavior](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html)

On **September 23, 2026**, AWS expanded MRSC to 15 supported Regions and allowed any three-Region combination among them, including cross-continent configurations. Mumbai and Singapore are included. Inter-Region latency remains a design input. [MRSC expansion](https://aws.amazon.com/about-aws/whats-new/2026/09/dynamodb-mrsc-additional-regions/)

**Interview correction:** “All global tables are eventually consistent” is outdated. State mode, read type, feature constraints, and latency tradeoff.

## 7. Streams, TTL, caching, and recovery

- DynamoDB Streams captures item changes for downstream processing. Consumers must handle retries and duplicates.
- TTL is asynchronous cleanup, not a precise timer or immediate security revocation. Enforce expiry in application logic where correctness requires it. MRSC has the stated TTL restriction.
- DAX can accelerate suitable read patterns, but caching is not a substitute for required consistency or good keys.
- PITR and backups protect against logical mistakes; replication can propagate the mistake.
- Store large object bytes in S3 and keep references/metadata in DynamoDB; coordinate lifecycle and orphan cleanup.

Do not treat Streams retention as an indefinite event archive. Capture events into an appropriate durable history if long replay is a requirement.

## 8. Security and cost

Use least-privilege IAM and supported resource policies, encryption, endpoint controls, and tenant-aware key design. A shared table is not tenant-isolated merely because each tenant has a different partition key; permissions and application authorization must enforce that boundary.

Cost includes read/write units, indexes, storage, backups, replication, streams/integrations, and caches. Large items and excessive index projections amplify cost. Measure consumed capacity and hot partitions before choosing a larger budget.

## 9. Common Patterns

Real-world DynamoDB architectures and implementation patterns with complete configurations.

### Pattern 1: Single-Table E-Commerce Design

**Use Case:** Store customers, orders, and order items in one table for efficient queries.

**Architecture:**
```
Single Table: Orders
├── Access Pattern 1: Get customer details → PK: CUSTOMER#<id>
├── Access Pattern 2: List customer orders → PK: CUSTOMER#<id>, SK: ORDER#<timestamp>#<id>
├── Access Pattern 3: Get order details → PK: ORDER#<id>, SK: META
├── Access Pattern 4: List order items → PK: ORDER#<id>, SK: ITEM#<id>
└── Access Pattern 5: Find orders by status → GSI: OrderStatusIndex
```

**Table Schema:**
```json
{
  "TableName": "ECommerceStore",
  "KeySchema": [
    {"AttributeName": "PK", "KeyType": "HASH"},
    {"AttributeName": "SK", "KeyType": "RANGE"}
  ],
  "AttributeDefinitions": [
    {"AttributeName": "PK", "AttributeType": "S"},
    {"AttributeName": "SK", "AttributeType": "S"},
    {"AttributeName": "GSI1PK", "AttributeType": "S"},
    {"AttributeName": "GSI1SK", "AttributeType": "S"}
  ],
  "BillingMode": "PAY_PER_REQUEST",
  "GlobalSecondaryIndexes": [
    {
      "IndexName": "OrderStatusIndex",
      "KeySchema": [
        {"AttributeName": "GSI1PK", "KeyType": "HASH"},
        {"AttributeName": "GSI1SK", "KeyType": "RANGE"}
      ],
      "Projection": {"ProjectionType": "ALL"}
    }
  ]
}
```

**Example Items:**
```json
// Customer record
{
  "PK": "CUSTOMER#alice@example.com",
  "SK": "PROFILE",
  "EntityType": "Customer",
  "Name": "Alice Johnson",
  "Email": "alice@example.com",
  "CreatedAt": "2026-01-15T10:30:00Z"
}

// Order record
{
  "PK": "CUSTOMER#alice@example.com",
  "SK": "ORDER#2026-10-13T14:20:00Z#456",
  "EntityType": "Order",
  "OrderId": "456",
  "Status": "SHIPPED",
  "Total": 125.99,
  "CreatedAt": "2026-10-13T14:20:00Z",
  "GSI1PK": "STATUS#SHIPPED",
  "GSI1SK": "2026-10-13T14:20:00Z"
}

// Order metadata (for direct order lookup)
{
  "PK": "ORDER#456",
  "SK": "META",
  "EntityType": "OrderMeta",
  "CustomerId": "alice@example.com",
  "Status": "SHIPPED",
  "Total": 125.99
}

// Order item
{
  "PK": "ORDER#456",
  "SK": "ITEM#001",
  "EntityType": "OrderItem",
  "ProductId": "PROD-789",
  "Quantity": 2,
  "Price": 62.99
}
```

**Query Examples:**
```python
import boto3
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('ECommerceStore')

# Get all orders for a customer (with pagination)
response = table.query(
    KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
    ExpressionAttributeValues={
        ':pk': 'CUSTOMER#alice@example.com',
        ':sk': 'ORDER#'
    },
    ScanIndexForward=False  # Newest first
)

# Get order details and items in one query
response = table.query(
    KeyConditionExpression='PK = :pk',
    ExpressionAttributeValues={':pk': 'ORDER#456'}
)

# Find all shipped orders
response = table.query(
    IndexName='OrderStatusIndex',
    KeyConditionExpression='GSI1PK = :status',
    ExpressionAttributeValues={':status': 'STATUS#SHIPPED'}
)
```

**Why This Pattern:**
- Single query returns customer + all orders (related data locality)
- Hierarchical sort key enables efficient prefix searches
- GSI provides alternate access pattern without scan
- Transaction can atomically update order + items

**Tradeoffs:**
- More complex data modeling (learning curve)
- Item size limits (400 KB) affect large orders
- GSI write amplification (every write writes to base + indexes)
- Harder to add ad-hoc queries later

---

### Pattern 2: Conditional Inventory Decrement

**Use Case:** Prevent overselling with optimistic locking.

**Item Structure:**
```json
{
  "PK": "PRODUCT#789",
  "SK": "INVENTORY",
  "AvailableQuantity": 50,
  "ReservedQuantity": 5,
  "Version": 12
}
```

**Conditional Update (Prevents Negative Inventory):**
```python
try:
    response = table.update_item(
        Key={'PK': 'PRODUCT#789', 'SK': 'INVENTORY'},
        UpdateExpression='SET AvailableQuantity = AvailableQuantity - :qty, #ver = #ver + :inc',
        ConditionExpression='AvailableQuantity >= :qty',
        ExpressionAttributeNames={'#ver': 'Version'},
        ExpressionAttributeValues={
            ':qty': 3,
            ':inc': 1
        },
        ReturnValues='ALL_NEW'
    )
    print(f"Success: New quantity = {response['Attributes']['AvailableQuantity']}")
except ClientError as e:
    if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
        print("Insufficient inventory")
    else:
        raise
```

**Transaction-Based Approach (Multi-Item Atomicity):**
```python
# Decrement inventory AND create order atomically
try:
    response = dynamodb_client.transact_write_items(
        TransactItems=[
            {
                'Update': {
                    'TableName': 'ECommerceStore',
                    'Key': {'PK': {'S': 'PRODUCT#789'}, 'SK': {'S': 'INVENTORY'}},
                    'UpdateExpression': 'SET AvailableQuantity = AvailableQuantity - :qty',
                    'ConditionExpression': 'AvailableQuantity >= :qty',
                    'ExpressionAttributeValues': {':qty': {'N': '3'}}
                }
            },
            {
                'Put': {
                    'TableName': 'ECommerceStore',
                    'Item': {
                        'PK': {'S': 'ORDER#456'},
                        'SK': {'S': 'META'},
                        'Status': {'S': 'PENDING'},
                        'Total': {'N': '89.97'}
                    }
                }
            }
        ]
    )
    print("Transaction successful")
except ClientError as e:
    if e.response['Error']['Code'] == 'TransactionCanceledException':
        reasons = e.response['Error']['Message']
        print(f"Transaction failed: {reasons}")
    else:
        raise
```

**Global Considerations (MREC mode):**
```
MREC (eventual consistency across regions):
├── Region A: Decrement inventory (condition passes locally)
├── Region B: Decrement inventory (condition passes locally)
└── Result: Both succeed locally, inventory goes negative globally

Solutions:
1. Use MRSC mode (synchronous cross-region, 3 regions only)
2. Primary region for inventory writes + read replicas
3. Reserve inventory with compensating transactions
4. Accept eventual consistency + periodic reconciliation
```

**Exam Tip:** Conditional writes prevent local race conditions. For global correctness, MREC requires application-level coordination or MRSC mode.

---

### Pattern 3: DynamoDB Streams → Lambda Event Processing

**Use Case:** Send email notifications when order status changes.

**Architecture:**
```
DynamoDB Table → DynamoDB Stream → Lambda → SES/SNS
  (order update)      (change capture)    (notification)
```

**Enable Streams:**
```bash
aws dynamodb update-table \
  --table-name ECommerceStore \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

**Lambda Function (Python):**
```python
import json
import boto3

ses = boto3.client('ses')

def lambda_handler(event, context):
    for record in event['Records']:
        if record['eventName'] == 'MODIFY':
            old_image = record['dynamodb'].get('OldImage', {})
            new_image = record['dynamodb']['NewImage']
            
            # Check if order status changed
            old_status = old_image.get('Status', {}).get('S')
            new_status = new_image['Status']['S']
            
            if old_status != new_status and new_status == 'SHIPPED':
                customer_email = new_image['CustomerEmail']['S']
                order_id = new_image['OrderId']['S']
                
                # Idempotency: check if notification already sent
                notification_key = f"NOTIFICATION#{order_id}#{new_status}"
                try:
                    table.put_item(
                        Item={
                            'PK': notification_key,
                            'SK': 'SENT',
                            'Timestamp': str(datetime.utcnow())
                        },
                        ConditionExpression='attribute_not_exists(PK)'
                    )
                    
                    # Send email
                    ses.send_email(
                        Source='orders@example.com',
                        Destination={'ToAddresses': [customer_email]},
                        Message={
                            'Subject': {'Data': f'Order {order_id} Shipped'},
                            'Body': {'Text': {'Data': f'Your order has shipped!'}}
                        }
                    )
                    print(f"Notification sent for order {order_id}")
                    
                except ClientError as e:
                    if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                        print(f"Notification already sent for {order_id}")
                    else:
                        raise
    
    return {'statusCode': 200}
```

**Lambda Event Source Mapping:**
```bash
aws lambda create-event-source-mapping \
  --function-name OrderNotificationProcessor \
  --event-source-arn arn:aws:dynamodb:us-east-1:123456789012:table/ECommerceStore/stream/2026-10-13T00:00:00.000 \
  --starting-position LATEST \
  --batch-size 100 \
  --maximum-batching-window-in-seconds 10
```

**Lambda Execution Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetRecords",
        "dynamodb:GetShardIterator",
        "dynamodb:DescribeStream",
        "dynamodb:ListStreams"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/ECommerceStore/stream/*"
    },
    {
      "Effect": "Allow",
      "Action": ["ses:SendEmail"],
      "Resource": "*"
    }
  ]
}
```

**Pitfalls to Avoid:**
- ❌ **No idempotency:** Duplicate stream records cause duplicate notifications
- ❌ **Synchronous mindset:** Stream processing is asynchronous (eventual)
- ❌ **Unbounded retries:** Failed records block shard processing
- ✅ **Solution:** Implement idempotency keys, DLQ for poison pills, monitor Lambda errors

**Exam Tip:** Streams deliver each record at least once. Make Lambda functions idempotent using conditional writes or deduplication tables.

---

### Pattern 4: Time-Series Data with TTL

**Use Case:** Store IoT sensor readings with automatic expiration.

**Table Schema:**
```json
{
  "TableName": "SensorData",
  "KeySchema": [
    {"AttributeName": "DeviceId", "KeyType": "HASH"},
    {"AttributeName": "Timestamp", "KeyType": "RANGE"}
  ],
  "AttributeDefinitions": [
    {"AttributeName": "DeviceId", "AttributeType": "S"},
    {"AttributeName": "Timestamp", "AttributeType": "N"}
  ],
  "TimeToLiveSpecification": {
    "Enabled": true,
    "AttributeName": "ExpiresAt"
  }
}
```

**Item Example:**
```json
{
  "DeviceId": "SENSOR-12345",
  "Timestamp": 1697200800,
  "Temperature": 22.5,
  "Humidity": 65,
  "ExpiresAt": 1704980800  // 90 days from now (Unix epoch seconds)
}
```

**Write Pattern (Python):**
```python
import time

def write_sensor_reading(device_id, temperature, humidity):
    current_time = int(time.time())
    expires_at = current_time + (90 * 24 * 60 * 60)  # 90 days
    
    table.put_item(
        Item={
            'DeviceId': device_id,
            'Timestamp': current_time,
            'Temperature': temperature,
            'Humidity': humidity,
            'ExpiresAt': expires_at
        }
    )
```

**Query Recent Readings:**
```python
# Get last 24 hours of readings for a device
current_time = int(time.time())
day_ago = current_time - (24 * 60 * 60)

response = table.query(
    KeyConditionExpression='DeviceId = :device AND #ts > :start',
    ExpressionAttributeNames={'#ts': 'Timestamp'},
    ExpressionAttributeValues={
        ':device': 'SENSOR-12345',
        ':start': day_ago
    },
    ScanIndexForward=False,  # Newest first
    Limit=1000
)
```

**GSI for Cross-Device Queries:**
```json
{
  "IndexName": "TimestampIndex",
  "KeySchema": [
    {"AttributeName": "SensorType", "KeyType": "HASH"},
    {"AttributeName": "Timestamp", "KeyType": "RANGE"}
  ],
  "Projection": {"ProjectionType": "ALL"}
}
```

**TTL Best Practices:**
- TTL is asynchronous (typically within 48 hours of expiration)
- Don't rely on TTL for security/compliance enforcement
- Verify item doesn't exist in queries (check ExpiresAt < now)
- TTL deletes consume write capacity (background process)
- Deleted items appear in Streams (eventName = REMOVE, userIdentity = dynamodb.amazonaws.com)

**Cost Optimization:**
```
Without TTL: 1 million items × 1 KB × $0.25/GB-month = $250/month
With TTL (90-day retention): ~250K avg items = $62.50/month
Savings: 75% storage cost reduction
```

**Exam Tip:** TTL is eventual deletion (48 hours typical). For immediate expiration enforcement, check ExpiresAt in application logic.

---

## 10. Troubleshooting and practical inspection

### Common Issues and Root Causes

| Symptom | Root Cause | Investigation | Resolution |
|---------|------------|---------------|------------|
| **Throttling with spare capacity** | Hot partition key, uneven data distribution | Check CloudWatch metrics per partition, access patterns | Add randomized suffix to partition key (write sharding) |
| **GSI throttling affects writes** | GSI has separate provisioned capacity, overloaded | Check GSI consumed vs provisioned capacity | Increase GSI WCUs or switch to on-demand |
| **Recent write missing from GSI query** | GSI is eventually consistent (propagation delay) | Query base table with strong consistency | Wait for GSI propagation (typically <1 second) |
| **Query returns empty with continuation token** | Filter expression removed all items from page | Continue pagination, items may exist in next page | Avoid filter expressions; use key conditions instead |
| **ConditionalCheckFailedException** | Item state changed between read and write (expected) | Check condition expression, item version | Retry with exponential backoff, refresh item state |
| **ValidationException on expression** | Reserved keyword used without attribute name placeholder | Use ExpressionAttributeNames for reserved words | Replace `"Name"` with `"#n"` + `{"#n": "Name"}` |
| **Item size exceeds 400 KB** | Too many attributes or large attribute values | Calculate item size (attributes + names) | Move large data to S3, store reference in DynamoDB |
| **Transaction failed** | One of 100 items failed condition/validation | Check CancellationReasons in response | Fix failing item, retry transaction |
| **Unprocessed items in BatchWriteItem** | Provisioned capacity exceeded, throttling | Exponential backoff, reduce batch size | Retry unprocessed items, increase WCUs |
| **High read costs** | Using Scan instead of Query, filtering after read | Check query patterns, access logs | Model access patterns into keys/indexes |
| **Global table conflicts** | Concurrent writes to same item in different regions (MREC) | Check item versions, last-writer-wins behavior | Use MRSC mode or coordinate writes to primary region |
| **Slow queries** | Large result sets, insufficient pagination | Check page sizes, query latency metrics | Use Limit parameter, paginate results |
| **DLQ messages from Streams** | Lambda processing errors, poison pills | Check Lambda CloudWatch logs, error types | Fix Lambda code, implement error handling |
| **TTL not deleting items** | ExpiresAt not in Unix epoch seconds, or <5 minutes past | Check TTL attribute format, current timestamp | Use Unix epoch seconds (not milliseconds) |

### Diagnostic Commands

**Check table details and capacity:**
```bash
# Get table description
aws dynamodb describe-table --table-name Orders | jq '.Table | {
  TableName,
  TableStatus,
  ItemCount,
  TableSizeBytes,
  BillingModeSummary,
  ProvisionedThroughput
}'

# Check GSI details
aws dynamodb describe-table --table-name Orders \
  | jq '.Table.GlobalSecondaryIndexes[] | {
      IndexName,
      IndexStatus,
      ProvisionedThroughput,
      ItemCount
    }'

# Check TTL status
aws dynamodb describe-time-to-live --table-name Orders

# Check Point-in-Time Recovery
aws dynamodb describe-continuous-backups --table-name Orders

# Check Streams configuration
aws dynamodb describe-table --table-name Orders \
  | jq '.Table.StreamSpecification'
```

**Query and scan examples:**
```bash
# Query with key condition (efficient)
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression 'PK = :pk AND begins_with(SK, :sk)' \
  --expression-attribute-values '{
    ":pk": {"S": "CUSTOMER#alice@example.com"},
    ":sk": {"S": "ORDER#"}
  }' \
  --consistent-read

# Get specific item
aws dynamodb get-item \
  --table-name Orders \
  --key '{"PK": {"S": "ORDER#456"}, "SK": {"S": "META"}}' \
  --consistent-read

# Query GSI
aws dynamodb query \
  --table-name Orders \
  --index-name OrderStatusIndex \
  --key-condition-expression 'GSI1PK = :status' \
  --expression-attribute-values '{":status": {"S": "STATUS#SHIPPED"}}'

# Scan with filter (expensive, avoid in production)
aws dynamodb scan \
  --table-name Orders \
  --filter-expression 'Total > :min' \
  --expression-attribute-values '{":min": {"N": "100"}}'
```

**CloudWatch Metrics:**
```bash
# Get consumed read capacity
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name ConsumedReadCapacityUnits \
  --dimensions Name=TableName,Value=Orders \
  --start-time 2026-10-13T00:00:00Z \
  --end-time 2026-10-14T00:00:00Z \
  --period 300 \
  --statistics Sum

# Get throttled requests
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name UserErrors \
  --dimensions Name=TableName,Value=Orders \
  --start-time 2026-10-13T00:00:00Z \
  --end-time 2026-10-14T00:00:00Z \
  --period 300 \
  --statistics Sum

# Check system errors
aws cloudwatch get-metric-statistics \
  --namespace AWS/DynamoDB \
  --metric-name SystemErrors \
  --dimensions Name=TableName,Value=Orders \
  --start-time 2026-10-13T00:00:00Z \
  --end-time 2026-10-14T00:00:00Z \
  --period 300 \
  --statistics Sum
```

**Test conditional writes:**
```bash
# Update with condition (optimistic locking)
aws dynamodb update-item \
  --table-name Orders \
  --key '{"PK": {"S": "PRODUCT#789"}, "SK": {"S": "INVENTORY"}}' \
  --update-expression 'SET AvailableQuantity = AvailableQuantity - :qty' \
  --condition-expression 'AvailableQuantity >= :qty' \
  --expression-attribute-values '{":qty": {"N": "5"}}' \
  --return-values ALL_NEW
```

**Calculate item size:**
```python
import sys

def calculate_item_size(item):
    """Calculate DynamoDB item size in bytes"""
    size = 0
    
    for attr_name, attr_value in item.items():
        # Attribute name
        size += len(attr_name)
        
        # Attribute value
        if 'S' in attr_value:  # String
            size += len(attr_value['S'].encode('utf-8'))
        elif 'N' in attr_value:  # Number
            size += len(attr_value['N'])
        elif 'B' in attr_value:  # Binary
            size += len(attr_value['B'])
        elif 'SS' in attr_value:  # String Set
            size += sum(len(s.encode('utf-8')) for s in attr_value['SS'])
        elif 'NS' in attr_value:  # Number Set
            size += sum(len(n) for n in attr_value['NS'])
        elif 'BS' in attr_value:  # Binary Set
            size += sum(len(b) for b in attr_value['BS'])
        elif 'M' in attr_value:  # Map
            size += calculate_item_size(attr_value['M'])
        elif 'L' in attr_value:  # List
            for elem in attr_value['L']:
                size += calculate_item_size({'item': elem})
        elif 'NULL' in attr_value:
            size += 1
        elif 'BOOL' in attr_value:
            size += 1
    
    return size

# Example usage
item = {
    'PK': {'S': 'CUSTOMER#alice@example.com'},
    'SK': {'S': 'ORDER#2026-10-13T14:20:00Z#456'},
    'Total': {'N': '125.99'}
}

size_bytes = calculate_item_size(item)
print(f"Item size: {size_bytes} bytes")
if size_bytes > 400 * 1024:
    print("WARNING: Item exceeds 400 KB limit!")
```

**Capacity calculation helpers:**
```python
import math

def calculate_rcu(item_size_kb, items_per_second, strongly_consistent=True, transactional=False):
    """Calculate required Read Capacity Units"""
    # Round up to nearest 4 KB
    units_per_read = math.ceil(item_size_kb / 4.0)
    
    # Eventually consistent reads use half the RCUs
    if not strongly_consistent:
        units_per_read = units_per_read / 2.0
    
    # Transactional reads cost 2x
    if transactional:
        units_per_read = units_per_read * 2
    
    return math.ceil(units_per_read * items_per_second)

def calculate_wcu(item_size_kb, items_per_second, transactional=False):
    """Calculate required Write Capacity Units"""
    # Round up to nearest 1 KB
    units_per_write = math.ceil(item_size_kb / 1.0)
    
    # Transactional writes cost 2x
    if transactional:
        units_per_write = units_per_write * 2
    
    return math.ceil(units_per_write * items_per_second)

# Examples
print(f"6 KB items, 100 reads/sec, strongly consistent: {calculate_rcu(6, 100)} RCUs")
print(f"6 KB items, 100 reads/sec, eventually consistent: {calculate_rcu(6, 100, strongly_consistent=False)} RCUs")
print(f"1.5 KB items, 50 writes/sec: {calculate_wcu(1.5, 50)} WCUs")
print(f"1.5 KB items, 50 transactional writes/sec: {calculate_wcu(1.5, 50, transactional=True)} WCUs")
```

**Check for hot partitions (CloudWatch Contributor Insights):**
```bash
# Create rule to find hot partition keys
aws dynamodb put-resource-policy \
  --resource-arn arn:aws:dynamodb:us-east-1:123456789012:table/Orders \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "contributorinsights.amazonaws.com"},
      "Action": "dynamodb:DescribeTable",
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Orders"
    }]
  }'

# Enable Contributor Insights
aws dynamodb update-contributor-insights \
  --table-name Orders \
  --contributor-insights-action ENABLE
```

## 10. Interview questions and answers

### Q1: DynamoDB or RDS – how do you decide?

**Decision Framework:**

| Factor | Choose DynamoDB | Choose RDS |
|--------|----------------|------------|
| **Access pattern** | Known queries via keys/indexes | Ad-hoc queries, complex joins |
| **Scale** | Massive scale (millions of req/sec) | Moderate scale (thousands of req/sec) |
| **Data model** | Key-value, document, flexible schema | Relational, structured schema |
| **Transactions** | Simple transactions (100 items max) | Complex multi-table transactions |
| **Latency** | Single-digit milliseconds | 10-100ms typical |
| **Operations** | Fully managed, serverless | Managed but need maintenance windows |
| **Cost model** | Pay per request/capacity | Pay for provisioned instance |

**Example scenarios:**

**Choose DynamoDB:**
- Session store (key-value lookups)
- Gaming leaderboards (sort by score)
- IoT sensor data (time-series)
- Shopping cart (simple get/put)
- User profiles (document store)

**Choose RDS:**
- Financial reporting (complex aggregations)
- Multi-table joins (order + customer + inventory)
- Ad-hoc analytics queries
- Strict ACID across many tables
- Legacy app requiring SQL

**Exam tip:** If the question mentions "predictable query patterns" or "known access patterns" → DynamoDB. If it mentions "complex queries" or "analytical workloads" → RDS or data warehouse.

---

### Q2: Can a filter expression lower read costs?

**Answer:** No. Filter expressions are applied **after** DynamoDB reads the data, so you're charged for all items read, not just items returned.

**How it works:**
```
1. DynamoDB reads items matching key condition (charges RCUs)
2. DynamoDB applies filter expression (no additional charge)
3. DynamoDB returns filtered results (fewer items than read)
4. You pay for ALL items read in step 1
```

**Example:**
```python
# Query reads 100 items (4 KB each) = 100 RCUs consumed
# Filter returns only 5 matching items
# Cost: 100 RCUs (not 5 RCUs)
response = table.query(
    KeyConditionExpression='PK = :pk',
    FilterExpression='Price > :min',
    ExpressionAttributeValues={':pk': 'PRODUCT', ':min': 100}
)
```

**Optimization strategies:**

1. **Model filter criteria into keys/indexes:**
```
Bad:  Query all products, filter by price range
Good: GSI with PK=CATEGORY, SK=PRICE → query price range directly
```

2. **Use sparse GSI for specific states:**
```
Bad:  Query all orders, filter Status='PENDING'
Good: GSI only indexes items with Status='PENDING' (sparse)
```

3. **Reduce item size:**
```
Bad:  Project ALL attributes to GSI
Good: Project only needed attributes
```

**Exam tip:** "How to reduce costs when querying data with specific criteria?" → Model criteria into keys/indexes, not filter expressions.

---

### Q3: How do you prevent overselling inventory in a global multi-region setup?

**Challenge:** MREC (Multi-Region Eventually Consistent) mode allows concurrent writes in different regions that both pass local conditions.

**Scenario:**
```
Inventory: 10 units
Region US: Customer A orders 8 units (condition passes: 10 >= 8)
Region EU: Customer B orders 5 units (condition passes: 10 >= 5)
Result: Both succeed locally, inventory = -3 globally
```

**Solutions:**

**Option 1: MRSC Mode (Synchronous)**
```
- Use MRSC global tables (3 regions required)
- Strong consistency guarantees global correctness
- Higher latency (cross-region coordination)
- Limitations: No TTL, no LSIs, no transactions
```

**Option 2: Primary Write Region**
```
- Designate one region for inventory writes
- Other regions are read replicas
- Route all inventory updates to primary region
- Accept higher latency for cross-region customers
```

**Option 3: Reserved Inventory**
```python
# Two-phase commit: reserve then confirm
# Phase 1: Reserve inventory
table.update_item(
    Key={'PK': 'PRODUCT#789', 'SK': 'INVENTORY'},
    UpdateExpression='SET ReservedQuantity = ReservedQuantity + :qty',
    ConditionExpression='AvailableQuantity - ReservedQuantity >= :qty',
    ExpressionAttributeValues={':qty': 5}
)

# Phase 2: Confirm order (within 10 minutes)
# If confirmed: AvailableQuantity -= qty, ReservedQuantity -= qty
# If expired: ReservedQuantity -= qty (TTL cleanup)
```

**Option 4: Accept Overselling + Compensation**
```
- Allow negative inventory temporarily
- Async reconciliation process
- Contact customers to cancel/delay orders
- Suitable for: non-critical inventory, high availability priority
```

**Exam tip:** MREC = eventual consistency, last-writer-wins. For global correctness, use MRSC or application-level coordination.

---

### Q4: Explain GSI vs LSI differences and when to use each

**Comparison:**

| Feature | GSI (Global Secondary Index) | LSI (Local Secondary Index) |
|---------|------------------------------|------------------------------|
| **Partition key** | Can differ from base table | Must be same as base table |
| **Sort key** | Optional, can differ | Must differ from base table |
| **When created** | Anytime (even after table exists) | Only at table creation time |
| **Capacity** | Separate provisioned capacity | Shares base table capacity |
| **Consistency** | Eventually consistent only | Eventually or strongly consistent |
| **Queries** | Different access pattern | Same PK, different sort order |
| **Item collection limit** | No limit | 10 GB per partition key value |
| **Use case** | Alternate queries | Alternate sort within partition |

**GSI Example:**
```
Base Table: PK=CustomerId, SK=OrderId
GSI: PK=Status, SK=OrderDate
Query: "Find all SHIPPED orders" → Use GSI
```

**LSI Example:**
```
Base Table: PK=CustomerId, SK=OrderDate
LSI: PK=CustomerId, SK=OrderTotal
Query: "Find customer's orders sorted by price" → Use LSI
```

**When to use GSI:**
- Need different partition key for alternate query pattern
- Want to query across all partitions
- Table already exists (can't add LSI)
- Don't need strong consistency

**When to use LSI:**
- Same partition key, need alternate sort order
- Need strong consistency option
- Items per partition < 10 GB
- Creating new table (can't add LSI later)

**Exam tip:** LSI is rare in practice due to 10 GB limit and creation-time-only constraint. Most scenarios use GSI.

---

### Q5: How do you handle one enormous tenant in a multi-tenant DynamoDB table?

**Problem:** One customer has 90% of traffic, causing hot partition.

**Solutions:**

**1. Write Sharding (Distribute Hot Key)**
```python
import random

# Add random suffix to partition key
tenant_id = "BIGCORP"
shard_id = random.randint(0, 9)  # 10 shards
pk = f"{tenant_id}#{shard_id}"

# Write distributed across 10 partitions
table.put_item(Item={'PK': pk, 'SK': order_id, ...})

# Read requires scatter-gather
results = []
for shard in range(10):
    pk = f"{tenant_id}#{shard}"
    response = table.query(KeyConditionExpression='PK = :pk', ...)
    results.extend(response['Items'])
```

**2. Dedicated Table for Large Tenant**
```
Small tenants: SharedTable (multi-tenant)
Large tenant:  BigCorpTable (dedicated)

Route by tenant size threshold (e.g., >10K requests/day)
```

**3. DynamoDB DAX (Caching)**
```
Cache hot tenant reads in DAX
Reduces read load on base table
Writes still go to base table
Eventual consistency acceptable
```

**4. Client-Side Caching**
```python
# Cache frequently accessed tenant data
from functools import lru_cache

@lru_cache(maxsize=1000)
def get_tenant_config(tenant_id):
    return table.get_item(Key={'PK': tenant_id})
```

**5. Increase Provisioned Capacity**
```
Not a full solution (hot partition still bottleneck)
But can help with burst traffic
Use with write sharding for best results
```

**Tradeoffs:**

| Solution | Pros | Cons |
|----------|------|------|
| Write sharding | Distributes load | Complex reads, increased latency |
| Dedicated table | Full isolation | Operational overhead, cost |
| DAX caching | Fast reads | Eventual consistency, cost |
| Client cache | Lowest latency | Stale data risk |

**Exam tip:** Hot partition question → Mention write sharding with random suffix + scatter-gather reads.

---

### Q6: Why do transactions cost 2x capacity units?

**Answer:** Transactional operations perform additional work to ensure ACID properties.

**What DynamoDB does for transactions:**

1. **Prepare Phase:**
   - Acquire locks on all items
   - Validate conditions on all items
   - Ensure no conflicts with concurrent operations

2. **Commit Phase:**
   - Apply changes atomically
   - Release locks
   - Update all indexes

3. **Additional Overhead:**
   - Conflict detection
   - Rollback capability
   - Cross-partition coordination

**Cost Example:**
```
Non-transactional write: 2 KB item = 2 WCUs
Transactional write: 2 KB item = 4 WCUs (2x cost)

Non-transactional read: 8 KB item, strongly consistent = 2 RCUs
Transactional read: 8 KB item = 4 RCUs (2x cost)
```

**When transactions are worth the cost:**

✅ **Use transactions when:**
- Need atomicity across multiple items
- Inventory decrement + order creation must both succeed
- Dual writes to main table + audit table
- Consistency more important than cost

❌ **Avoid transactions when:**
- Single item operations (use conditional writes instead)
- Can tolerate eventual consistency
- High-throughput workload (cost prohibitive)
- Items don't need atomic updates

**Exam tip:** Transactions guarantee atomicity across up to 100 items and 4 MB total. Cost is 2x normal operations.

---

### Q7: Explain DynamoDB consistency models

**Strong Consistency:**
```
Read reflects all successful writes that occurred before the read
Guaranteed to see most recent data
Higher latency (waits for replication confirmation)
Costs 2x RCUs vs eventually consistent
```

**Eventual Consistency:**
```
Read might not reflect most recent write immediately
Typically consistent within 1 second
Lower latency (reads from any replica)
Costs half the RCUs of strong consistency
Default for GSI queries
```

**Consistency Matrix:**

| Operation | Supports Strong Consistency? |
|-----------|------------------------------|
| GetItem | Yes (opt-in with ConsistentRead=true) |
| Query on base table | Yes (opt-in) |
| Query on LSI | Yes (opt-in) |
| Query on GSI | No (always eventually consistent) |
| Scan | Yes (opt-in) |
| Global tables (MREC) | No (cross-region eventual) |
| Global tables (MRSC) | Yes (cross-region strong) |

**Code Example:**
```python
# Eventually consistent read (default, cheaper)
response = table.get_item(
    Key={'PK': 'ORDER#456', 'SK': 'META'}
)

# Strongly consistent read (2x cost)
response = table.get_item(
    Key={'PK': 'ORDER#456', 'SK': 'META'},
    ConsistentRead=True
)

# GSI query (always eventually consistent)
response = table.query(
    IndexName='StatusIndex',
    KeyConditionExpression='Status = :status',
    # ConsistentRead not supported for GSI
)
```

**Exam scenarios:**

| Scenario | Consistency Needed |
|----------|-------------------|
| Display user's own recent order | Strong (user expects to see it) |
| Display product catalog | Eventual (slight staleness OK) |
| Banking transaction | Strong + transactional |
| Social media feed | Eventual (performance > freshness) |
| Inventory check before purchase | Strong (prevent overselling) |

**Exam tip:** Strong consistency costs 2x RCUs. GSIs are always eventually consistent. MRSC global tables provide global strong consistency.

---

### Q8: How does TTL work and what are its limitations?

**TTL (Time To Live) Overview:**
- Automatically deletes expired items
- No additional write capacity cost for deletions
- Asynchronous background process
- Typically completes within 48 hours of expiration

**How it works:**
```
1. Enable TTL on table, specify attribute name (e.g., "ExpiresAt")
2. Write items with ExpiresAt timestamp (Unix epoch seconds)
3. DynamoDB background process scans for expired items
4. Expired items deleted automatically (eventually)
```

**Configuration:**
```bash
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt
```

**Example Item:**
```python
import time

# Session expires in 24 hours
expiration_time = int(time.time()) + (24 * 60 * 60)

table.put_item(
    Item={
        'SessionId': 'abc123',
        'UserId': 'user456',
        'Data': {...},
        'ExpiresAt': expiration_time  # Unix epoch seconds
    }
)
```

**Key Limitations:**

❌ **Not immediate:** Items may exist up to 48 hours past expiration
❌ **Not for security:** Don't rely on TTL for access control (check expiration in app)
❌ **Not guaranteed order:** Expired items may not be deleted in expiration order
❌ **MRSC incompatible:** TTL not supported in MRSC global tables
❌ **Stream events:** TTL deletes appear in Streams (can trigger Lambdas)

**Best Practices:**

```python
# Always check expiration in application logic
def get_session(session_id):
    response = table.get_item(Key={'SessionId': session_id})
    
    if 'Item' in response:
        item = response['Item']
        if item['ExpiresAt'] < int(time.time()):
            return None  # Expired, don't use
        return item
    return None
```

**TTL Delete Events in Streams:**
```json
{
  "eventName": "REMOVE",
  "userIdentity": {
    "type": "Service",
    "principalId": "dynamodb.amazonaws.com"
  },
  "dynamodb": {
    "Keys": {...},
    "OldImage": {...}
  }
}
```

**Cost Savings Example:**
```
Without TTL: 10M items × 1 KB × $0.25/GB-month = $2,500/month
With TTL (7-day retention): ~2.3M avg items = $575/month
Savings: 77% storage cost reduction
```

**Exam tip:** TTL is eventual deletion (up to 48 hours). For immediate enforcement, check expiration in application code. TTL deletes don't consume write capacity but do appear in Streams.

---

### Q9: What is a sparse GSI and when should you use it?

**Sparse GSI:** Index that only includes items with specific attributes present.

**How it works:**
```
GSI only indexes items where the GSI key attributes exist
Items without GSI key attributes are not in the index
Dramatically reduces index size and cost
```

**Example: Track Pending Orders Only**

**Table Items:**
```json
// Active order (has PendingStatus attribute)
{
  "PK": "ORDER#123",
  "SK": "META",
  "Status": "PENDING",
  "PendingStatus": "PENDING",  // GSI key attribute
  "CreatedAt": "2026-10-13T10:00:00Z"
}

// Completed order (no PendingStatus attribute)
{
  "PK": "ORDER#124",
  "SK": "META",
  "Status": "COMPLETED",
  "CompletedAt": "2026-10-13T11:00:00Z"
  // No PendingStatus → not in GSI
}
```

**GSI Configuration:**
```json
{
  "IndexName": "PendingOrdersIndex",
  "KeySchema": [
    {"AttributeName": "PendingStatus", "KeyType": "HASH"},
    {"AttributeName": "CreatedAt", "KeyType": "RANGE"}
  ],
  "Projection": {"ProjectionType": "ALL"}
}
```

**Result:**
- Sparse GSI only contains PENDING orders
- When order completes, remove `PendingStatus` attribute → item removed from GSI
- Much cheaper than indexing all orders

**Use Cases:**

1. **Work Queues:**
```
GSI key: ProcessingStatus = "PENDING"
Query: Get all pending work items
When processed: Remove ProcessingStatus attribute
```

2. **Exception Tracking:**
```
GSI key: HasError = "true"
Query: Find all items with errors
When fixed: Remove HasError attribute
```

3. **Feature Flags:**
```
GSI key: BetaUser = "true"
Query: Get all beta users
When graduated: Remove BetaUser attribute
```

**Cost Comparison:**
```
Full GSI (1M orders, 50% pending):
- Index size: 1M items
- Storage cost: ~$0.25/GB for all items

Sparse GSI (1M orders, 50% pending):
- Index size: 500K items
- Storage cost: Half the full GSI cost
- Write amplification: Only when status changes
```

**Exam tip:** Sparse GSI = index only items with specific attribute. Efficient for querying subset of items (pending orders, error states, active sessions).

---

### Q10: How do BatchWriteItem and TransactWriteItems differ?

**Comparison:**

| Feature | BatchWriteItem | TransactWriteItems |
|---------|---------------|-------------------|
| **Atomicity** | No (items processed independently) | Yes (all-or-nothing) |
| **Item limit** | 25 items | 100 items |
| **Size limit** | 16 MB total | 4 MB total |
| **Operations** | Put, Delete only | Put, Update, Delete, ConditionCheck |
| **Conditions** | No conditions supported | Conditions on each item |
| **Cost** | Normal WCUs | 2x WCUs (transactional) |
| **Partial success** | Yes (returns unprocessed items) | No (all fail if any fails) |
| **Cross-table** | Yes | Yes |
| **Use case** | Bulk writes, retryable | Atomic multi-item updates |

**BatchWriteItem Example:**
```python
# Some items may succeed, some may fail
response = dynamodb.batch_write_item(
    RequestItems={
        'Orders': [
            {'PutRequest': {'Item': {'PK': 'ORDER#1', ...}}},
            {'PutRequest': {'Item': {'PK': 'ORDER#2', ...}}},
            {'DeleteRequest': {'Key': {'PK': 'ORDER#OLD', 'SK': 'META'}}}
        ]
    }
)

# Check for unprocessed items and retry
if response['UnprocessedItems']:
    # Retry with exponential backoff
    retry_items = response['UnprocessedItems']
```

**TransactWriteItems Example:**
```python
# All items succeed or all fail
try:
    response = dynamodb.transact_write_items(
        TransactItems=[
            {
                'Update': {
                    'TableName': 'Orders',
                    'Key': {'PK': 'INVENTORY#123', 'SK': 'STOCK'},
                    'UpdateExpression': 'SET Qty = Qty - :dec',
                    'ConditionExpression': 'Qty >= :dec',
                    'ExpressionAttributeValues': {':dec': 5}
                }
            },
            {
                'Put': {
                    'TableName': 'Orders',
                    'Item': {'PK': 'ORDER#456', 'SK': 'META', 'Qty': 5}
                }
            }
        ]
    )
    print("Transaction succeeded")
except ClientError as e:
    if e.response['Error']['Code'] == 'TransactionCanceledException':
        # All items rolled back
        print("Transaction failed - no changes made")
```

**When to use BatchWriteItem:**
- Bulk loading data
- Independent writes (order doesn't matter)
- Can handle partial failures
- Cost-sensitive (avoid 2x transaction cost)

**When to use TransactWriteItems:**
- Need atomicity (inventory + order)
- Conditional updates across items
- Consistency more important than cost
- Cannot tolerate partial success

**Exam tip:** Batch operations are NOT transactions. Use TransactWriteItems for atomic multi-item operations.

---

### Q11: Explain single-table design tradeoffs

**Single-Table Design:**
Store multiple entity types (customers, orders, products) in one table using strategic key patterns.

**Pros:**

✅ **Fewer API calls:** Get related data in one query
✅ **Transactions work:** All entities in same table
✅ **Cost optimization:** Fewer round trips
✅ **Better performance:** One network call vs many

**Cons:**

❌ **Complexity:** Steep learning curve, hard to understand
❌ **Rigid access patterns:** Must know queries upfront
❌ **Harder to change:** Adding new access pattern may require migration
❌ **Item size limits:** 400 KB for entire entity graph
❌ **Scanning difficult:** Mixed entity types complicate scans

**Example: E-Commerce Single Table**
```
PK                          SK                      EntityType
CUSTOMER#alice@example.com  PROFILE                 Customer
CUSTOMER#alice@example.com  ORDER#2026-10-13#456    CustomerOrder
ORDER#456                   META                    Order
ORDER#456                   ITEM#001                OrderItem
PRODUCT#789                 DETAILS                 Product
```

**Query Patterns:**
```python
# Get customer + all their orders (1 query)
response = table.query(
    KeyConditionExpression='PK = :pk',
    ExpressionAttributeValues={':pk': 'CUSTOMER#alice@example.com'}
)

# Get order + all its items (1 query)
response = table.query(
    KeyConditionExpression='PK = :pk',
    ExpressionAttributeValues={':pk': 'ORDER#456'}
)
```

**Multi-Table Alternative:**
```
CustomersTable: PK=CustomerId
OrdersTable: PK=OrderId, GSI on CustomerId
OrderItemsTable: PK=OrderId, SK=ItemId
ProductsTable: PK=ProductId

# Get customer orders: 2 queries
# Get order items: 2 queries
# More tables, more queries, higher latency
```

**When to use single-table:**
- Access patterns well-defined
- Need transaction across entities
- Performance critical
- Team has DynamoDB expertise

**When to use multi-table:**
- Access patterns evolving
- Team new to DynamoDB
- Need simpler data model
- Different access patterns per entity

**Exam tip:** Single-table design is NOT mandatory. It's an optimization for known access patterns and transactions. Multi-table is simpler and more flexible.

---

### Q12: How do global tables handle conflicts in MREC mode?

**MREC (Multi-Region Eventually Consistent) Conflict Resolution:**

**Last Writer Wins (LWW):**
- DynamoDB uses timestamps to resolve conflicts
- Most recent write wins (by DynamoDB internal timestamp, not your attribute)
- Concurrent writes to same item may result in lost updates

**Scenario:**
```
Time    Region US-EAST-1           Region EU-WEST-1
T0      Item: {Stock: 100}          Item: {Stock: 100}
T1      Write: Stock = 95           
T2                                  Write: Stock = 90
T3      Replication: Stock = 90     Replication: Stock = 95  ← conflict!
T4      Final: Stock = 90           Final: Stock = 90        ← LWW

Result: US write at T1 is lost (EU write at T2 won)
```

**Conflict Indicators:**
```python
# Check if item was updated by replication
response = table.get_item(
    Key={'PK': 'PRODUCT#789', 'SK': 'INVENTORY'},
    ReturnConsumedCapacity='TOTAL'
)

# aws:rep:updatetime attribute shows last replication timestamp
# aws:rep:updateregion shows which region made the update
```

**Strategies to Handle Conflicts:**

**1. Avoid Conflicts (Regional Writes):**
```
- US customers write to US region only
- EU customers write to EU region only
- Each region "owns" subset of data
```

**2. Conflict-Free Data Types:**
```python
# Instead of: SET Stock = 95
# Use: SET Stock = Stock - 5  (commutative operation)

# Better: Track additions/subtractions separately
{
  "Stock": 100,
  "Sold": 5,      # Increment only
  "Restocked": 3  # Increment only
}
# Computed: Available = Stock + Restocked - Sold
```

**3. Version Vectors (Application-Level):**
```python
{
  "Stock": 95,
  "Versions": {
    "us-east-1": 3,
    "eu-west-1": 2
  }
}
# Application merges conflicts using vector clocks
```

**4. Use MRSC Mode:**
```
- Synchronous cross-region replication
- Strong consistency globally
- No conflicts (coordination prevents them)
- Higher latency, feature limitations
```

**Exam tip:** MREC uses last-writer-wins for conflicts. For global correctness without conflicts, use MRSC mode (3 regions) or coordinate writes to primary region.

---

## Quick Reference Card

### Capacity Calculation Formulas

**Read Capacity Units (RCUs):**
```
Item Size Unit = 4 KB (round up)
Eventually Consistent Read = 0.5 RCU per item
Strongly Consistent Read = 1 RCU per item
Transactional Read = 2 RCUs per item

Example: 100 reads/sec of 6 KB items (strongly consistent)
= 100 × ceil(6/4) × 1 = 100 × 2 = 200 RCUs
```

**Write Capacity Units (WCUs):**
```
Item Size Unit = 1 KB (round up)
Standard Write = 1 WCU per item
Transactional Write = 2 WCUs per item

Example: 50 writes/sec of 1.5 KB items
= 50 × ceil(1.5/1) × 1 = 50 × 2 = 100 WCUs
```

### Key Design Decision Matrix

| Access Pattern | Key Design |
|----------------|------------|
| **Get single item** | PK = unique ID |
| **Get items for entity** | PK = entity ID, SK = item ID |
| **Range query (time)** | PK = entity, SK = timestamp |
| **Sort by attribute** | PK = category, SK = attribute value |
| **Filter by status** | GSI with PK = status, SK = timestamp |
| **Multiple access patterns** | GSI with alternate keys |
| **Work queue** | Sparse GSI (only pending items) |

### GSI vs LSI Quick Reference

|  | GSI | LSI |
|--|-----|-----|
| **PK** | Different OK | Same as table |
| **SK** | Different OK | Different required |
| **Create** | Anytime | Table creation only |
| **Capacity** | Separate | Shared |
| **Consistency** | Eventual only | Eventual or strong |
| **Limit** | 20 per table | 5 per table |
| **Item collection** | No limit | 10 GB per PK |

### Operation Limits

| Resource | Limit |
|----------|-------|
| **Item size** | 400 KB (including attribute names) |
| **Partition key** | 2,048 bytes |
| **Sort key** | 1,024 bytes |
| **Transaction items** | 100 items, 4 MB total |
| **BatchGetItem** | 100 items, 16 MB |
| **BatchWriteItem** | 25 items, 16 MB |
| **Query/Scan result** | 1 MB per page |
| **GSI per table** | 20 |
| **LSI per table** | 5 |
| **Projected attributes (GSI)** | 100 |

### Common CLI Commands

```bash
# Create table with GSI
aws dynamodb create-table \
  --table-name Orders \
  --attribute-definitions \
    AttributeName=PK,AttributeType=S \
    AttributeName=SK,AttributeType=S \
    AttributeName=GSI1PK,AttributeType=S \
  --key-schema \
    AttributeName=PK,KeyType=HASH \
    AttributeName=SK,KeyType=RANGE \
  --global-secondary-indexes \
    "[{\"IndexName\":\"StatusIndex\",\"KeySchema\":[{\"AttributeName\":\"GSI1PK\",\"KeyType\":\"HASH\"}],\"Projection\":{\"ProjectionType\":\"ALL\"}}]" \
  --billing-mode PAY_PER_REQUEST

# Put item
aws dynamodb put-item \
  --table-name Orders \
  --item '{"PK":{"S":"ORDER#123"},"SK":{"S":"META"},"Total":{"N":"99.99"}}'

# Get item (strong consistency)
aws dynamodb get-item \
  --table-name Orders \
  --key '{"PK":{"S":"ORDER#123"},"SK":{"S":"META"}}' \
  --consistent-read

# Query with key condition
aws dynamodb query \
  --table-name Orders \
  --key-condition-expression 'PK = :pk AND begins_with(SK, :sk)' \
  --expression-attribute-values '{":pk":{"S":"CUSTOMER#alice"},":sk":{"S":"ORDER#"}}'

# Conditional update (optimistic locking)
aws dynamodb update-item \
  --table-name Orders \
  --key '{"PK":{"S":"PRODUCT#789"},"SK":{"S":"INVENTORY"}}' \
  --update-expression 'SET Stock = Stock - :dec' \
  --condition-expression 'Stock >= :dec' \
  --expression-attribute-values '{":dec":{"N":"5"}}'

# Enable TTL
aws dynamodb update-time-to-live \
  --table-name Sessions \
  --time-to-live-specification Enabled=true,AttributeName=ExpiresAt

# Enable Point-in-Time Recovery
aws dynamodb update-continuous-backups \
  --table-name Orders \
  --point-in-time-recovery-specification PointInTimeRecoveryEnabled=true

# Enable Streams
aws dynamodb update-table \
  --table-name Orders \
  --stream-specification StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

### Cost Optimization Checklist

✅ **Use on-demand for variable workloads** (avoid over-provisioning)
✅ **Project only needed attributes to GSI** (reduce storage + write costs)
✅ **Sparse GSI for subset queries** (index only relevant items)
✅ **TTL for auto-expiration** (reduce storage costs)
✅ **Eventually consistent reads** (half the RCU cost)
✅ **Batch operations** (reduce API call overhead)
✅ **Model access patterns into keys** (avoid scans/filters)
✅ **DAX for read-heavy workloads** (cache hot data)
✅ **Reserved capacity for predictable load** (up to 76% savings)

---

## Exam Tips Summary

### Must-Know Concepts (SAA-C03)

1. **Key Design = Query Design** → Model access patterns into partition/sort keys, not filter expressions

2. **Query vs Scan** → Query uses keys (efficient), Scan reads all data (expensive)

3. **GSI Always Eventual** → GSIs are eventually consistent; use base table/LSI for strong reads

4. **Filter != Cost Savings** → Filters applied after read; charged for all items scanned

5. **Transactions Cost 2x** → Transactional operations cost double capacity units

6. **Strong Consistency Costs 2x** → Strong reads cost double RCUs vs eventual

7. **Conditional Writes** → Prevent race conditions with condition expressions (optimistic locking)

8. **Batch ≠ Transaction** → BatchWriteItem is NOT atomic; use TransactWriteItems for atomicity

9. **Hot Partition** → Single key with high traffic; solution = write sharding with suffixes

10. **Sparse GSI** → Index only items with specific attribute; efficient for subset queries

11. **TTL Eventual** → Deletions within 48 hours; check expiration in app for immediate enforcement

12. **Global Tables Modes** → MREC (eventual, last-writer-wins) vs MRSC (strong, 3 regions)

13. **LSI Creation Time** → LSIs can only be created with table; GSIs can be added anytime

14. **Item Size Limit** → 400 KB max including attribute names and values

15. **Streams = At Least Once** → Stream records may duplicate; make consumers idempotent

### Common Exam Scenarios

| Scenario | Solution |
|----------|----------|
| **Prevent overselling inventory** | Conditional write with `Stock >= :qty` condition |
| **High read costs from queries** | Model filter criteria into GSI keys, not FilterExpression |
| **Need cross-region writes** | DynamoDB global tables (MREC or MRSC) |
| **Track only pending orders** | Sparse GSI with PendingStatus attribute |
| **Hot partition (one tenant dominates)** | Write sharding with random suffix |
| **Need strong consistency cross-region** | MRSC global tables (3 regions required) |
| **Atomic multi-item update** | TransactWriteItems (max 100 items, 4 MB) |
| **Auto-delete expired sessions** | TTL with ExpiresAt attribute (Unix epoch) |
| **Process table changes** | DynamoDB Streams + Lambda trigger |
| **Alternate query pattern** | Add GSI with different partition/sort keys |
| **Complex analytics queries** | Export to S3 + Athena, or use DynamoDB streams → Redshift |
| **Single-digit millisecond reads** | DynamoDB DAX (in-memory cache) |
| **Prevent lost updates** | Conditional write with version attribute |
| **Cost optimization** | On-demand billing + sparse GSI + TTL + eventual reads |
| **Time-series data** | PK = device ID, SK = timestamp |

### Red Flags (Anti-Patterns)

❌ **Using Scan for production queries** → Model queries into keys/indexes

❌ **FilterExpression for cost optimization** → Filters applied after read, full cost charged

❌ **BatchWriteItem for atomic updates** → Use TransactWriteItems for atomicity

❌ **Relying on TTL for immediate deletion** → TTL is eventual (48 hours); check expiration in app

❌ **GSI for strong consistency** → GSIs are always eventually consistent

❌ **Single hot partition key** → Use write sharding or dedicated table

❌ **Storing large objects in DynamoDB** → Store in S3, keep reference in DynamoDB

❌ **Expecting ACID across regions (MREC)** → Use MRSC or coordinate writes to primary region

❌ **Creating LSI after table exists** → LSIs must be created with table

❌ **Using DynamoDB like SQL database** → No joins, no complex queries; key-value model

❌ **Not handling unprocessed items** → BatchGetItem/BatchWriteItem may return partial results

❌ **Strong reads on every query** → Use eventually consistent when staleness acceptable (2x cost)

❌ **Transactions for single-item operations** → Use conditional writes (cheaper)

❌ **Ignoring item size limits** → 400 KB max; split large items or use S3

❌ **All attributes in GSI projection** → Project only needed attributes to reduce cost

---

## 11. Quick revision

- Design queries before keys and indexes.
- Query ≠ Scan; filters do not erase read cost.
- GSI eventual; table/LSI can be strong.
- Conditional writes prevent stale read-modify-write races.
- Batch APIs are not transactions.
- MREC and MRSC have different guarantees and limitations.

## 13. AWS sources

- [Secondary indexes](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/SecondaryIndexes.html)
- [Read/write capacity](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/read-write-operations.html)
- [Read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html)
- [Constraints](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Constraints.html)
- [Global tables and mode restrictions](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/V2globaltables_HowItWorks.html)
- [September 2026 MRSC expansion](https://aws.amazon.com/about-aws/whats-new/2026/09/dynamodb-mrsc-additional-regions/)

**Related:** [RDS](aws-rds-cheatsheet.md), [Aurora](aws-aurora-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md).
