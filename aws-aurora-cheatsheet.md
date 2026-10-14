# Amazon Aurora Cheat Sheet

**Interview preparation: cluster storage, readers, Serverless v2, Global Database, and failover.**

**Verified:** 2026-10-13. Engine/version and Region restrictions matter, especially for auto-pause, global features, and scaling limits.

[All AWS cheatsheets](aws-core-services-interview-index.md) · [RDS fundamentals](aws-rds-cheatsheet.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Storage and availability](#2-storage-and-availability)
- [3. Endpoints and consistency](#3-endpoints-and-consistency)
- [4. Provisioned versus Serverless v2](#4-provisioned-versus-serverless-v2)
- [5. Current correction: scale to zero](#5-current-correction-scale-to-zero)
- [6. Global Database and regional recovery](#6-global-database-and-regional-recovery)
- [7. Backups, cloning, and deployment](#7-backups-cloning-and-deployment)
- [8. Security, performance, and costs](#8-security-performance-and-costs)
- [9. Common patterns](#9-common-patterns)
- [10. Troubleshooting and practical inspection](#10-troubleshooting-and-practical-inspection)
- [11. Quick reference card](#11-quick-reference-card)
- [12. Exam tips summary](#12-exam-tips-summary)
- [13. Interview questions and detailed answers](#13-interview-questions-and-detailed-answers)
- [14. Quick revision](#14-quick-revision)
- [15. AWS sources](#15-aws-sources)

---

## 1. Fundamentals and mental model

Aurora is a MySQL- or PostgreSQL-compatible relational database with a distributed storage architecture. Compatibility does not mean every upstream extension, version, and operational behavior is identical.

```text
Writer endpoint → writer instance ─┐
Reader endpoint → reader instances ├── shared distributed cluster volume
Instance endpoint → chosen DB ────┘      replicated across three AZs
```

Compute and storage scale differently. Adding a reader does not create another independent full copy of the cluster volume inside the same Aurora cluster.

## 2. Storage and availability

Aurora stores six copies of data across three AZs. Its storage design tolerates storage failures independently of DB instance failover. Place reader instances across AZs to provide compute failover targets as well. [Aurora high availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)

**Trap:** replicated storage does not guarantee uninterrupted queries if the only DB instance fails. Storage durability, compute availability, and application connection recovery are distinct.

| Component | Responsibility |
|---|---|
| Writer | Handles writes and can serve reads |
| Aurora reader | Offloads reads; eligible failover target |
| Cluster volume | Distributed durable storage |
| Promotion tier | Influences failover target selection |
| Application pool | Reconnects and resolves ambiguous transaction outcomes |

## 3. Endpoints and consistency

| Endpoint | Intended use | Trap |
|---|---|---|
| Cluster/writer | Writes and writer-dependent reads | Existing connections still fail during failover |
| Reader | Distribute new connections among readers | Not per-query load balancing |
| Instance | Address one specific instance | Application must handle that instance changing role |
| Custom | Route to selected subset | Membership and role constraints need maintenance |

The reader endpoint can connect to the writer if no readers exist. Long-lived pools can produce uneven reader load. Aurora replicas can have lag; route critical read-after-write workflows appropriately rather than assuming every reader is immediately current.

## 4. Provisioned versus Serverless v2

| Option | Use when | Cost/operation concern |
|---|---|---|
| Provisioned instances | Predictable resource needs and instance control | Resize planning and idle capacity |
| Serverless v2 | Variable demand and granular compute adjustment | Minimum/maximum ACUs and scaling behavior |
| Mixed cluster where supported | Different writer/reader resource profiles | Capacity/failover behavior must be deliberate |

An Aurora Capacity Unit (ACU) represents approximately 2 GiB of memory with corresponding CPU/network capacity. Serverless v2 adjusts in fine increments; it does not eliminate query tuning or connection limits. Readers still provide horizontal read scaling. [Serverless architecture](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)

Do not memorize one universal maximum ACU or storage size: these depend on engine/platform versions. Set the maximum high enough for real peaks and monitor saturation.

## 5. Current correction: scale to zero

**Outdated claim:** “Serverless v2 always has a 0.5 ACU floor.” Supported versions can use **0 ACU with automatic pause/resume**. Idle detection, configured timeout, and feature compatibility determine whether pause actually happens.

- Open connections can prevent pausing.
- RDS Proxy connections can prevent pausing.
- Global Database has auto-pause restrictions, including the primary writer and secondary cluster instances.
- Resume latency matters for the first request after idle.
- Storage and other retained resources are still billable while compute is paused.

This makes auto-pause useful for suitable intermittent workloads; it is not an automatic recommendation for latency-sensitive production traffic. [Auto-pause requirements and limitations](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)

## 6. Global Database and regional recovery

Aurora Global Database uses a primary Region for writes and secondary Regions for local reads and disaster recovery. Cross-Region replication is asynchronous; unplanned promotion may lose changes that have not replicated.

| Operation | Purpose | Interview distinction |
|---|---|---|
| Planned switchover | Move primary during a controlled operation | Coordinate synchronization and application cutover |
| Unplanned failover | Recover from primary Region impairment | Evaluate replication lag/data-loss risk |
| Write forwarding where supported | Send writes from secondary through to primary | Does not create independent multi-writer Regions |

Plan traffic routing, DB endpoints, credentials, KMS keys, alarms, recovery capacity, and failback. A replicated database alone is not a complete multi-Region application. [Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)

## 7. Backups, cloning, and deployment

- Use automated backups/PITR for logical recovery and snapshots for explicit recovery points.
- Test restoration and credentials/network access before declaring an RTO.
- Fast cloning is useful for development and tests; understand copy-on-write storage cost as data diverges.
- Use supported blue/green deployment workflows for safer upgrades; check engine/version constraints and cutover behavior.
- Keep schema changes compatible with both old and new application versions during rollout.

Do not confuse a clone, a read replica, and a regional secondary: they have different synchronization and lifecycle purposes.

## 8. Security, performance, and costs

Use private networking, TLS, least-privilege database users, controlled secret rotation/IAM authentication where supported, and KMS encryption. Protect both data access and administrative actions.

Measure query plans, DB load/waits, locks, connections, buffer-cache behavior, replica lag, and ACU/instance saturation. Use current **CloudWatch Database Insights** terminology following the July 2026 Performance Insights transition. [Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)

Compare Aurora Standard with I/O-Optimized based on the actual compute/storage/I/O bill. I/O-Optimized can benefit I/O-intensive workloads, but “always cheaper” is not a defensible answer.

## 9. Common Patterns

### Pattern 1: Global Database for DR
```bash
# Primary region: us-east-1
aws rds create-global-cluster \
  --global-cluster-identifier my-global-db \
  --engine aurora-mysql \
  --engine-version 8.0.mysql_aurora.3.02.0

# Add primary cluster
aws rds create-db-cluster \
  --db-cluster-identifier primary-cluster \
  --engine aurora-mysql \
  --global-cluster-identifier my-global-db \
  --master-username admin \
  --master-user-password <password> \
  --region us-east-1

# Secondary region: eu-west-1 (read-only, <1s lag)
aws rds create-db-cluster \
  --db-cluster-identifier secondary-cluster \
  --engine aurora-mysql \
  --global-cluster-identifier my-global-db \
  --region eu-west-1
```

**Use Case:** Multi-region disaster recovery with RPO < 1 second, RTO < 1 minute

---

### Pattern 2: Auto Scaling Read Replicas
```python
# Auto-scaling policy for read replicas
import boto3

client = boto3.client('application-autoscaling')

# Register Aurora cluster as scalable target
client.register_scalable_target(
    ServiceNamespace='rds',
    ResourceId='cluster:my-aurora-cluster',
    ScalableDimension='rds:cluster:ReadReplicaCount',
    MinCapacity=1,
    MaxCapacity=15
)

# Create scaling policy based on CPU
client.put_scaling_policy(
    PolicyName='aurora-read-replica-scaling',
    ServiceNamespace='rds',
    ResourceId='cluster:my-aurora-cluster',
    ScalableDimension='rds:cluster:ReadReplicaCount',
    PolicyType='TargetTrackingScaling',
    TargetTrackingScalingPolicyConfiguration={
        'TargetValue': 70.0,
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'RDSReaderAverageCPUUtilization'
        },
        'ScaleInCooldown': 300,
        'ScaleOutCooldown': 60
    }
)
```

**Architecture:**
```
[Primary Writer] --> [Storage Layer (6 copies)]
                          |
        +-----------------+-----------------+
        |                 |                 |
    [Reader 1]       [Reader 2]  ... [Reader 15]
                  (Auto-scales based on CPU/Connections)
```

---

### Pattern 3: Aurora Serverless v2 with Proxy
```bash
# Create Serverless v2 cluster
aws rds create-db-cluster \
  --db-cluster-identifier serverless-cluster \
  --engine aurora-mysql \
  --engine-mode provisioned \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --master-username admin \
  --master-user-password <password>

# Create Serverless v2 instance
aws rds create-db-instance \
  --db-instance-identifier serverless-instance \
  --db-instance-class db.serverless \
  --engine aurora-mysql \
  --db-cluster-identifier serverless-cluster

# Add RDS Proxy for connection pooling
aws rds create-db-proxy \
  --db-proxy-name my-proxy \
  --engine-family MYSQL \
  --auth SecretArn=arn:aws:secretsmanager:region:account:secret:db-secret \
  --role-arn arn:aws:iam::account:role/RDSProxyRole \
  --vpc-subnet-ids subnet-1 subnet-2 \
  --require-tls

# Register target
aws rds register-db-proxy-targets \
  --db-proxy-name my-proxy \
  --db-cluster-identifiers serverless-cluster
```

**Use Case:** Variable workloads (e.g., development, staging, sporadic apps) with connection pooling

---

### Pattern 4: Blue/Green Deployments
```bash
# Create blue/green deployment for version upgrade
aws rds create-blue-green-deployment \
  --blue-green-deployment-identifier my-bg-deployment \
  --source-arn arn:aws:rds:region:account:cluster:production-cluster \
  --target-engine-version 8.0.mysql_aurora.3.03.0 \
  --target-db-parameter-group-name aurora-mysql8-params

# Test green environment
# ... run tests against green cluster endpoint ...

# Switchover (typically <1 minute downtime)
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier my-bg-deployment \
  --switchover-timeout 300

# Delete old blue environment after validation
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier my-bg-deployment \
  --delete-target
```

**Timeline:**
```
Blue (Production)  --> Clone --> Green (Staging)
      |                             |
   [Users] ----------------------> [Testing]
      |                             |
   [Switchover in <1 min]          |
      |                             |
   [Users] <---------------------- [Now Production]
```

---

### Pattern 5: Cross-Region Read Replica
```bash
# Primary cluster in us-east-1
aws rds create-db-cluster \
  --db-cluster-identifier primary-us-east \
  --engine aurora-postgresql \
  --master-username admin \
  --master-user-password <password> \
  --region us-east-1

# Cross-region read replica in ap-southeast-1
aws rds create-db-cluster \
  --db-cluster-identifier replica-ap-southeast \
  --replication-source-identifier arn:aws:rds:us-east-1:account:cluster:primary-us-east \
  --engine aurora-postgresql \
  --region ap-southeast-1
```

**Latency:** Typical cross-region lag is 1-3 seconds (async replication)

---

### Pattern 6: Backtrack for Point-in-Time Recovery
```bash
# Enable backtrack (MySQL only, up to 72 hours)
aws rds modify-db-cluster \
  --db-cluster-identifier my-cluster \
  --backtrack-window 72 \
  --apply-immediately

# Backtrack to specific time
aws rds backtrack-db-cluster \
  --db-cluster-identifier my-cluster \
  --backtrack-to "2024-01-15T10:30:00Z"

# Check backtrack status
aws rds describe-db-cluster-backtracks \
  --db-cluster-identifier my-cluster
```

**Use Case:** Recover from accidental deletes/updates without restoring from snapshot

---

## 10. Troubleshooting and practical inspection

| Symptom | Investigate |
|---|---|
| One reader much busier | Long-lived pools and connection-level balancing |
| Writes fail on read connection | Reader/instance endpoint and current instance role |
| Serverless never pauses | Connections, proxy, global/other incompatible features |
| High latency after quiet period | Resume/warmup versus SQL regression |
| Failover occurred but app remains down | DNS cache, connection pool, credentials, retry limits |
| Secondary reads stale | Global replication lag and consistency expectations |

Read-only examples:

```bash
aws rds describe-db-clusters --db-cluster-identifier orders-aurora
aws rds describe-db-instances --filters Name=db-cluster-id,Values=orders-aurora
aws rds describe-db-cluster-endpoints --db-cluster-identifier orders-aurora
aws rds describe-global-clusters
```

---

## 11. Quick Reference Card

### Aurora vs RDS Comparison
| Feature | Aurora | RDS MySQL/PostgreSQL |
|---------|--------|---------------------|
| **Storage auto-scaling** | Yes (10GB → 128TB) | Manual resize required |
| **Read replicas** | Up to 15 (same region) | Up to 5 |
| **Replication lag** | <10ms (storage-level) | Seconds to minutes |
| **Failover time** | <30 seconds | 1-2 minutes |
| **Backup retention** | 1-35 days | 0-35 days |
| **Backtrack** | Yes (MySQL only, 72h) | No |
| **Global Database** | Yes (cross-region <1s lag) | Manual cross-region replica |
| **Serverless** | v1 and v2 | Not available |
| **Cost** | ~20% higher than RDS | Baseline cost |

---

### Aurora Serverless Decision Matrix
| Scenario | v1 | v2 | Provisioned |
|----------|----|----|-------------|
| Infrequent workloads (pause between uses) | ✅ | ❌ | ❌ |
| Variable but consistent traffic | ❌ | ✅ | ❌ |
| Predictable, steady traffic | ❌ | ❌ | ✅ |
| Need instant scaling | ❌ | ✅ | ❌ |
| Dev/test environments | ✅ | ✅ | ❌ |
| Production with SLA | ❌ | ✅ | ✅ |
| Multi-AZ required | ❌ | ✅ | ✅ |

---

### Instance Class Selection Guide
```
┌─────────────────────────────────────────────────┐
│ Workload Type          Recommended Class        │
├─────────────────────────────────────────────────┤
│ Dev/Test               db.t4g.medium            │
│ Small prod (<500GB)    db.r6g.large             │
│ Medium prod (500GB-2TB) db.r6g.xlarge/2xlarge  │
│ Large prod (>2TB)      db.r6g.4xlarge+          │
│ Write-intensive        db.r6i (Intel + NVMe)    │
│ Read-intensive         db.r6g (Graviton2)       │
│ Variable workload      db.serverless            │
└─────────────────────────────────────────────────┘
```

---

### Key CloudWatch Metrics
```bash
# Monitor critical metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBClusterIdentifier,Value=my-cluster \
  --start-time 2024-01-15T00:00:00Z \
  --end-time 2024-01-15T23:59:59Z \
  --period 300 \
  --statistics Average
```

**Top Metrics to Monitor:**
1. **CPUUtilization** → Scale if >70% sustained
2. **DatabaseConnections** → Check against max_connections
3. **AuroraReplicaLag** → Should be <100ms
4. **CommitLatency** → Write performance indicator
5. **SelectLatency** → Read performance indicator
6. **FreeableMemory** → Should stay >20% of total
7. **VolumeReadIOPs / VolumeWriteIOPs** → I/O patterns

---

### Cost Optimization
```python
# Stop unused Aurora clusters (Serverless v1 auto-pauses)
# For provisioned clusters, use EventBridge + Lambda

import boto3
from datetime import datetime, timedelta

rds = boto3.client('rds')

def stop_idle_clusters():
    clusters = rds.describe_db_clusters()['DBClusters']
    
    for cluster in clusters:
        cluster_id = cluster['DBClusterIdentifier']
        
        # Check CloudWatch for low activity
        cw = boto3.client('cloudwatch')
        stats = cw.get_metric_statistics(
            Namespace='AWS/RDS',
            MetricName='DatabaseConnections',
            Dimensions=[{'Name': 'DBClusterIdentifier', 'Value': cluster_id}],
            StartTime=datetime.utcnow() - timedelta(hours=4),
            EndTime=datetime.utcnow(),
            Period=3600,
            Statistics=['Average']
        )
        
        avg_connections = sum(p['Average'] for p in stats['Datapoints']) / len(stats['Datapoints'])
        
        if avg_connections < 5 and 'dev' in cluster_id:
            print(f"Stopping {cluster_id} (low activity)")
            rds.stop_db_cluster(DBClusterIdentifier=cluster_id)
```

**Cost Savings Tips:**
- Use Graviton2 instances (db.r6g) → 20% cheaper than Intel
- Enable auto-pause for Serverless v1 (idle >5 min)
- Use Reserved Instances for production → 40-60% savings
- Delete old snapshots (keep only required retention)
- Use Aurora I/O-Optimized for high I/O workloads → predictable pricing

---

## 12. Exam Tips Summary

### Key Concepts for AWS Exams

1. **Aurora Storage Architecture**
   - 6 copies across 3 AZs (4/6 write quorum, 3/6 read quorum)
   - Self-healing storage (automatic detection and repair)
   - Only redo logs written to storage (not full pages like RDS)

2. **Failover Mechanism**
   - Priority tiers (0-15, lower = higher priority)
   - <30 seconds typical failover time
   - Promotes read replica to writer automatically

3. **Global Database**
   - Primary region: read-write
   - Secondary regions: read-only (<1 second lag)
   - Disaster recovery: promote secondary to primary in <1 minute
   - Uses storage-based replication (not binlog/logical)

4. **Serverless v1 vs v2**
   - **v1:** Auto-pause after inactivity, scales in ACU units (2GB RAM per ACU), cold start latency
   - **v2:** No cold start, instant scaling (0.5-128 ACUs in 0.5 increments), Multi-AZ support
   - **Exam trap:** v1 cannot scale to zero instantly; v2 cannot auto-pause

5. **Backtrack (MySQL only)**
   - Rewinds database to specific point in time (up to 72 hours)
   - Does NOT create new cluster (in-place operation)
   - Faster than snapshot restore
   - Not available for PostgreSQL (use PITR from snapshot instead)

6. **Aurora Replicas vs MySQL Read Replicas**
   - Aurora replicas: same storage layer, <10ms lag, auto-failover
   - MySQL read replicas: async replication, seconds lag, no auto-failover

7. **Cloning**
   - Fast, copy-on-write mechanism
   - Uses same storage until data diverges
   - Use case: create dev/test from production in minutes

8. **Endpoints**
   - **Cluster endpoint:** Always points to primary writer
   - **Reader endpoint:** Load balances across all read replicas
   - **Custom endpoint:** Target specific instance subset
   - **Instance endpoint:** Direct connection to specific instance

9. **Performance Insights**
   - Free for 7 days retention
   - Paid for longer retention
   - Shows top SQL, wait events, database load

10. **Integration with Other Services**
    - **Lambda:** Invoke functions from stored procedures (Aurora MySQL)
    - **S3:** Export/import data directly (SELECT INTO S3, LOAD FROM S3)
    - **Secrets Manager:** Automatic password rotation
    - **RDS Proxy:** Connection pooling (reduce database overhead)

11. **Blue/Green Deployments**
    - Clone production to staging (green)
    - Test upgrades on green
    - Switchover with <1 minute downtime
    - Easy rollback (swap back to blue)

12. **Cost Factors**
    - Instance hours (per instance)
    - Storage (per GB-month) → no charge for backup storage up to DB size
    - I/O requests (unless I/O-Optimized)
    - Backup storage beyond DB size
    - Data transfer (cross-region, out to internet)

13. **Security Best Practices**
    - Encryption at rest (KMS) → must enable at creation
    - Encryption in transit (SSL/TLS)
    - IAM database authentication (no passwords)
    - Network isolation (VPC, security groups, private subnets)

14. **Multi-Master (MySQL only)**
    - All nodes are read-write
    - Conflict detection and resolution
    - Use case: Multi-region active-active
    - Exam mention: rarely tested, know it exists

15. **Machine Learning Integration**
    - Aurora ML: invoke SageMaker/Comprehend from SQL
    - Example: `SELECT aws_sagemaker_invoke_endpoint('fraud-model', input_data) FROM transactions;`

---

### Scenario-Based Exam Questions

**Scenario 1:** Application experiences high connection churn (1000s of new connections per minute).
- **Solution:** Add RDS Proxy in front of Aurora cluster
- **Why:** Proxy pools connections, reduces overhead on database

**Scenario 2:** Need to test database upgrade without impacting production.
- **Solution:** Use Blue/Green Deployment
- **Why:** Creates clone, allows testing, fast switchover

**Scenario 3:** Accidentally deleted 10,000 rows 20 minutes ago, need fastest recovery.
- **Solution:** Use Backtrack (if MySQL and enabled)
- **Why:** Faster than snapshot restore, in-place rewind

**Scenario 4:** Read-heavy workload in multiple global regions with low latency.
- **Solution:** Aurora Global Database with secondary clusters in each region
- **Why:** <1 second replication lag, local reads in each region

**Scenario 5:** Variable workload during business hours, idle at night.
- **Solution:** Aurora Serverless v1 with auto-pause enabled
- **Why:** Automatically pauses after inactivity, scales on demand

**Scenario 6:** Need 99.99% availability with automated failover.
- **Solution:** Aurora Multi-AZ with read replicas
- **Why:** Automatic failover to replica in <30 seconds

---

## 13. Interview questions and detailed answers

### Q1: Explain Aurora's 6-copy storage architecture and quorum model
**Answer:**
Aurora stores 6 copies of data across 3 Availability Zones (2 copies per AZ).

**Write Quorum (4/6):**
- 4 out of 6 copies must acknowledge a write
- Can tolerate 2 copy failures without impacting writes
- Ensures durability

**Read Quorum (3/6):**
- 3 out of 6 copies must be available for reads
- Can tolerate 3 copy failures and still serve reads
- Ensures availability

**Self-Healing:**
- Continuous background scanning
- Detects and repairs corrupt blocks automatically
- Uses peer-to-peer gossip protocol

**Benefit:** Can lose entire AZ + 1 copy and still function.

---

### Q2: Aurora Serverless v1 vs v2 - When to use each?
**Answer:**

**Use Serverless v1 when:**
- Infrequent, intermittent workloads (e.g., weekend batch jobs)
- Development/test databases that aren't used 24/7
- Need auto-pause to save costs (pauses after 5-10 min idle)
- Can tolerate cold start latency (~25 seconds)

**Use Serverless v2 when:**
- Variable but consistent workload (e.g., e-commerce with traffic spikes)
- Need instant scaling without cold start
- Production workloads requiring Multi-AZ
- Fine-grained scaling (0.5 ACU increments vs v1's 2 ACU)
- Can't afford connection drops during scaling

**Key Difference:**
- **v1:** Scales capacity (ACU) by proxy switchover → brief connection drop
- **v2:** Scales capacity without disconnecting clients → seamless

**Exam Trap:** v1 can auto-pause; v2 support varies by configuration. v2 is better for production.

---

### Q3: How does Aurora achieve <30 second failover?
**Answer:**

**Traditional RDS Failover (1-2 minutes):**
1. Detect primary failure (30-60 seconds)
2. Promote standby
3. Update DNS (30-60 seconds propagation)

**Aurora Failover (<30 seconds):**
1. **Shared storage layer:** Replica already has all data (no sync needed)
2. **Fast detection:** Continuous health checks every 1 second
3. **Priority-based promotion:**
   - Tier 0 replicas checked first
   - Same tier → largest replica promoted
4. **DNS update:** Cluster endpoint updated to new primary
5. **No buffer pool warming needed:** Replica already has cache

**Steps:**
```
[Primary fails] → [Health check detects in 1s]
               → [Promote highest priority replica in 5-10s]
               → [Update cluster endpoint in 5-10s]
               → [New primary ready in <30s total]
```

**Application Best Practice:**
- Use cluster endpoint (not instance endpoint)
- Implement connection retry logic
- Monitor `FailoverStarted` CloudWatch event

---

### Q4: Explain Aurora Global Database architecture and use cases
**Answer:**

**Architecture:**
```
Primary Region (us-east-1)
├── Primary Writer (read-write)
├── Storage Layer (6 copies)
└── Read Replicas (0-15)
         |
         | (Storage-level replication, <1s lag)
         ↓
Secondary Region (eu-west-1)
├── Secondary Cluster (read-only)
└── Read Replicas (0-15)
```

**Replication:**
- Physical replication at storage layer (not logical/binlog)
- Typical lag: <1 second
- Dedicated replication infrastructure

**Disaster Recovery:**
- Promote secondary to primary in <1 minute
- RPO (data loss): <1 second
- RTO (recovery time): <1 minute

**Use Cases:**
1. **Multi-region DR:** Fast recovery from regional outage
2. **Global read scaling:** Serve users from nearby region
3. **Data locality:** Comply with data residency requirements

**Exam Tip:** Global Database ≠ Multi-Master. Secondaries are read-only until promoted.

---

### Q5: How does Aurora Backtrack differ from snapshot restore?
**Answer:**

**Backtrack (MySQL only):**
- Rewinds cluster to specific timestamp (last 72 hours)
- **In-place operation:** No new cluster created
- **Speed:** Minutes (depends on change volume)
- **Use case:** Accidental DELETE/UPDATE, wrong deployment
- **Limitation:** Only goes backward (can't fast-forward)
- **Cost:** Charged per change record retained

**Snapshot Restore:**
- Creates new cluster from backup
- **Speed:** 10-30 minutes (depends on size)
- **Use case:** Test in parallel, major corruption, go back >72 hours
- **Flexibility:** Can restore to different instance class/VPC

**Comparison:**
| Feature | Backtrack | Snapshot Restore |
|---------|-----------|------------------|
| Speed | 5-15 min | 10-30 min |
| New cluster? | No | Yes |
| Downtime | ~30s | Testing required |
| Go back | Up to 72h | Up to 35 days |
| PostgreSQL | ❌ | ✅ |

**Exam Scenario:** "Recover from accidental data deletion 20 minutes ago" → **Backtrack** (fastest)

---

### Q6: Explain Aurora cloning and its use cases
**Answer:**

**Clone Mechanism:**
- Copy-on-write at storage layer
- Initial clone uses same storage pages as source
- New pages allocated only when data diverges
- **Speed:** Minutes (regardless of database size)

**Process:**
```bash
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier prod-cluster \
  --db-cluster-identifier dev-clone \
  --restore-type copy-on-write \
  --use-latest-restorable-time
```

**Use Cases:**
1. **Dev/Test Environments:**
   - Clone production for testing
   - Full dataset without storage duplication cost
2. **Analytics:**
   - Clone for heavy queries
   - Don't impact production
3. **Blue/Green Deployments:**
   - Clone for testing upgrades
   - Fast switchover after validation
4. **Pre-Production Staging:**
   - Test with real data
   - Reset frequently without cost

**Cost:**
- Initially, minimal storage cost (shared pages)
- Cost increases as clone data diverges
- Compute billed separately (instance hours)

**Exam Tip:** "Create test environment from 10TB production database in 5 minutes" → **Aurora clone**

---

### Q7: When should you use RDS Proxy with Aurora?
**Answer:**

**Problem RDS Proxy Solves:**
- Applications open many short-lived connections (connection churn)
- Lambda functions creating 100s of connections simultaneously
- Max connections exhausted

**How Proxy Helps:**
1. **Connection pooling:** Maintains pool of database connections
2. **Multiplexing:** Many client connections share fewer DB connections
3. **Failover handling:** Reduces failover time by maintaining connections
4. **IAM authentication:** Centralized auth management

**Architecture:**
```
[Lambda x 1000] → [RDS Proxy] → [Aurora Cluster]
                  (Connection      (Few stable
                   pooling)         connections)
```

**Configuration:**
```python
import pymysql

# Connect through proxy endpoint
connection = pymysql.connect(
    host='my-proxy.proxy-xyz.us-east-1.rds.amazonaws.com',
    user='admin',
    password='password',
    database='mydb',
    ssl={'ssl': True}
)
```

**When to Use:**
- Serverless applications (Lambda, Fargate)
- Applications with unpredictable connection patterns
- Need to minimize failover impact (proxy maintains connections)

**When NOT to Use:**
- Stable, long-lived connections (traditional apps)
- Low connection count (<100 concurrent)

**Exam Scenario:** "Lambda function timing out due to database connection errors" → **Add RDS Proxy**

---

### Q8: Compare Aurora Multi-Master with Global Database
**Answer:**

**Aurora Multi-Master (MySQL only):**
- All nodes in **same region** are read-write
- Conflict detection if two nodes update same row
- Use case: Write scalability within a region
- **Limitation:** Single region only

**Aurora Global Database:**
- Primary region: read-write
- Secondary regions: read-only (until promoted)
- Use case: Multi-region DR, global read scaling
- **Limitation:** Writes only in primary region

**Comparison:**
| Feature | Multi-Master | Global Database |
|---------|--------------|-----------------|
| Write nodes | Multiple (same region) | Single region |
| Regions | Single | Multiple |
| Conflict resolution | Yes | N/A (single writer) |
| Use case | Write scaling | DR, global reads |
| Availability | 2024 - Limited | Generally Available |

**Exam Tip:** Multi-Master rarely tested; focus on Global Database for cross-region scenarios.

---

### Q9: Describe Aurora's integration with AWS Lambda
**Answer:**

**Invoke Lambda from SQL (Aurora MySQL):**
```sql
-- Setup: Grant Lambda invoke permission
-- In MySQL:
SELECT LAMBDA_ASYNC('arn:aws:lambda:region:account:function:ProcessOrder', 
                     JSON_OBJECT('order_id', order_id, 'total', total_amount))
FROM orders 
WHERE status = 'pending';
```

**Use Cases:**
1. **Trigger business logic:** Send email when order placed
2. **Data enrichment:** Call external API to validate address
3. **Event streaming:** Send change events to EventBridge
4. **Notifications:** Alert when threshold exceeded

**Setup Requirements:**
```bash
# 1. Create IAM role for Aurora to invoke Lambda
# 2. Attach role to Aurora cluster
aws rds add-role-to-db-cluster \
  --db-cluster-identifier my-cluster \
  --feature-name Lambda \
  --role-arn arn:aws:iam::account:role/AuroraLambdaRole

# 3. Install aws_lambda extension in Aurora
# 4. Grant Lambda permissions in MySQL
```

**Lambda Accessing Aurora:**
- Use RDS Proxy for connection pooling
- Use Secrets Manager for credentials
- Deploy Lambda in same VPC (private subnets)
- Use IAM database authentication (no passwords)

**Exam Tip:** Aurora → Lambda is async; Lambda → Aurora needs connection management.

---

### Q10: How do you secure Aurora at rest and in transit?
**Answer:**

**At Rest Encryption:**
```bash
# Enable at cluster creation (cannot enable later)
aws rds create-db-cluster \
  --db-cluster-identifier secure-cluster \
  --engine aurora-mysql \
  --kms-key-id arn:aws:kms:region:account:key/key-id \
  --storage-encrypted
```

**Key Points:**
- Must enable at creation (can't encrypt existing unencrypted cluster)
- Uses AWS KMS (customer-managed or AWS-managed key)
- Encrypts: storage, backups, snapshots, replicas
- Minimal performance impact (<5%)

**Migrating Unencrypted → Encrypted:**
1. Create snapshot of unencrypted cluster
2. Copy snapshot with encryption enabled
3. Restore from encrypted snapshot
4. Update application endpoint

**In Transit Encryption:**
```python
# Force SSL connections
import pymysql

connection = pymysql.connect(
    host='my-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
    user='admin',
    password='password',
    ssl={'ca': '/path/to/rds-ca-bundle.pem'}  # RDS CA bundle
)
```

**Database-Level Enforcement:**
```sql
-- Require SSL for specific user
GRANT USAGE ON *.* TO 'appuser'@'%' REQUIRE SSL;

-- Verify SSL connection
SHOW STATUS LIKE 'Ssl_cipher';  -- Should show cipher if SSL active
```

**IAM Database Authentication:**
```python
import boto3
import pymysql

rds_client = boto3.client('rds')

# Generate auth token (valid 15 minutes)
token = rds_client.generate_db_auth_token(
    DBHostname='my-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
    Port=3306,
    DBUsername='iamuser'
)

# Connect using token as password
connection = pymysql.connect(
    host='my-cluster.cluster-xyz.us-east-1.rds.amazonaws.com',
    user='iamuser',
    password=token,
    ssl={'ca': '/path/to/rds-ca-bundle.pem'}
)
```

**Network Security:**
- Deploy in private subnets (no internet gateway)
- Security group: restrict to application security group only
- NACLs: additional layer (optional)

**Exam Checklist:**
- ✅ Encryption at rest (KMS)
- ✅ Encryption in transit (SSL/TLS)
- ✅ IAM database authentication
- ✅ Private subnets + security groups
- ✅ Secrets Manager for password rotation
- ✅ Enable automated backups (encryption inherits)

---

**Additional Quick Answers:**

**Why Aurora instead of ordinary RDS?** Discuss storage architecture, read scaling/failover, global needs, compatibility, and cost. Avoid claiming every application becomes faster automatically.

**Does a reader endpoint balance every SQL statement?** No, it balances connections. Pool sizing and lifetime influence distribution.

**Can Serverless v2 scale to zero?** Yes on supported configurations with auto-pause; active connections and certain features prevent it, and resume latency must be acceptable.

**Does Global Database offer active-active writers?** The standard topology has a primary writer Region. Forwarding writes still uses that writer.

**What about Aurora DSQL?** It is a separate distributed SQL service. Do not transfer its consistency/topology properties to an Aurora MySQL/PostgreSQL cluster merely because they share a brand name.

---

## 14. Quick revision

- Six storage copies/three AZs does not remove compute failover needs.
- Reader endpoint balances connections, not individual queries.
- Serverless v2 scales compute; storage/queries/connections remain design concerns.
- Zero ACU is conditional, not universal.
- Global Database replication is asynchronous; assess RPO.
- Aurora DSQL and Aurora MySQL/PostgreSQL are different services.

---

## 15. AWS sources

- [High availability](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Concepts.AuroraHighAvailability.html)
- [Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [Auto-pause](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html)
- [Global Database](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html)
- [Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)

**Related:** [RDS](aws-rds-cheatsheet.md), [DynamoDB](aws-dynamodb-cheatsheet.md), [Route 53](aws-route53-cheatsheet.md).
