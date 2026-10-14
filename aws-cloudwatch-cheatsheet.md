# Amazon CloudWatch Cheat Sheet

**Interview preparation: metrics, logs, alarms, traces, incident diagnosis, and monitoring cost.**

**Verified:** 2026-10-13. Availability and pricing depend on feature, Region, and telemetry volume.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. CloudWatch versus adjacent services](#2-cloudwatch-versus-adjacent-services)
- [3. Metrics: identity, resolution, and retention](#3-metrics-identity-resolution-and-retention)
- [4. Alarm design](#4-alarm-design)
- [5. Logs and queries](#5-logs-and-queries)
- [6. Collecting telemetry](#6-collecting-telemetry)
- [7. Current change: Database Insights](#7-current-change-database-insights)
- [8. A practical incident workflow](#8-a-practical-incident-workflow)
- [9. Cost and operational traps](#9-cost-and-operational-traps)
- [10. Common patterns](#10-common-patterns)
- [11. Troubleshooting and practical inspection](#11-troubleshooting-and-practical-inspection)
- [12. Quick reference card](#12-quick-reference-card)
- [13. Exam tips summary](#13-exam-tips-summary)
- [14. Interview Q&A](#14-interview-qa)
- [15. Quick revision](#15-quick-revision)
- [16. AWS sources](#16-aws-sources)

---

## 1. Fundamentals and mental model

CloudWatch collects and analyzes operational telemetry and supports alarms and automation. Good observability connects symptoms seen by users to the components that caused them.

```text
Services/applications → metrics → alarms → notification / controlled action
                      → logs → queries / metric filters / subscriptions
                      → traces → request path and dependency diagnosis
```

| Signal | Answers | Example |
|---|---|---|
| Metrics | How much, how often, how long? | Error rate, p95 latency, queue age |
| Logs | What happened for this event? | Structured failure with request ID |
| Traces | Where did this request spend time? | API → Lambda → database |
| Events | What changed? | Deployment, instance replacement, failover |

## 2. CloudWatch versus adjacent services

| Service | Primary role |
|---|---|
| CloudWatch | Operational telemetry and alarms |
| CloudTrail | Audit AWS API/account activity |
| AWS Config | Resource configuration history and compliance evaluation |
| EventBridge | Event routing and invocation workflows |
| X-Ray / Application Signals integrations | Distributed request/application performance analysis |

CloudTrail events can feed an operational alerting pipeline, but CPU utilization is not an API audit event. “CloudWatch Events” in older notes maps to the EventBridge lineage; say which capability you mean.

## 3. Metrics: identity, resolution, and retention

A metric is identified by namespace, name, and dimensions. Different dimension combinations are different time series. Adding `requestId` as a dimension can create explosive cardinality and cost.

| Original/retrieved granularity | Retention |
|---|---|
| Less than 60 seconds | 3 hours |
| 1 minute | 15 days |
| 5 minutes | 63 days |
| 1 hour | 455 days (15 months) |

CloudWatch rolls up older points; 15-month retention does not mean one-second detail for 15 months. Metric publication resolution and alarm evaluation period are separate choices. [Metric concepts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html)

Use `Sum` for counts over a period, appropriate averages for utilization, and percentiles for tail latency where supported. Do not sum averages or average percentiles and call that the fleet percentile.

## 4. Alarm design

Alarms have `OK`, `ALARM`, and `INSUFFICIENT_DATA` states. Configure threshold, statistic, period, evaluation periods, datapoints-to-alarm, and missing-data treatment.

| Choice | Example reasoning |
|---|---|
| M out of N | Tolerate a brief spike while catching sustained errors |
| Missing as breaching | Useful for an expected heartbeat |
| Missing as not breaching | Can suit sparse error-only metrics |
| Missing/ignore | Useful where absence should preserve uncertainty/state |
| Composite alarm | Correlate symptoms and reduce duplicate paging |
| Anomaly detection | Detect deviations from a changing baseline |

An alarm on `Errors > 0` can be noisy at scale. Consider error rate plus a minimum traffic condition, latency SLOs, and sustained queue age. Notification paths and runbooks need testing too. [CloudWatch alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html)

## 5. Logs and queries

Organize log groups by application/environment with deliberate retention, encryption, and access. Log streams separate sources. Avoid recording secrets, tokens, and unnecessary personal data.

Use structured fields such as service, level, event type, request ID, trace ID, duration, and outcome. Keep high-cardinality investigation identifiers in logs rather than unbounded metric dimensions.

Example **Logs Insights QL** query, assuming JSON logs with a `level` field:

```text
fields @timestamp, @message, requestId
| filter level = "ERROR"
| sort @timestamp desc
| limit 50
```

Metric filters turn matching new log events into metrics. Subscription filters deliver logs to downstream consumers. They are different tools with different costs and failure modes. Narrow log groups and time windows to control query scans. [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)

## 6. Collecting telemetry

- EC2 basic platform metrics do not automatically expose guest memory and filesystem usage. Use an agent/custom collection.
- Container Insights provides container-oriented views; instrument the application for business outcomes too.
- Lambda metrics identify invocations/errors/duration/throttling, while application logs explain the error.
- Application Signals and tracing connect service/dependency performance; sampling means traces are not a complete audit ledger.
- Synthetics can exercise critical user journeys even when real traffic is low.

Prefer an explicit telemetry contract: units, dimensions, labels, retention, sampling, owner, and the action an alert should cause. [CloudWatch features](https://aws.amazon.com/cloudwatch/features/)

## 7. Current change: Database Insights

AWS documents the Performance Insights transition with an end-of-life date of **July 31, 2026**. Use **CloudWatch Database Insights** when describing the current database monitoring experience.

Standard and Advanced modes differ in features and retention options. Verify engine/Region support rather than assuming every database exposes identical query-plan or SQL analysis features. A fleet view, database waits, logs, and application context can help distinguish slow SQL from connection or application problems. [Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)

## 8. A practical incident workflow

1. Confirm user impact: errors, latency, availability, affected tenants/Regions.
2. Compare to recent deployments and configuration changes.
3. Identify the failing service/dependency with metrics and traces.
4. Inspect correlated structured logs for representative failed requests.
5. Mitigate using a tested rollback, scaling, traffic shift, or dependency protection action.
6. Confirm recovery with user-facing signals, then investigate root cause.

Example: API p95 rises while Lambda duration and DB wait time rise but CPU stays low. Investigate database locks/I/O and connection pooling; adding function concurrency may worsen pressure.

## 9. Cost and operational traps

Major drivers include custom metric cardinality, log ingestion/storage, queries scanning large ranges, detailed monitoring, alarms, and enhanced feature usage.

Use retention policies, targeted sampling, bounded dimensions, appropriate log classes, and query discipline. Do not remove the evidence needed to investigate incidents merely to lower ingestion cost.

Dashboard presence is not alert coverage. An alarm with disabled actions, a wrong Region/dimension, or an unconfirmed notification subscription can silently fail operationally.

---

## 10. Common Patterns

### Pattern 1: Multi-Region Health Dashboard with Synthetics
```bash
# Create canary to test critical endpoint
aws synthetics create-canary \
  --name api-health-check \
  --artifact-s3-location s3://my-bucket/canary-artifacts/ \
  --execution-role-arn arn:aws:iam::account:role/CloudWatchSyntheticsRole \
  --schedule Expression="rate(5 minutes)" \
  --runtime-version syn-nodejs-puppeteer-6.2 \
  --code Handler=pageLoadBlueprint.handler,ZipFile=fileb://canary.zip

# Create alarm on canary failure
aws cloudwatch put-metric-alarm \
  --alarm-name api-canary-failure \
  --alarm-description "API health check failing" \
  --metric-name SuccessPercent \
  --namespace CloudWatchSynthetics \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 90 \
  --comparison-operator LessThanThreshold \
  --dimensions Name=CanaryName,Value=api-health-check \
  --alarm-actions arn:aws:sns:us-east-1:account:critical-alerts
```

**Architecture:**
```
[Synthetics Canary (us-east-1)] → [API Endpoint]
[Synthetics Canary (eu-west-1)] → [API Endpoint]
           |                            |
     [CloudWatch Metrics] → [Composite Alarm] → [SNS → PagerDuty]
```

---

### Pattern 2: Application Performance Monitoring with Embedded Metrics
```python
# AWS Lambda with embedded metric format (EMF)
import json
from datetime import datetime

def lambda_handler(event, context):
    start_time = datetime.now()
    
    try:
        # Process request
        result = process_order(event)
        
        # Emit custom metrics using EMF
        metrics = {
            "_aws": {
                "Timestamp": int(datetime.now().timestamp() * 1000),
                "CloudWatchMetrics": [{
                    "Namespace": "OrderService",
                    "Dimensions": [["Environment", "Operation"]],
                    "Metrics": [
                        {"Name": "ProcessingTime", "Unit": "Milliseconds"},
                        {"Name": "OrdersProcessed", "Unit": "Count"}
                    ]
                }]
            },
            "Environment": "production",
            "Operation": "CreateOrder",
            "ProcessingTime": (datetime.now() - start_time).total_seconds() * 1000,
            "OrdersProcessed": 1,
            "OrderId": result['order_id'],
            "CustomerId": event['customer_id']
        }
        
        # Print to stdout - CloudWatch Logs automatically parses EMF
        print(json.dumps(metrics))
        
        return {'statusCode': 200, 'body': json.dumps(result)}
        
    except Exception as e:
        # Emit error metric
        error_metrics = {
            "_aws": {
                "Timestamp": int(datetime.now().timestamp() * 1000),
                "CloudWatchMetrics": [{
                    "Namespace": "OrderService",
                    "Metrics": [{"Name": "Errors", "Unit": "Count"}]
                }]
            },
            "Errors": 1,
            "ErrorType": type(e).__name__
        }
        print(json.dumps(error_metrics))
        raise
```

**Benefit:** Zero-overhead custom metrics (no PutMetricData API calls), atomic logs + metrics

---

### Pattern 3: Composite Alarms for Reduced Alert Noise
```bash
# Create individual alarms
aws cloudwatch put-metric-alarm \
  --alarm-name high-error-rate \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold

aws cloudwatch put-metric-alarm \
  --alarm-name high-latency \
  --metric-name Duration \
  --namespace AWS/Lambda \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 5000 \
  --comparison-operator GreaterThanThreshold

# Create composite alarm (only alerts when BOTH are in ALARM)
aws cloudwatch put-composite-alarm \
  --alarm-name service-degraded \
  --alarm-description "Service experiencing both errors and latency" \
  --alarm-rule "ALARM(high-error-rate) AND ALARM(high-latency)" \
  --alarm-actions arn:aws:sns:us-east-1:account:pager-duty \
  --actions-enabled
```

**Use Case:** Reduce false positives - only page when multiple correlated signals indicate real degradation

---

### Pattern 4: Log Aggregation with Subscription Filters
```bash
# Create centralized log aggregation
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/orders-service \
  --filter-name errors-to-elasticsearch \
  --filter-pattern "[level=ERROR, ...]" \
  --destination-arn arn:aws:lambda:us-east-1:account:function:log-to-elasticsearch

# Alternative: Stream to Kinesis for real-time processing
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/orders-service \
  --filter-name all-logs-to-kinesis \
  --filter-pattern "" \
  --destination-arn arn:aws:kinesis:us-east-1:account:stream/centralized-logs
```

**Lambda handler for log processing:**
```python
import json
import gzip
import base64

def lambda_handler(event, context):
    # Decompress CloudWatch Logs payload
    compressed_payload = base64.b64decode(event['awslogs']['data'])
    uncompressed_payload = gzip.decompress(compressed_payload)
    log_data = json.loads(uncompressed_payload)
    
    # Process each log event
    for log_event in log_data['logEvents']:
        message = json.loads(log_event['message'])
        
        # Send to Elasticsearch, S3, or other destination
        if message.get('level') == 'ERROR':
            index_to_elasticsearch(message)
```

---

### Pattern 5: Anomaly Detection for Dynamic Thresholds
```bash
# Create anomaly detection alarm (learns baseline from historical data)
aws cloudwatch put-metric-alarm \
  --alarm-name api-requests-anomaly \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --evaluation-periods 2 \
  --threshold-metric-id e1 \
  --alarm-actions arn:aws:sns:us-east-1:account:alerts \
  --metrics file://anomaly-alarm.json

# anomaly-alarm.json
{
  "Metrics": [
    {
      "Id": "m1",
      "ReturnData": true,
      "MetricStat": {
        "Metric": {
          "Namespace": "AWS/ApiGateway",
          "MetricName": "Count",
          "Dimensions": [{"Name": "ApiName", "Value": "OrdersAPI"}]
        },
        "Period": 300,
        "Stat": "Sum"
      }
    },
    {
      "Id": "e1",
      "Expression": "ANOMALY_DETECTION_BAND(m1, 2)",
      "Label": "Count (expected)"
    }
  ]
}
```

**Use Case:** Detect traffic drops (DDoS, outage) or spikes without manual threshold tuning

---

### Pattern 6: Cross-Account Log Aggregation
```bash
# In each application account, create subscription filter to central account
aws logs put-subscription-filter \
  --log-group-name /aws/lambda/app-service \
  --filter-name to-central-logging \
  --filter-pattern "" \
  --destination-arn arn:aws:logs:us-east-1:CENTRAL-ACCOUNT:destination:CentralLogsDestination

# In central account, create destination
aws logs put-destination \
  --destination-name CentralLogsDestination \
  --target-arn arn:aws:kinesis:us-east-1:CENTRAL-ACCOUNT:stream/central-logs \
  --role-arn arn:aws:iam::CENTRAL-ACCOUNT:role/CloudWatchLogsRole

# Set destination policy to allow source accounts
aws logs put-destination-policy \
  --destination-name CentralLogsDestination \
  --access-policy '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"AWS": ["123456789012", "234567890123"]},
      "Action": "logs:PutSubscriptionFilter",
      "Resource": "arn:aws:logs:us-east-1:CENTRAL-ACCOUNT:destination:CentralLogsDestination"
    }]
  }'
```

**Architecture:**
```
[App Account 1] → [Subscription] ┐
[App Account 2] → [Subscription] ├→ [Central Account Kinesis] → [S3 + Athena]
[App Account N] → [Subscription] ┘
```

---

### Pattern 7: Cost-Optimized Logs with Infrequent Access
```python
# Automate log class management based on query patterns
import boto3
from datetime import datetime, timedelta

logs = boto3.client('logs')

def optimize_log_classes():
    # Get all log groups
    paginator = logs.get_paginator('describe_log_groups')
    
    for page in paginator.paginate():
        for log_group in page['logGroups']:
            log_group_name = log_group['logGroupName']
            
            # Check query frequency (last 30 days)
            queries = logs.describe_queries(
                logGroupName=log_group_name,
                status='Complete',
                maxResults=1000
            )
            
            recent_queries = [q for q in queries['queries'] 
                             if q['creationTime'] > (datetime.now() - timedelta(days=30)).timestamp() * 1000]
            
            # If rarely queried, switch to Infrequent Access (50% cost savings)
            if len(recent_queries) < 5 and log_group.get('logGroupClass') != 'INFREQUENT_ACCESS':
                print(f"Switching {log_group_name} to Infrequent Access")
                logs.put_log_group_class(
                    logGroupName=log_group_name,
                    logGroupClass='INFREQUENT_ACCESS'
                )
```

**Cost Impact:** Infrequent Access = 50% lower ingestion, same query cost (use for compliance/audit logs)

---

## 11. Troubleshooting and practical inspection

| Symptom | Check |
|---|---|
| No metric points | Region, namespace, exact dimensions, timestamp, permissions |
| `INSUFFICIENT_DATA` | Publication cadence, time range, wrong identity, missing-data policy |
| Alarm never notifies | State transition, actions enabled, SNS subscription/permissions |
| EC2 memory absent | Agent installed/configured and permitted to publish |
| Logs absent | Log driver/agent, IAM, network, log group/stream, retention |
| Bill spikes | Cardinality growth, noisy logs, repeated broad queries |

Read-only metadata examples:

```bash
aws cloudwatch describe-alarms --alarm-names orders-high-error-rate
aws cloudwatch describe-alarm-history --alarm-name orders-high-error-rate
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/orders
```

---

## 12. Quick Reference Card

### CloudWatch Components Comparison
| Component | Purpose | Cost Driver | Retention |
|-----------|---------|-------------|-----------|
| **Metrics** | Time-series data points | Custom metrics, dimensions, API calls | 15 days (1-min), 15 months (1-hour) |
| **Logs** | Event records | Ingestion GB, storage GB-month, queries | Configurable (1 day - never expire) |
| **Alarms** | Threshold monitoring | Per alarm-metric combination | N/A (state only) |
| **Dashboards** | Visualization | Per dashboard-month | N/A |
| **Synthetics** | Proactive monitoring | Per canary run | 31 days |
| **Application Signals** | Service maps, SLOs | Trace ingestion, metric publishing | Varies by signal |

---

### Metric Statistics Selection Guide
```
┌────────────────────────────────────────────────────┐
│ Metric Type          Recommended Statistic         │
├────────────────────────────────────────────────────┤
│ CPU/Memory Usage     Average (or p95 for peaks)    │
│ Request Count        Sum                            │
│ Error Count          Sum                            │
│ Error Rate           Average (errors/total)         │
│ Latency              p50, p90, p95, p99             │
│ Queue Depth          Average or Maximum             │
│ Active Connections   Average or Maximum             │
│ Throughput (bytes)   Sum                            │
└────────────────────────────────────────────────────┘
```

**Anti-patterns:**
- ❌ Averaging percentiles across instances
- ❌ Summing averages to get fleet total
- ❌ Using Minimum for error detection (always 0 if any success)

---

### Alarm Evaluation Decision Matrix
| Scenario | Evaluation Periods | Datapoints to Alarm | Missing Data |
|----------|-------------------|---------------------|--------------|
| Sustained high load | 5 of 5 | Tolerate no gaps | Treat as breaching |
| Spike tolerance | 3 of 5 | Allow 2 breaches | Treat as not breaching |
| Heartbeat monitoring | 1 of 1 | Immediate alert | Treat as breaching |
| Sparse error metric | 2 of 3 | Recent errors only | Ignore (maintain state) |

---

### Log Insights Query Patterns
```text
# Find errors in last hour
fields @timestamp, @message, @requestId
| filter @message like /ERROR/
| sort @timestamp desc
| limit 100

# Calculate p95 latency
fields @timestamp, duration
| stats percentile(duration, 95) as p95_latency by bin(5m)

# Count errors by type
fields @timestamp, errorType
| filter level = "ERROR"
| stats count() by errorType
| sort count desc

# Find slow requests (>1s)
filter duration > 1000
| fields @timestamp, @requestId, duration, endpoint
| sort duration desc

# Correlate logs with trace ID
fields @timestamp, @message, traceId
| filter traceId = "1-67891234-5678901234567890abcd"
| sort @timestamp asc
```

---

### Cost Optimization Quick Wins
```python
# 1. Set retention policies (reduce storage costs)
import boto3
logs = boto3.client('logs')

for log_group in logs.describe_log_groups()['logGroups']:
    if 'dev' in log_group['logGroupName']:
        logs.put_retention_policy(
            logGroupName=log_group['logGroupName'],
            retentionInDays=7  # vs default "never expire"
        )

# 2. Use metric filters instead of custom PutMetricData (free)
logs.put_metric_filter(
    logGroupName='/aws/lambda/orders',
    filterName='error-count',
    filterPattern='[level=ERROR]',
    metricTransformations=[{
        'metricName': 'ErrorCount',
        'metricNamespace': 'OrderService',
        'metricValue': '1',
        'defaultValue': 0
    }]
)

# 3. Sample high-volume metrics (use Embedded Metric Format with sampling)
import random
if random.random() < 0.1:  # 10% sampling
    emit_detailed_metrics()
```

**Monthly Cost Estimates:**
- Custom metric: $0.30/metric/month (first 10k metrics)
- Log ingestion: $0.50/GB
- Log storage (Standard): $0.03/GB-month
- Log storage (Infrequent Access): $0.015/GB-month
- Alarm: $0.10/alarm/month (standard), $0.30 (high-resolution)
- Synthetics canary: $0.0012/run

---

### CloudWatch Agent Configuration
```json
{
  "metrics": {
    "namespace": "CustomApp",
    "metrics_collected": {
      "cpu": {
        "measurement": [{"name": "cpu_usage_idle", "rename": "CPU_IDLE", "unit": "Percent"}],
        "totalcpu": false
      },
      "mem": {
        "measurement": [{"name": "mem_used_percent", "rename": "MEM_USED", "unit": "Percent"}]
      },
      "disk": {
        "measurement": [{"name": "disk_used_percent", "rename": "DISK_USED", "unit": "Percent"}],
        "resources": ["/"]
      }
    },
    "aggregation_dimensions": [["InstanceId"], ["InstanceId", "InstanceType"]]
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [{
          "file_path": "/var/log/app/*.log",
          "log_group_name": "/aws/ec2/app",
          "log_stream_name": "{instance_id}/{hostname}",
          "timezone": "UTC"
        }]
      }
    }
  }
}
```

---

## 13. Exam Tips Summary

### Key Concepts for AWS Exams

1. **Metric Resolution and Retention**
   - Standard resolution: 1 minute (free for AWS services)
   - High resolution: 1 second (custom metrics only, higher cost)
   - Retention: 3 hours (sub-minute) → 15 days (1-min) → 15 months (1-hour aggregates)

2. **Alarm States**
   - **OK:** Metric within threshold
   - **ALARM:** Metric breached threshold
   - **INSUFFICIENT_DATA:** Not enough data or alarm just created

3. **Missing Data Treatment**
   - **breaching:** Treat as bad (use for heartbeats)
   - **notBreaching:** Treat as good (use for sparse error metrics)
   - **ignore:** Maintain current state
   - **missing:** Specific to evaluation logic

4. **Logs Insights vs CloudWatch Logs**
   - **Logs Insights:** Interactive query interface (pay per GB scanned)
   - **Metric Filters:** Real-time metric extraction from logs (free)
   - **Subscription Filters:** Stream logs to Lambda/Kinesis/Firehose (free, destination costs apply)

5. **Embedded Metric Format (EMF)**
   - Log structured JSON that CloudWatch auto-converts to metrics
   - No PutMetricData API calls needed (cost-effective)
   - Atomic logs + metrics
   - Lambda automatically parses stdout for EMF

6. **CloudWatch Agent**
   - Required for EC2 memory/disk metrics (not available by default)
   - Collects logs and custom metrics
   - SSM Parameter Store for centralized configuration

7. **Synthetics Canaries**
   - Scheduled scripts that monitor endpoints/UIs
   - Built on Chrome/Chromium (Puppeteer)
   - Use for: uptime monitoring, broken links, checkout flows, API health
   - Runs in AWS-managed environment (no infrastructure)

8. **Composite Alarms**
   - Combine multiple alarms with AND/OR logic
   - Reduce alert fatigue by correlating symptoms
   - Example: Alert only if errors AND latency both high

9. **Anomaly Detection**
   - ML-powered baselines (learns from 2+ weeks history)
   - Auto-adjusts for daily/weekly patterns
   - Use when normal behavior varies (traffic patterns, sales cycles)

10. **Database Insights (formerly Performance Insights)**
    - Transitioned July 2026
    - Standard (7 days free) vs Advanced (longer retention, paid)
    - Shows top SQL, wait events, database load

11. **Cross-Account Monitoring**
    - Use IAM roles + resource policies
    - Central monitoring account can view metrics/logs from application accounts
    - CloudWatch cross-account observability: automatic dashboards

12. **Metric Math**
    - Combine metrics with expressions: `m1/m2*100` (error rate %)
    - ANOMALY_DETECTION_BAND(m1, 2) for dynamic thresholds
    - FILL(m1, 0) to handle missing data
    - Useful for SLIs: (total_requests - errors) / total_requests * 100

13. **Log Classes**
    - **Standard:** All features, frequent queries
    - **Infrequent Access:** 50% cheaper ingestion, same query cost
    - Switch to IA for compliance/audit logs queried < monthly

14. **CloudWatch vs CloudTrail vs Config**
    - **CloudWatch:** Operational metrics, logs, alarms (how is the system performing?)
    - **CloudTrail:** API audit trail (who did what, when?)
    - **Config:** Resource configuration history (is it compliant?)

15. **Alarm Actions**
    - SNS notification
    - Auto Scaling action (add/remove capacity)
    - EC2 action (stop, terminate, reboot, recover)
    - Systems Manager action (run automation)

---

### Scenario-Based Exam Questions

**Scenario 1:** Need to monitor EC2 instance memory usage.
- **Solution:** Install CloudWatch Agent, configure to publish mem_used_percent
- **Why:** Basic EC2 metrics don't include memory/disk (only CPU, network, disk I/O)

**Scenario 2:** Alert when Lambda errors > 5% of invocations.
- **Solution:** Use Metric Math alarm: `errors/invocations*100 > 5`
- **Why:** Direct error count doesn't account for traffic volume

**Scenario 3:** Reduce alarm noise from transient spikes.
- **Solution:** Set datapoints to alarm = 3 out of 5 evaluation periods
- **Why:** Allows brief spikes while catching sustained issues

**Scenario 4:** Centralize logs from 20 application accounts.
- **Solution:** Subscription filters in each account → Central Kinesis → S3
- **Why:** Cross-account log aggregation for compliance and centralized search

**Scenario 5:** Emit custom metrics from Lambda without increasing cost.
- **Solution:** Use Embedded Metric Format (EMF) in stdout
- **Why:** No PutMetricData API calls (which cost $0.01 per 1000 requests)

**Scenario 6:** Detect traffic drop (possible DDoS or outage).
- **Solution:** Anomaly detection alarm on request count (learns baseline)
- **Why:** Fixed threshold won't catch drops, anomaly detection flags deviations

**Scenario 7:** Query logs only when investigating incidents (infrequent).
- **Solution:** Use Infrequent Access log class
- **Why:** 50% lower ingestion cost, same query cost (cost-optimized for rare queries)

---

## 14. Interview Q&A

### Q1: Explain CloudWatch metric dimensions and cardinality concerns
**Answer:**

**What are dimensions?**
Dimensions are name-value pairs that identify a metric. Different dimension combinations = different time series.

**Example:**
```
Namespace: OrderService
MetricName: ResponseTime
Dimensions:
  - Environment=production, Region=us-east-1
  - Environment=production, Region=eu-west-1
  - Environment=staging, Region=us-east-1
```
This creates **3 separate time series** (one per unique dimension combination).

**Cardinality Problem:**
Adding high-cardinality dimensions (RequestID, UserID, SessionID) creates near-infinite time series:
- 1 million unique RequestIDs = 1 million metrics
- Cost: $0.30/metric/month × 1 million = $300,000/month

**Best Practice:**
- ✅ Use bounded dimensions: Environment, Region, Service, Operation
- ✅ Keep high-cardinality IDs in logs (query when needed)
- ❌ Avoid: RequestID, UserID, TraceID as dimensions

**Interview Tip:** "Dimensions are for aggregation axes; logs are for investigation details."

---

### Q2: How would you design alarms for a queue-processing system?
**Answer:**

**Metrics to Monitor:**
1. **ApproximateAgeOfOldestMessage** (most critical)
   - Threshold: > 5 minutes sustained
   - Indicates: backlog growing faster than processing

2. **ApproximateNumberOfMessagesVisible**
   - Context-dependent (10 messages might be normal or alarming)
   - Use with age metric for full picture

3. **NumberOfMessagesDeleted**
   - Sudden drop indicates consumer failure

4. **DLQ Messages**
   - Any message in DLQ = investigation needed

**Interview Key Point:** "Age captures backlog growth; depth needs processing rate context."

---

### Q3: Explain the difference between metric filters and subscription filters
**Answer:**

**Metric Filters:**
- **Purpose:** Extract metrics from log patterns
- **Direction:** Logs → Metrics
- **Cost:** Free (metric storage costs apply)
- **Use case:** Convert log events to quantifiable metrics

**Subscription Filters:**
- **Purpose:** Stream logs to destinations (Lambda, Kinesis, Firehose)
- **Direction:** Logs → Processing pipeline
- **Cost:** Free (destination processing costs apply)
- **Use case:** Real-time log processing, centralized aggregation

**Interview Tip:** "Metric filters for dashboards/alarms; subscription filters for complex processing."

---

### Q4: How does Embedded Metric Format (EMF) reduce costs?
**Answer:**

**Problem:** PutMetricData API costs $0.01 per 1000 requests

**EMF Solution:** Print JSON to Lambda stdout - CloudWatch auto-extracts metrics

**Cost Comparison (1M Lambda invocations):**
- **PutMetricData:** $10
- **EMF:** $0 (logs already sent)

**Interview Tip:** "EMF is the preferred way to emit custom metrics from Lambda - zero API cost and atomic logs."

---

### Q5: How do you monitor EC2 instance memory and disk usage?
**Answer:**

**Problem:** EC2 default metrics DON'T include memory or disk usage

**Solution:** Install CloudWatch Agent

**Interview Tip:** "Default EC2 metrics are host-level only; guest metrics require the CloudWatch Agent."

---

### Q6: Design an alarm strategy to reduce alert fatigue
**Answer:**

**Strategies:**
1. **Composite Alarms** - correlate multiple signals (only alert when errors AND latency high)
2. **M out of N evaluation** - tolerate transient spikes (3 of 5 periods)
3. **Anomaly Detection** - dynamic thresholds for variable workloads
4. **Appropriate thresholds** - error rate + minimum traffic (not absolute errors)

**Interview Tip:** "Page on user impact, not component symptoms."

---

### Q7: Compare CloudWatch with third-party monitoring
**Answer:**

**CloudWatch Strengths:**
- Native AWS integration, low latency, cost-effective for AWS workloads
- No data egress costs

**Limitations:**
- Basic UI, AWS-centric, limited cross-cloud support

**Third-Party (Datadog, New Relic):**
- Better UX, cross-cloud, advanced APM
- Higher cost (2-5x), data egress fees

**Interview Tip:** "CloudWatch excels for AWS-native workloads; third-party tools win on UX and multi-cloud."

---

### Q8: How would you troubleshoot missing CloudWatch metrics?
**Answer:**

**Checklist:**
1. Verify metric identity (namespace, name, dimensions)
2. Check IAM permissions (PutMetricData)
3. Verify region (metrics are region-specific)
4. Check time range
5. Dimension mismatch (case-sensitive)
6. Agent/service running
7. Network/VPC issues

**Interview Tip:** "Start with list-metrics to confirm existence, then check permissions and region."

---

### Q9: Explain CloudWatch Logs Insights query language
**Answer:**

**Purpose:** Interactive SQL-like querying of CloudWatch Logs (pay per GB scanned)

**Basic Structure:**
```
fields <field-list>
| filter <condition>
| stats <aggregation> by <field>
| sort <field> <asc|desc>
| limit <number>
```

**Interview Tip:** "Logs Insights is for ad-hoc investigation; metric filters for real-time dashboards/alarms."

---

### Q10: Design a monitoring strategy for a microservices architecture
**Answer:**

**Strategy:**
1. **Golden Signals** per service: Latency, Traffic, Errors, Saturation
2. **Centralized logging** with subscription filters → Kinesis → S3
3. **Distributed tracing** (X-Ray) for request flow visibility
4. **Composite alarms** aligned to SLAs
5. **Synthetics canaries** for critical user journeys
6. **EMF for cost-effective custom metrics**

**Interview Key Point:** "Composite alarms aligned to SLAs, not individual component thresholds."

---

**Additional Quick Answers:**

**What would you alert on for a queue worker?** Oldest-message age, processing failures, DLQ arrivals, and consumer health; queue depth alone lacks workload context.

**Metrics or logs?** Metrics detect and summarize; logs explain individual events; traces connect the path. Use them together.

**Why not every request ID as a metric dimension?** It creates nearly one time series per request, hurting cost and usefulness. Keep IDs in logs/traces.

**Can missing data mean healthy?** Only by explicit policy. A missing heartbeat is a failure, while a missing sparse error metric can be normal.

**How do you avoid alert fatigue?** Page on actionable user-impacting conditions, group correlated symptoms, assign owners, and tune with incident evidence.

---

## 15. Quick Revision

- Metrics detect and summarize; logs explain individual events; traces connect paths
- Dimension cardinality drives cost: bound dimensions, not requestIDs
- Alarm on sustained user impact (M out of N), not transient spikes
- EMF = cost-effective custom metrics from Lambda (no API calls)
- EC2 memory/disk requires CloudWatch Agent (not default metrics)
- Composite alarms reduce noise by correlating symptoms
- Log Classes: Standard for frequent queries, Infrequent Access for compliance logs

---

## 16. AWS sources

- [Metric identity, resolution, and retention](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html)
- [Alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html)
- [CloudWatch Logs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html)
- [CloudWatch features](https://aws.amazon.com/cloudwatch/features/)
- [Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)

**Related:** [RDS](aws-rds-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md), [EventBridge](aws-eventbridge-cheatsheet.md).
