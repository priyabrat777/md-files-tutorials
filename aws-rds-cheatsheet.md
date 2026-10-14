# Amazon RDS Cheat Sheet

**Complete Guide to Managed Relational Databases for AWS SAA-C03 Certification and Interview Preparation**

**Verified:** 2026-10-13. Engine, version, deployment type, and Region determine feature support. Aurora has a [separate guide](aws-aurora-cheatsheet.md).

[All AWS cheatsheets](aws-core-services-interview-index.md)

---

## Table of Contents

1. [Fundamentals and Mental Model](#1-fundamentals-and-mental-model)
2. [Deployment Types](#2-deployment-types)
3. [Read Replicas and Consistency](#3-read-replicas-and-consistency)
4. [Backups, Restore, and DR](#4-backups-restore-and-dr)
5. [Connections, Failover, and RDS Proxy](#5-connections-failover-and-rds-proxy)
6. [Security](#6-security)
7. [Performance and Scaling](#7-performance-and-scaling)
8. [Observability](#8-observability)
9. [Common Patterns](#9-common-patterns)
10. [Troubleshooting](#10-troubleshooting)
11. [Quick Reference Card](#11-quick-reference-card)
12. [Exam Tips Summary](#12-exam-tips-summary)
13. [Interview Questions and Model Answers](#13-interview-questions-and-model-answers)
14. [AWS Sources](#14-aws-sources)

---

## 1. Fundamentals and Mental Model

RDS manages infrastructure and common database operations for supported relational engines. You still own schema design, indexes, queries, access, data lifecycle, and recovery requirements.

### Full Architecture Mental Model

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          APPLICATION TIER                                │
│                     (EC2, ECS, Lambda, on-prem)                          │
└───────────────────────────────┬──────────────────────────────────────────┘
                                │ SQL connections
                                ▼
┌──────────────────────────────────────────────────────────────────────────┐
│                 CONNECTION POOL / RDS PROXY (optional)                   │
│          Multiplexes thousands of app connections → fewer DB connections │
│          Handles failover reconnect; pins long transactions to one conn.  │
└──────────┬──────────────────────────────────────────────────────────────┘
           │
     ┌─────┴──────────────────────────────────┐
     │ WRITE PATH                             │ READ PATH
     ▼                                        ▼
┌─────────────────┐                ┌──────────────────────────────┐
│  WRITER ENDPOINT│                │  READER ENDPOINT / REPLICAS  │
│  (Primary DB)   │                │  Async replication lag may   │
│  All writes go  │──async copy──► │  apply. Not for read-after-  │
│  here           │                │  write consistency.          │
└────────┬────────┘                └──────────────────────────────┘
         │
         │ Synchronous (Multi-AZ DB instance)
         │ Semi-sync (Multi-AZ DB cluster)
         ▼
┌─────────────────┐
│  STANDBY / AZ-2 │  (Multi-AZ only — does NOT serve reads in DB instance)
│  Hot standby    │
└─────────────────┘
         │
         │ Continuous backup (transaction logs)
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│              AUTOMATED BACKUPS → AMAZON S3 (managed by RDS)             │
│  Daily full snapshot + continuous transaction logs (1–35 day retention)  │
│  Enables point-in-time recovery to any second within retention window    │
└──────────────────────────┬───────────────────────────────────────────────┘
                           │  Restore always creates a NEW DB instance
                           ▼
                 ┌──────────────────────┐
                 │  NEW DB INSTANCE     │
                 │  Validate, re-point  │
                 │  app connection str. │
                 └──────────────────────┘
```

### Multi-AZ Failover Flow Diagram

```
Normal Operation:
┌──────────────────────────────────────────────────────────────────────────┐
│  Application writes to primary                                           │
│  DNS: orders-db.abc123.us-east-1.rds.amazonaws.com → Primary (AZ-A)     │
└──────────────────────────────────────────────────────────────────────────┘

Availability Zone A              Availability Zone B
┌─────────────────────┐          ┌─────────────────────┐
│  ┌────────────────┐ │          │  ┌────────────────┐ │
│  │   PRIMARY DB   │ │──sync───►│  │  STANDBY DB    │ │
│  │   (Active)     │ │repl.     │  │  (Passive)     │ │
│  └────────────────┘ │          │  └────────────────┘ │
│                     │          │                     │
│  App connections    │          │  No connections     │
│  DNS resolves here  │          │                     │
└─────────────────────┘          └─────────────────────┘

Failure Detection (AZ-A primary failure):
┌──────────────────────────────────────────────────────────────────────────┐
│  1. RDS detects primary unresponsive                                     │
│  2. Health checks fail (typically 3 missed in ~30s)                      │
│  3. Automatic failover initiated                                         │
└──────────────────────────────────────────────────────────────────────────┘

Availability Zone A              Availability Zone B
┌─────────────────────┐          ┌─────────────────────┐
│  ┌────────────────┐ │          │  ┌────────────────┐ │
│  │   PRIMARY DB   │ │ FAILED   │  │  STANDBY DB    │ │
│  │   (Failed)     │ │    X     │  │  (Promoting)   │ │
│  └────────────────┘ │          │  └────────────────┘ │
│                     │          │                     │
│  All connections    │          │  Promotion in       │
│  are terminated     │          │  progress...        │
└─────────────────────┘          └─────────────────────┘

After Failover Complete (~60 seconds total):
┌──────────────────────────────────────────────────────────────────────────┐
│  DNS: orders-db.abc123.us-east-1.rds.amazonaws.com → NEW Primary (AZ-B) │
│  Application reconnects to same DNS name                                 │
│  (Apps with retry logic reconnect automatically)                         │
└──────────────────────────────────────────────────────────────────────────┘

Availability Zone A              Availability Zone B
┌─────────────────────┐          ┌─────────────────────┐
│  ┌────────────────┐ │          │  ┌────────────────┐ │
│  │   OLD PRIMARY  │ │          │  │  NEW PRIMARY   │ │
│  │   (Down)       │ │          │  │  (Active)      │ │
│  └────────────────┘ │          │  └────────────────┘ │
│                     │          │                     │
│  No connections     │          │  All app conns here │
│                     │          │  DNS resolves here  │
└─────────────────────┘          └─────────────────────┘

Key Points:
├─ DNS endpoint stays the same (no application config change needed)
├─ Existing connections are terminated (applications must retry)
├─ No data loss (synchronous replication ensures RPO ≈ 0)
├─ RTO ≈ 60 seconds for Multi-AZ DB instance, ~35 seconds for Multi-AZ cluster
└─ RDS Event: "Multi-AZ instance failover completed" logged
```

### Read Replica Lag Flow Diagram

```
Write Flow (Writer → Replica):
┌──────────────────────────────────────────────────────────────────────────┐
│  T=0: Application writes Order #12345 to PRIMARY                         │
│       Primary commits transaction                                        │
│       Primary writes to binary log / WAL                                 │
└──────────────────────────────────────────────────────────────────────────┘
         │
         │ Asynchronous replication (network delay)
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  T=0.5s: Replica receives binary log event                               │
│          Replica begins applying transaction                             │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  T=1.2s: Replica completes applying transaction                          │
│          Order #12345 now visible on replica                             │
│          ReplicaLag = 1.2 seconds                                        │
└──────────────────────────────────────────────────────────────────────────┘

Read-After-Write Consistency Problem:
┌──────────────────────────────────────────────────────────────────────────┐
│  T=0: User submits order, app writes to PRIMARY → Success               │
│  T=0.1s: App redirects user to order confirmation page                   │
│  T=0.2s: App reads from REPLICA to display order → NOT FOUND (stale!)   │
│  T=1.2s: Order finally visible on replica                                │
│                                                                          │
│  User experience: "Your order was placed... wait, order not found!?"     │
└──────────────────────────────────────────────────────────────────────────┘

Solution Pattern - Route Read-After-Write to Primary:
┌──────────────────────────────────────────────────────────────────────────┐
│  Write Path:                                                             │
│  ├─ All writes → Primary endpoint                                        │
│  └─ Immediate read after write → Primary endpoint (consistent read)      │
│                                                                          │
│  Read Path (eventually consistent okay):                                 │
│  ├─ Analytics queries → Replica endpoint                                 │
│  ├─ Report generation → Replica endpoint                                 │
│  └─ Search/browse (no fresh write) → Replica endpoint                    │
└──────────────────────────────────────────────────────────────────────────┘

Monitoring Replica Lag:
┌──────────────────────────────────────────────────────────────────────────┐
│  CloudWatch Metric: ReplicaLag (in seconds)                              │
│                                                                          │
│  ReplicaLag   │                                                          │
│   (seconds)   │                   ▲                                      │
│    60 ────────│                  ╱│╲                                     │
│    50         │                 ╱ │ ╲                                    │
│    40         │                ╱  │  ╲                                   │
│    30 - - - - │ Alarm threshold   │   ╲ ╲                               │
│    20         │              ╱    │    ╲ ╲                              │
│    10         │    ╱╲   ╱╲╱      │     ╲ ╲___                           │
│     0 ────────├───┴──┴─┴─────────┴──────┴─────> Time                    │
│                                                                          │
│  Normal: 0-5 seconds                                                     │
│  Warning: 5-30 seconds (may impact user experience)                      │
│  Critical: > 30 seconds (investigate immediately)                        │
│  Growing lag: Replica cannot keep up with write volume                   │
└──────────────────────────────────────────────────────────────────────────┘
```

### Core Separation of Concerns

```
What RDS manages for you:           What you still own:
├── Hardware provisioning           ├── Schema design & indexing
├── OS patching (maintenance win.)  ├── Query optimization
├── DB engine installation/upgrade  ├── Access control (DB users)
├── Automated backups               ├── Data lifecycle decisions
├── HA failover (Multi-AZ)          ├── RPO/RTO requirements
├── Storage scaling (autoscaling)   ├── Application connection logic
└── Monitoring integration          └── DR testing and validation
```

### Choose your configuration in three independent steps

```
Step 1: Engine         MySQL · PostgreSQL · MariaDB · Oracle · SQL Server
Step 2: Deployment     Single-AZ · Multi-AZ instance · Multi-AZ cluster · Read Replica
Step 3: Storage/Size   gp2 · gp3 · io1 · io2 · Instance class (db.t3 → db.r7g)
```

**Exam Tip:** These three decisions are independent. A Multi-AZ deployment does not automatically add read replicas, and adding a read replica does not improve HA.

---

## 2. Deployment Types

### Comparison Table

| Deployment | Topology | Replication | Failover Time | Serves Reads? | Primary Use |
|---|---|---|---|---|---|
| **Single-AZ instance** | One instance | None | Manual restore only | Primary only | Dev/Test, non-critical |
| **Multi-AZ DB instance** | Primary + 1 standby (different AZ) | Synchronous | ~60 seconds | Standby does **not** serve reads | Production HA |
| **Multi-AZ DB cluster** | 1 writer + 2 readers (3 AZs) | Semi-synchronous | ~35 seconds | Both readers serve reads | Production HA + read scale |
| **Read Replica** | Primary + 1–5 async copies | Asynchronous | Promotion required (manual) | Yes | Read scaling, DR |

---

### Deployment Type 1: Single-AZ Instance

```
        Availability Zone A
       ┌────────────────────┐
       │  ┌──────────────┐  │
       │  │  DB Instance │  │
       │  │  (Primary)   │  │
       │  └──────────────┘  │
       │                    │
       │  EBS Storage       │
       └────────────────────┘

No standby. Failure = manual restore from backup.
Replication: None
Serves reads: Primary only
Failover: None (restore to new instance from backup)
```

- Use for: Dev/Test, non-critical workloads, cost optimization
- Not suitable for: Production systems requiring high availability

---

### Deployment Type 2: Multi-AZ DB Instance

```
        Availability Zone A            Availability Zone B
       ┌────────────────────┐         ┌────────────────────┐
       │  ┌──────────────┐  │         │  ┌──────────────┐  │
       │  │  DB Instance │  │─sync──► │  │  Standby DB  │  │
       │  │  (Primary)   │  │repl.    │  │  (No reads!) │  │
       │  └──────────────┘  │         │  └──────────────┘  │
       │  EBS Storage       │         │  EBS Storage       │
       └────────────────────┘         └────────────────────┘
              │
     Application connects to
     single DNS endpoint.
     On failover, DNS flips to
     standby (~60 seconds).

Replication: SYNCHRONOUS (data written to both before commit returns)
Failover time: ~60 seconds
Serves reads: Primary only (standby does NOT serve reads)
```

- **Key point:** The standby is a warm-standby only, not a read replica. It exists only for failover.
- Synchronous replication ensures zero data loss on failover (RPO ≈ 0).
- On failover, DNS record TTL matters — applications should use low TTL and retry logic.

---

### Deployment Type 3: Multi-AZ DB Cluster

```
  Availability Zone A    Availability Zone B    Availability Zone C
 ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
 │  ┌────────────┐  │   │  ┌────────────┐  │   │  ┌────────────┐  │
 │  │  Writer    │  │──►│  │  Reader 1  │  │   │  │  Reader 2  │  │
 │  │  Instance  │  │   │  │ (serves    │  │   │  │ (serves    │  │
 │  │            │  │──►│  │  reads)    │  │   │  │  reads)    │  │
 │  └────────────┘  │   │  └────────────┘  │   │  └────────────┘  │
 └──────────────────┘   └──────────────────┘   └──────────────────┘
          │                      ▲                       ▲
          └──── semi-sync ───────┴───────────────────────┘
                replication (ack from at least 1 reader before commit)

Replication: SEMI-SYNCHRONOUS (writer waits for at least 1 reader to acknowledge)
Failover time: ~35 seconds (faster than DB instance because a reader is already in sync)
Serves reads: YES — both readers serve queries
Supported engines: MySQL 8.0.28+, PostgreSQL 13.4+
```

- **Key point:** Readers serve real read traffic. This is different from Multi-AZ DB *instance* standby.
- Faster failover than Multi-AZ DB instance because a reader is already caught up.
- Use the reader endpoint to distribute reads across both readers automatically.

---

### Deployment Type 4: Read Replica

```
      Primary DB Instance                   Read Replica
     ┌────────────────────┐               ┌────────────────────┐
     │  ┌──────────────┐  │               │  ┌──────────────┐  │
     │  │  DB (Writer) │  │──async──────► │  │ DB (Reader)  │  │
     │  └──────────────┘  │  replication  │  └──────────────┘  │
     └────────────────────┘               └────────────────────┘
                                           Separate DB endpoint.
                                           May lag behind primary.
                                           Cross-region possible.
                                           Can be promoted to standalone.

Replication: ASYNCHRONOUS (replica may lag; possible data loss on promotion)
Failover time: Manual promotion required (not automatic)
Serves reads: YES
Cross-region: YES (useful for DR or local read latency reduction)
```

- Up to 5 read replicas per source instance (engine-dependent)
- Replication lag is a real concern — monitor `ReplicaLag` CloudWatch metric
- Promoting a replica makes it a standalone instance; replication is severed
- Cross-region read replicas require enabling automated backups on the source

---

## 3. Read Replicas and Consistency

### How Async Replication Works

```
Write path:
App → Writer → Committed to primary storage
                    │
                    └── Async copy ──► Replica (may arrive after commit returns to app)

Read-after-write problem:
1. App writes user record to primary
2. App immediately reads from replica
3. Replica hasn't received the write yet → stale read

Solution: Route read-after-write operations to the writer endpoint or add retry logic.
```

### CLI: Create a Read Replica

```bash
# Create a read replica in the same region
aws rds create-db-instance-read-replica \
  --db-instance-identifier orders-db-replica \
  --source-db-instance-identifier orders-db \
  --db-instance-class db.r6g.large \
  --availability-zone us-east-1b \
  --publicly-accessible false \
  --tags Key=Environment,Value=production

# Create a cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier orders-db-replica-eu \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:orders-db \
  --db-instance-class db.r6g.large \
  --region eu-west-1 \
  --destination-region eu-west-1
```

### Monitoring Replication Lag

```bash
# Check ReplicaLag metric via CLI
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name ReplicaLag \
  --dimensions Name=DBInstanceIdentifier,Value=orders-db-replica \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 60 \
  --statistics Average
```

**ReplicaLag** is measured in seconds. A value growing over time indicates:
- Long-running transactions on the primary blocking replication
- Replica instance is undersized (CPU/IO can't keep up with write volume)
- High write throughput saturating the replica's storage I/O
- Replication errors (check DB error logs)

**Set a CloudWatch alarm:** Alert when ReplicaLag > 30 seconds for production replicas used in read paths.

### Promote Read Replica for DR

Use this procedure when promoting a cross-region replica after a primary region failure:

```bash
# Step 1: Promote the read replica to a standalone DB instance
# (This breaks replication — it becomes its own writer)
aws rds promote-read-replica \
  --db-instance-identifier orders-db-replica-eu \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-03:00"

# Step 2: Wait for promotion to complete (status: available)
aws rds wait db-instance-available \
  --db-instance-identifier orders-db-replica-eu

# Step 3: Get the new endpoint
aws rds describe-db-instances \
  --db-instance-identifier orders-db-replica-eu \
  --query "DBInstances[0].Endpoint"

# Step 4: Update application connection strings to point to new endpoint
# (Update Secrets Manager, SSM Parameter Store, or environment config)

# Step 5: Validate data integrity
# Run read/write smoke tests against the promoted instance
# Check for any replication lag that existed before promotion
```

**Warning:** Data written to the primary *after* the last replicated transaction is permanently lost. Replication lag at promotion time equals your data loss. This is your RPO.

### When NOT to Use Read Replicas for Scaling

```
Read replicas help with:                  Read replicas do NOT help with:
├── Offloading analytics/reports          ├── Write bottlenecks
├── Reducing read latency                 ├── Lock contention
├── Serving different geographic regions  ├── Missing indexes on the writer
└── DR (with promotion procedure)         └── Slow queries that also run on replicas
```

---

## 4. Backups, Restore, and DR

### Backup Mechanisms Overview

| Mechanism | RPO | RTO | Retention | What it does NOT solve |
|---|---|---|---|---|
| **Automated backups + transaction logs** | Near-zero (to any second in retention window) | 20 min – hours (depends on DB size + warmup) | 1–35 days (configurable) | Immediate service continuity; must restore to new instance |
| **Manual snapshot** | Point-in-time of snapshot creation | 20 min – hours | Until manually deleted | Continuous point-in-time history between snapshots |
| **Multi-AZ failover** | ~0 (synchronous replication) | ~35–60 seconds | N/A (availability mechanism) | Logical corruption, regional failure |
| **Cross-region read replica** | Seconds to minutes (async lag) | Minutes (promotion procedure) | N/A | Automatic failover; requires manual promotion |
| **Cross-region snapshot copy** | Point-in-time of copy | Hours (restore + warmup) | Until deleted | Near-zero RPO |

**RPO** = maximum acceptable data loss (how much data can you afford to lose?)  
**RTO** = maximum acceptable downtime (how long can the system be unavailable?)

"We have snapshots" answers neither without restore tests and retention details.

### Automated Backup Window

```
Backup window and maintenance window are separate 30-minute windows.
They must NOT overlap.

Best practice:
├── Backup window: Low-traffic period (e.g., 02:00–02:30 UTC)
├── Maintenance window: After backup window (e.g., 03:00–03:30 UTC)
└── Buffer time between them: Avoid back-to-back

During automated backup:
├── Slight I/O suspension possible for Single-AZ (brief performance dip)
├── Multi-AZ: Backup taken from standby — no I/O impact on primary
└── Transaction logs captured continuously between daily snapshots
```

### CLI: Create a Manual Snapshot

```bash
# Create a manual DB snapshot
aws rds create-db-snapshot \
  --db-instance-identifier orders-db \
  --db-snapshot-identifier orders-db-snapshot-2024-01-15

# Wait for snapshot to be available
aws rds wait db-snapshot-available \
  --db-snapshot-identifier orders-db-snapshot-2024-01-15

# List available snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier orders-db \
  --query "DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]"
```

### CLI: Restore to Point in Time

```bash
# Restore to a specific point in time (creates a NEW DB instance)
aws rds restore-db-instance-to-point-in-time \
  --source-db-instance-identifier orders-db \
  --target-db-instance-identifier orders-db-restored-jan15 \
  --restore-time "2024-01-15T14:30:00Z" \
  --db-instance-class db.r6g.large \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --no-publicly-accessible

# Check restore progress
aws rds describe-db-instances \
  --db-instance-identifier orders-db-restored-jan15 \
  --query "DBInstances[0].[DBInstanceStatus,Endpoint]"
```

### CLI: Copy a Snapshot Cross-Region

```bash
# Copy snapshot from us-east-1 to eu-west-1 for DR
aws rds copy-db-snapshot \
  --source-db-snapshot-identifier arn:aws:rds:us-east-1:123456789012:snapshot:orders-db-snapshot-2024-01-15 \
  --target-db-snapshot-identifier orders-db-snapshot-2024-01-15-eu \
  --kms-key-id arn:aws:kms:eu-west-1:123456789012:key/your-key-id \
  --region eu-west-1

# Verify copy status
aws rds describe-db-snapshots \
  --db-snapshot-identifier orders-db-snapshot-2024-01-15-eu \
  --region eu-west-1 \
  --query "DBSnapshots[0].[Status,PercentProgress]"
```

### Important Restore Caveats

```
Restores always create a NEW DB instance — not in-place replacement.
Plan for:
├── New DNS endpoint → Update connection strings
├── DB parameter group may default back → Re-apply custom settings
├── Security groups may need re-applying
├── Storage warmup → Restored EBS volumes may have higher latency
│   until all data blocks are accessed (lazy loading)
├── Application validation before routing traffic
└── DNS TTL of old endpoint → Ensure stale connections drain first
```

---

## 5. Connections, Failover, and RDS Proxy

### The Failover Problem

```
Before failover:                      After failover (Multi-AZ):
App → DNS → Primary (AZ-A)           App → DNS → New Primary (AZ-B, was standby)

Problem: DNS TTL caching
├── App has stale cached DNS → still tries to connect to old (dead) primary
├── Existing open connections to old primary are broken
└── Connection pool holds dead connections until timeout

Solution requirements:
├── DNS re-resolution on reconnect (use short TTL, 5–30s)
├── Bounded retry with exponential backoff
├── Idempotent transaction handling
└── Detect broken connections quickly (TCP keepalive, connection validation)
```

### RDS Proxy Architecture

```
                    APPLICATION TIER
           ┌────────────────────────────────┐
           │  Many short-lived connections  │
           │  (Lambda, ECS tasks, threads)  │
           └───────────────────┬────────────┘
                               │ Many connections
                               ▼
           ┌────────────────────────────────┐
           │           RDS PROXY            │
           │  ┌──────────────────────────┐  │
           │  │  Connection Pool         │  │
           │  │  Multiplexing Layer      │  │◄── Stores credentials in
           │  │  Idle connection mgmt    │  │    Secrets Manager
           │  └──────────────────────────┘  │
           └──────────┬─────────────────────┘
                      │ Few persistent connections
                      ▼
           ┌────────────────────────────────┐
           │         RDS DB INSTANCE        │
           │  (Primary / Writer endpoint)   │
           └────────────────────────────────┘

Benefits:
├── Reduces DB connection overhead (especially with Lambda)
├── Faster failover: Proxy absorbs failover, app sees brief pause not error
├── Credential rotation: Proxy fetches fresh secrets from Secrets Manager
└── IAM authentication: Proxy handles token validation
```

### Session Pinning

Session pinning occurs when RDS Proxy cannot multiplex a connection — it pins the client connection to a specific DB connection for the duration of the session.

```
Multiplexing (efficient):             Session pinning (reduces efficiency):
Client A ─┐                          Client A ──────► DB Connection 1 (pinned)
Client B ─┤─► DB Connection 1        Client B ──────► DB Connection 2 (pinned)
Client C ─┘                          Client C ──────► DB Connection 3 (pinned)

Proxy can reuse one DB connection     Each client holds its own DB connection
for multiple clients.                 → no multiplexing benefit.
```

**Triggers for session pinning:**
- `SET` statements that change session variables (e.g., `SET autocommit = 0`)
- Multi-statement transactions held open across requests
- Prepared statements
- Temp tables and cursors
- `LOCK TABLE` statements
- MySQL: `sql_mode` changes

**How to reduce pinning:** Avoid session-state-changing SQL outside transactions. Use connection-level settings in the application config rather than `SET` statements.

### CLI: Create an RDS Proxy

```bash
# Create RDS Proxy (requires an IAM role and a Secrets Manager secret)
aws rds create-db-proxy \
  --db-proxy-name orders-db-proxy \
  --engine-family MYSQL \
  --auth '[
    {
      "AuthScheme": "SECRETS",
      "SecretArn": "arn:aws:secretsmanager:us-east-1:123456789012:secret:rds/orders-db-credentials",
      "IAMAuth": "REQUIRED"
    }
  ]' \
  --role-arn arn:aws:iam::123456789012:role/RDSProxyRole \
  --vpc-subnet-ids subnet-0123456789 subnet-9876543210 \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --require-tls \
  --idle-client-timeout 1800

# Register the DB instance as the proxy target
aws rds register-db-proxy-targets \
  --db-proxy-name orders-db-proxy \
  --db-instance-identifiers orders-db

# Get the proxy endpoint to use in your application
aws rds describe-db-proxies \
  --db-proxy-name orders-db-proxy \
  --query "DBProxies[0].Endpoint"
```

### Lambda + RDS Proxy Pattern

Lambda creates a new execution environment (and new DB connection) for every concurrent invocation. At scale, this exhausts `max_connections`.

```
Without RDS Proxy:                    With RDS Proxy:
1000 concurrent Lambdas               1000 concurrent Lambdas
        │                                      │
        ▼ 1000 connections                     ▼ 1000 connections
   RDS DB (max_connections                 RDS PROXY
   often < 500 for small                       │
   instance sizes)                             ▼ ~20 persistent connections
   ❌ Connection refused                   RDS DB
                                          ✅ Healthy
```

**Implementation pattern:**
1. Create RDS Proxy in the same VPC as the DB
2. Configure Lambda to run in the same VPC (attach to same subnets + security groups that can reach proxy)
3. Point Lambda connection string to proxy endpoint instead of DB endpoint
4. Use IAM authentication: Lambda assumes an IAM role → generates auth token → Proxy validates via IAM
5. Reuse the database connection across Lambda invocations (connection is held in the proxy pool, not in Lambda's container — benefit comes from the proxy side, not from reusing in Lambda memory)

---

## 6. Security

### Network Security

```
Correct security posture:
┌──────────────────────────────────────────────────────┐
│  VPC                                                  │
│  ┌─────────────────────┐   ┌────────────────────┐   │
│  │  App Tier (private) │──►│ DB Subnet Group    │   │
│  │  SG: app-sg         │   │ (private subnets)  │   │
│  └─────────────────────┘   │ RDS SG: only allow │   │
│                             │ port 3306/5432     │   │
│                             │ from app-sg        │   │
│                             └────────────────────┘   │
└──────────────────────────────────────────────────────┘

DB subnet group ≠ security. You also need:
├── Security group inbound rule: source = app security group ID (not CIDR)
├── Route table: no route to IGW from DB subnets
├── "Publicly Accessible" = false (even if SG is restrictive, avoid exposure)
└── VPC flow logs for auditing connection patterns
```

### IAM Database Authentication

IAM DB auth lets applications authenticate to MySQL and PostgreSQL RDS using short-lived tokens generated by AWS IAM, instead of a static database password.

```
How it works:
1. Application calls AWS API: generate-db-auth-token
2. IAM validates the IAM principal's identity and permissions
3. RDS returns a 15-minute auth token (signed string)
4. Application connects to RDS using username + token as password
5. RDS validates token against IAM — grants connection if valid
6. Token expires after 15 minutes (but open connection persists)

Supported engines: MySQL, PostgreSQL (not Oracle, SQL Server, MariaDB)
Max connections via IAM auth: 200 per second (per DB)
```

**CLI: Generate a DB Auth Token**

```bash
# Generate a 15-minute auth token for MySQL
aws rds generate-db-auth-token \
  --hostname orders-db.cxyz.us-east-1.rds.amazonaws.com \
  --port 3306 \
  --region us-east-1 \
  --username iam_user

# Use token to connect (e.g., with mysql client)
RDSHOST="orders-db.cxyz.us-east-1.rds.amazonaws.com"
TOKEN=$(aws rds generate-db-auth-token \
  --hostname $RDSHOST --port 3306 \
  --region us-east-1 --username iam_user)

mysql -h $RDSHOST -u iam_user --password=$TOKEN \
  --ssl-ca=/path/to/rds-ca-2019-root.pem
```

**IAM Policy required to use IAM DB auth:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "rds-db:connect",
      "Resource": "arn:aws:rds-db:us-east-1:123456789012:dbuser:db-ABCDEFGHIJKL01234/iam_user"
    }
  ]
}
```

**Important:** IAM authentication handles *authentication* only. The DB user must still have SQL `GRANT` privileges for the databases/tables it needs.

### TLS Enforcement

```bash
# For MySQL: set require_secure_transport = ON in the parameter group
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-mysql-params \
  --parameters "ParameterName=require_secure_transport,ParameterValue=ON,ApplyMethod=immediate"

# For PostgreSQL: set ssl = 1 and rds.force_ssl = 1
aws rds modify-db-parameter-group \
  --db-parameter-group-name my-pg-params \
  --parameters \
    "ParameterName=rds.force_ssl,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=ssl,ParameterValue=1,ApplyMethod=immediate"

# Apply the updated parameter group to the DB instance
aws rds modify-db-instance \
  --db-instance-identifier orders-db \
  --db-parameter-group-name my-mysql-params \
  --apply-immediately
```

Download the AWS RDS CA bundle for certificate verification from:  
`https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`

### Encryption at Rest

```
Encryption at rest covers:
├── DB instance storage (EBS volumes)
├── Automated backups
├── Manual snapshots
├── Read replicas (if source is encrypted, replica is encrypted)
└── Transaction logs in S3

What it does NOT cover:
├── Data in transit (use TLS separately)
└── DB-level encryption of individual columns (application-side concern)

CRITICAL CONSTRAINT:
You CANNOT encrypt an unencrypted RDS instance in-place.

Workaround to encrypt an existing unencrypted instance:
1. Take a snapshot of the unencrypted instance
2. Copy the snapshot, enabling encryption during copy (specify KMS key)
3. Restore the encrypted snapshot to a new encrypted DB instance
4. Update application to point to new encrypted instance
5. Decommission the old unencrypted instance

aws rds copy-db-snapshot \
  --source-db-snapshot-identifier unencrypted-snapshot \
  --target-db-snapshot-identifier encrypted-snapshot \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-key-id
```

**KMS key considerations:**
- Default AWS-managed key (`aws/rds`) works but you cannot rotate it manually
- Customer-managed CMK gives you full key control and audit trail via CloudTrail
- If you delete the KMS key, you permanently lose access to all encrypted data
- Cross-region snapshot copy requires a KMS key *in the destination region*

### Secrets Manager for Credential Rotation

```
Pattern:
App → Secrets Manager (get secret) → DB password
                │
         Automatic rotation
         Lambda rotates password
         on schedule (e.g., 30 days)

Benefit: No hardcoded passwords. Rotation is transparent to the app
         if it fetches the secret on each connection (or caches briefly).
```

---

## 7. Performance and Scaling

### Investigation Order

```
Performance problem → investigate in this order:

1. Workload changes      → Did traffic increase? New query patterns?
2. Slow SQL / query plans → EXPLAIN, Database Insights top SQL
3. Locks / long txns     → Show processlist, pg_stat_activity, wait events
4. CPU / Memory          → CPUUtilization, FreeableMemory CloudWatch metrics
5. Storage latency/IOPS  → ReadLatency, WriteLatency, DiskQueueDepth
6. Connections / network → DatabaseConnections, NetworkReceiveThroughput

Do NOT jump to "add a read replica" or "scale up" before steps 1–3.
```

### Storage Types Comparison

| Storage Type | Max IOPS | Max Throughput | Best For | Burst Behavior |
|---|---|---|---|---|
| **gp2** (General Purpose SSD) | 16,000 | 250 MB/s | General workloads, small DBs | Yes — IOPS burst based on volume size (3 IOPS/GB, min 100, max 3,000) |
| **gp3** (General Purpose SSD v2) | 64,000 | 4,000 MB/s | Most workloads (recommended default) | No burst — baseline IOPS configurable independently from storage size |
| **io1** (Provisioned IOPS SSD) | 64,000 | 2,000 MB/s | I/O-intensive, consistent low latency | No burst — guaranteed provisioned IOPS |
| **io2** (Provisioned IOPS SSD v2) | 256,000 | 4,000 MB/s | Highest performance, mission-critical | No burst — guaranteed provisioned IOPS, higher durability |

**gp3 is the recommended default** for new workloads. It decouples IOPS from storage size, letting you provision exactly the IOPS you need without over-provisioning storage.

**Avoid gp2 for new deployments:** gp2 ties IOPS to volume size (3 IOPS/GB), so you may end up paying for more storage than needed just to get sufficient IOPS.

### Storage Autoscaling

```
How it works:
├── Enable autoscaling with a maximum storage threshold
├── RDS monitors free storage space
├── When free storage < 10% (or < 5 GB) for 5 minutes, autoscaling triggers
├── Storage increases by the larger of: 5 GB or 10% of current size
└── New size cannot exceed the maximum threshold you set

aws rds modify-db-instance \
  --db-instance-identifier orders-db \
  --max-allocated-storage 500 \
  --apply-immediately
```

**When to be careful with storage autoscaling:**
- It is one-way — storage can only grow, never shrink
- Scaling during a large data load can cause unexpected cost increases
- If you're approaching maximum threshold and need to grow further, update the max
- Monitor `FreeStorageSpace` metric and set an alarm before autoscaling triggers unexpectedly

### Enhanced Monitoring vs CloudWatch Metrics

| Feature | CloudWatch Metrics | Enhanced Monitoring |
|---|---|---|
| **Granularity** | 1-minute minimum | 1-second to 60-second |
| **Source** | Hypervisor-level | OS agent on the DB instance |
| **Visibility** | Instance-level metrics | Per-process OS metrics (CPU, memory per process) |
| **Storage cost** | Included | Stored in CloudWatch Logs (cost based on retention) |
| **Use case** | Dashboards, alarms, trends | Diagnosing noisy neighbor, OS-level bottlenecks |
| **Setup** | Automatic | Enable per instance, requires IAM role |

**Key distinction:** CloudWatch CPU metrics come from the hypervisor and may not accurately reflect DB process CPU. Enhanced Monitoring gives you the actual OS view, including `%guest`, `%iowait`, and per-process breakdowns.

### Database Insights (Replaces Performance Insights)

**AWS is retiring Performance Insights on July 31, 2026.** Use Database Insights in all current answers.

```
Database Insights Query Analysis Workflow:

1. Navigate to CloudWatch → Database Insights
2. Select the DB instance
3. View DB Load graph:
   ├── Y-axis: Average Active Sessions (AAS)
   ├── X-axis: Time
   └── Color-coded by wait event type (I/O, lock, CPU, etc.)

4. Identify the dominant wait event during peak:
   ├── io/file/sql/FTS → I/O bottleneck → scale storage, add IOPS
   ├── lock/table_lock  → Lock contention → find blocking queries
   ├── CPU              → Query optimization needed
   └── wait/io/socket   → Network or connection saturation

5. Drill into Top SQL:
   ├── Sort by "Load" or "Average latency"
   ├── Click a SQL digest to see execution plan
   └── Compare before/after with index changes

6. Standard mode: 7-day retention, sampled metrics
   Advanced mode: 1-year retention, SQL plan capture, anomaly detection
```

---

## 8. Observability

### Key Metrics to Monitor

| Metric | Namespace | What It Indicates | Alert Threshold |
|---|---|---|---|
| `CPUUtilization` | AWS/RDS | CPU pressure on DB instance | > 80% sustained |
| `FreeableMemory` | AWS/RDS | Available RAM (low = pressure) | < 256 MB (tune per instance) |
| `DatabaseConnections` | AWS/RDS | Active connections to DB | > 80% of `max_connections` |
| `ReadLatency` | AWS/RDS | Average read I/O latency | > 20ms (tune per workload) |
| `WriteLatency` | AWS/RDS | Average write I/O latency | > 20ms |
| `DiskQueueDepth` | AWS/RDS | I/O requests waiting in queue | > 1 sustained |
| `FreeStorageSpace` | AWS/RDS | Remaining storage | < 20% of allocated |
| `ReplicaLag` | AWS/RDS | Replication lag in seconds | > 30s for production replicas |
| `FailoverTime` | AWS/RDS | Duration of last failover | Alert on any failover event |
| `NetworkReceiveThroughput` | AWS/RDS | Inbound network to DB | Compare vs instance network limit |

```bash
# List recent DB events (last 24 hours = 1440 minutes)
aws rds describe-events \
  --source-type db-instance \
  --source-identifier orders-db \
  --duration 1440

# Describe a specific DB instance
aws rds describe-db-instances \
  --db-instance-identifier orders-db

# Describe snapshots for a DB
aws rds describe-db-snapshots \
  --db-instance-identifier orders-db

# Describe a cluster
aws rds describe-db-clusters \
  --db-cluster-identifier orders-cluster
```

---

## 9. Common Patterns

### Pattern 1: Multi-AZ Web Application Database with Read Replicas

**Use case:** Production web application with both HA and read scaling requirements.

```
Architecture:
┌─────────────────────────────────────────────────────────────────┐
│  Application Load Balancer                                       │
│         │                                                        │
│  ┌──────┴────────┐                                              │
│  │  App Servers  │  (read-write) → Writer endpoint               │
│  │  (EC2/ECS)    │  (read-only)  → Reader endpoint               │
│  └───────────────┘                                              │
└─────────────────────────────────────────────────────────────────┘
         │ Writer                          │ Reader
         ▼                                ▼
  Multi-AZ DB instance             Read Replica(s)
  (Primary + Standby)              (same or cross-region)
  └─ Synchronous replication       └─ Async replication
  └─ ~60s failover                 └─ ReplicaLag monitoring needed
```

**Implementation notes:**
- Use separate connection strings in application: `WRITE_DB_URL` and `READ_DB_URL`
- For write endpoint: Multi-AZ DB instance or Multi-AZ DB cluster endpoint
- For read endpoint: Read replica endpoint or Multi-AZ cluster reader endpoint
- Enable RDS Proxy if your application uses short-lived connections (ECS tasks, Lambda-like patterns)

```bash
# Create Multi-AZ DB instance
aws rds create-db-instance \
  --db-instance-identifier orders-db \
  --db-instance-class db.r6g.large \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password "$(aws secretsmanager get-random-password \
    --password-length 32 --query RandomPassword --output text)" \
  --allocated-storage 100 \
  --storage-type gp3 \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/your-key-id \
  --multi-az \
  --db-subnet-group-name my-db-subnet-group \
  --vpc-security-group-ids sg-0123456789abcdef0 \
  --backup-retention-period 7 \
  --preferred-backup-window "02:00-02:30" \
  --preferred-maintenance-window "sun:03:00-sun:03:30" \
  --deletion-protection \
  --no-publicly-accessible
```

---

### Pattern 2: Cross-Region Disaster Recovery with Read Replica Promotion

**Use case:** Business continuity if the primary AWS region becomes unavailable.

```
Normal operation:
us-east-1 (Primary)               eu-west-1 (DR)
┌──────────────────┐              ┌──────────────────┐
│  Multi-AZ DB     │──async──────►│  Cross-region    │
│  (orders-db)     │  replication │  Read Replica    │
└──────────────────┘              └──────────────────┘

Disaster (us-east-1 unavailable):
eu-west-1 (now Primary after DR activation)
┌──────────────────────────────────────────────────────────────┐
│  1. Promote cross-region replica to standalone writer        │
│  2. Update application DNS/connection strings to eu-west-1   │
│  3. Re-enable Multi-AZ in eu-west-1 for HA                   │
│  4. When us-east-1 recovers, create new replica from eu-west-1│
└──────────────────────────────────────────────────────────────┘
```

**DR runbook steps:**
1. `aws rds promote-read-replica --db-instance-identifier orders-db-replica-eu`
2. Wait for status `available`
3. Update Secrets Manager / Route 53 / connection strings
4. Validate writes and reads on promoted instance
5. Enable Multi-AZ on the promoted instance for HA in eu-west-1
6. Document and test this runbook regularly (never test DR for the first time during an actual incident)

**RPO reality check:** If replication lag was 45 seconds at the time of the disaster, you lose 45 seconds of writes. Factor this into your RPO requirement before choosing this pattern.

---

### Pattern 3: Lambda Application Using RDS Proxy

**Use case:** Serverless or highly concurrent applications that would exhaust DB connections without a proxy.

```
Problem without proxy:
Lambda (concurrency: 500) → 500 new DB connections per burst
  └─ db.t3.medium max_connections ≈ 90 → Connection refused

Solution with proxy:
Lambda (concurrency: 500) → RDS Proxy (multiplexes) → ~20 DB connections
  └─ DB stays healthy

IAM auth flow:
1. Lambda execution role has rds-db:connect permission
2. Lambda calls generate-db-auth-token (15-min token)
3. Lambda connects to proxy endpoint with username + token
4. Proxy validates token via IAM, forwards connection to DB
5. Proxy multiplexes multiple Lambda connections to fewer DB connections
```

**Python (boto3) example:**

```python
import boto3
import pymysql
import os

def get_db_auth_token():
    rds_client = boto3.client('rds', region_name=os.environ['AWS_REGION'])
    token = rds_client.generate_db_auth_token(
        DBHostname=os.environ['DB_HOST'],
        Port=3306,
        DBUsername=os.environ['DB_USER'],
        Region=os.environ['AWS_REGION']
    )
    return token

def lambda_handler(event, context):
    connection = pymysql.connect(
        host=os.environ['DB_HOST'],  # RDS Proxy endpoint
        user=os.environ['DB_USER'],
        password=get_db_auth_token(),
        database='orders',
        ssl_ca='/var/task/rds-ca-2019-root.pem',
        connect_timeout=10
    )
    try:
        with connection.cursor() as cursor:
            cursor.execute("SELECT COUNT(*) FROM orders WHERE status = 'pending'")
            result = cursor.fetchone()
        return {'pending_orders': result[0]}
    finally:
        connection.close()
```

---

### Pattern 4: Blue/Green Deployment for Engine Upgrade or Schema Change

**Use case:** Zero-downtime or minimal-downtime engine version upgrade or schema migration.

```
Blue/Green deployment managed by RDS:

BLUE (current production)           GREEN (new version / schema)
┌─────────────────────┐            ┌─────────────────────┐
│  orders-db          │────sync───►│  orders-db-green    │
│  MySQL 8.0.32       │ (logical   │  MySQL 8.0.35       │
│  (production)       │  repl.)    │  (staging)          │
└─────────────────────┘            └─────────────────────┘
         │                                   │
         ▼ Traffic: 100%                     ▼ Traffic: 0%
   App connections                     No app connections yet

Switchover:
1. Test green environment thoroughly
2. RDS flushes replication lag, quiesces blue, switches over
3. Green becomes production (blue is now standby)
4. Update app connection strings (or use Route 53 CNAME swap)
5. Blue remains available for rollback (up to 14 days)

CLI:
aws rds create-blue-green-deployment \
  --blue-green-deployment-name orders-upgrade \
  --source arn:aws:rds:us-east-1:123456789012:db:orders-db \
  --target-engine-version 8.0.35

# When ready to cut over:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier bgd-XXXXXXXXX \
  --switchover-timeout 300
```

**When to use blue/green:**
- Major version engine upgrades where in-place upgrade is risky
- Schema migrations that require testing before cutover
- Situations where rollback in < 60 seconds is a requirement
- Not available for all engines/versions — check AWS documentation

---

## 10. Troubleshooting

### Symptom → Root Cause Mapping

| Symptom | Likely Root Causes | How to Diagnose | Fix |
|---|---|---|---|
| **Connection timeout** | SG rule missing; wrong port; DB not started; max_connections reached; DNS resolution failure | `telnet <endpoint> 3306`, check SG inbound rules, check `DatabaseConnections` metric | Open correct SG port from app SG, increase max_connections or add proxy |
| **Authentication failure** | Wrong username/password; IAM auth token expired; Secrets Manager rotation in progress; clock skew (IAM auth) | Check Secrets Manager current secret version; verify IAM token generation | Fetch fresh secret; re-generate IAM token; fix NTP sync |
| **Read returns stale data** | Connected to replica endpoint instead of writer; high ReplicaLag; application-level cache is stale | Check which endpoint is in connection string; check `ReplicaLag` metric | Route read-after-write to writer endpoint; add lag check before reads |
| **CPU low but DB slow** | Lock contention; I/O wait; connection queue depth; storage credit exhaustion (gp2 burst) | Check `DiskQueueDepth`, wait events in Database Insights; show processlist | Find blocking query; kill long-running transactions; upgrade to gp3 |
| **High CPU** | Missing index (full table scan); N+1 queries; cartesian join; sudden traffic spike | Database Insights Top SQL; EXPLAIN on slow queries; query digest | Add index; refactor query; add caching layer |
| **Performance after failover** | Stale connection pool; DNS caching; new primary's buffer pool cold (needs warmup) | Check app logs for connection errors; check failover event in RDS Events | Restart connection pool; DNS re-resolution; buffer pool warmup takes minutes |
| **Replica lag grows** | Long-running transactions on primary; primary write volume too high; replica instance too small; DDL statements | Check `ReplicaLag` trend; compare replica instance class to primary; check for ALTER TABLE | Upgrade replica instance; break large transactions; schedule DDL in off-peak |
| **Storage full** | Data growth exceeded provisioned size; autoscaling max threshold hit; logs/temp files growing | Check `FreeStorageSpace` metric; check table sizes; check temp space | Increase storage or max-autoscaling limit; purge old data; archive |
| **Failover took longer than expected** | DNS TTL too long in application; connection pool not clearing dead connections; heavy transaction in-flight | Check application retry logs; check DNS TTL configured; check RDS Events | Reduce DNS TTL; enable proper retry/timeout in connection pool |
| **Snapshot restore is slow** | EBS lazy loading (blocks read from S3 until accessed); large DB size | `FreeStorageSpace` after restore shows correct size; I/O is high | Warm up by reading all data: `pg_prewarm` or full table scan |

### CLI Inspection Commands

```bash
# Describe DB instance state, endpoint, status
aws rds describe-db-instances \
  --db-instance-identifier orders-db

# Check recent events (failures, failovers, backups)
aws rds describe-events \
  --source-type db-instance \
  --source-identifier orders-db \
  --duration 1440 \
  --query "Events[*].[Date,Message]"

# List snapshots
aws rds describe-db-snapshots \
  --db-instance-identifier orders-db \
  --query "DBSnapshots[*].[DBSnapshotIdentifier,SnapshotCreateTime,Status]"

# Describe pending modifications
aws rds describe-db-instances \
  --db-instance-identifier orders-db \
  --query "DBInstances[0].PendingModifiedValues"

# Check parameter group settings
aws rds describe-db-parameters \
  --db-parameter-group-name my-mysql-params \
  --query "Parameters[?ParameterName=='max_connections']"
```

---

## 11. Quick Reference Card

### Deployment Type Comparison

| | Single-AZ | Multi-AZ DB Instance | Multi-AZ DB Cluster | Read Replica |
|---|---|---|---|---|
| **# Instances** | 1 | 2 (1 primary + 1 standby) | 3 (1 writer + 2 readers) | 1 per replica |
| **Replication** | None | Synchronous | Semi-synchronous | Asynchronous |
| **Failover** | Manual restore | ~60 seconds (auto) | ~35 seconds (auto) | Manual promotion |
| **Standby serves reads?** | N/A | ❌ No | ✅ Yes | ✅ Yes |
| **Data loss on failover** | Data in backup | ~0 (synchronous) | ~0 (semi-sync) | Seconds to minutes of lag |
| **Use for** | Dev/Test | Production HA | Production HA + reads | Read scale, DR |
| **Cost** | 1x instance | ~2x instance | ~3x instance | 1x per replica |

### Storage Type Comparison

| Type | IOPS | Throughput | Burst | Best For | Avoid When |
|---|---|---|---|---|---|
| **gp2** | 3 IOPS/GB (100–16,000) | 128–250 MB/s | Yes (bucket-based) | Legacy workloads | New deployments |
| **gp3** | 3,000–64,000 (independent) | 125–4,000 MB/s | No | Most workloads (default) | — |
| **io1** | 1,000–64,000 | 2,000 MB/s | No | I/O-intensive, consistent perf | Cost-sensitive |
| **io2** | 1,000–256,000 | 4,000 MB/s | No | Mission-critical, highest perf | — |

### Key Metrics Reference

| Metric | What It Tells You | Alert If |
|---|---|---|
| `CPUUtilization` | Processing pressure | > 80% sustained 15 min |
| `FreeableMemory` | Available RAM | < 256 MB |
| `DatabaseConnections` | Active connections | > 80% of max_connections |
| `ReadLatency` / `WriteLatency` | Storage I/O speed | > 20ms |
| `DiskQueueDepth` | I/O queue depth | > 1 sustained |
| `FreeStorageSpace` | Remaining space | < 20% of allocated |
| `ReplicaLag` | Replication delay | > 30 seconds |
| `BurstBalance` | gp2 IOPS burst credits | < 20% (upgrade to gp3) |
| `NetworkReceiveThroughput` | Inbound DB network | Nearing instance limit |

### Common Parameter Group Settings

| Parameter | Engine | Effect | Recommended Value |
|---|---|---|---|
| `max_connections` | MySQL | Maximum concurrent connections | `{DBInstanceClassMemory/12582880}` (formula for dynamic sizing) |
| `innodb_buffer_pool_size` | MySQL | Memory for InnoDB cache | 75% of total RAM (set via formula in RDS) |
| `innodb_flush_log_at_trx_commit` | MySQL | Durability vs performance | `1` (safest, default), `2` (performance, risk on crash) |
| `slow_query_log` | MySQL | Log slow queries | `1` (enable for tuning) |
| `long_query_time` | MySQL | Threshold for slow query log | `1` (seconds) |
| `shared_buffers` | PostgreSQL | Buffer pool size | 25% of RAM (RDS manages automatically) |
| `log_min_duration_statement` | PostgreSQL | Log slow queries (ms) | `1000` (log queries > 1s) |
| `rds.force_ssl` | PostgreSQL | Require TLS connections | `1` |
| `require_secure_transport` | MySQL | Require TLS connections | `ON` |

```bash
# View current effective value for a parameter
aws rds describe-db-parameters \
  --db-parameter-group-name my-mysql-params \
  --query "Parameters[?ParameterName=='max_connections'].[ParameterName,ParameterValue,IsModifiable]"
```

---

## 12. Exam Tips Summary

### Must-Know Concepts

1. **Multi-AZ DB instance standby does NOT serve reads.** The standby exists solely for failover. "Multi-AZ never serves reads" applies to the *DB instance* deployment only — the Multi-AZ *DB cluster* readers do serve reads.

2. **Multi-AZ is for availability; Read Replicas are for read scaling.** These are separate capabilities. Don't confuse them on the exam.

3. **Restores always create a new DB instance.** Point-in-time restore, snapshot restore, and cross-region restore all produce a new endpoint. Plan for connection string updates.

4. **You cannot encrypt an unencrypted RDS instance in-place.** The process is: snapshot → copy with encryption → restore encrypted snapshot to new instance.

5. **RDS Proxy improves connection management but does NOT increase DB compute capacity.** It helps Lambda and other bursty workloads avoid connection exhaustion. It does not make queries run faster.

6. **Session pinning reduces RDS Proxy multiplexing benefits.** SET statements, open transactions, temp tables, and cursors pin a client to a specific DB connection.

7. **IAM database authentication uses 15-minute tokens.** It handles authentication only. SQL GRANT privileges are still required separately.

8. **ReplicaLag is asynchronous replication delay.** Growing lag → replica may serve stale reads. Monitor with CloudWatch metric `ReplicaLag`.

9. **Blue/green deployment allows rollback.** The blue environment stays available for up to 14 days after switchover. Use for risky schema migrations or major version upgrades.

10. **gp3 is the recommended default storage type.** It decouples IOPS from storage size unlike gp2. Choose io1/io2 only when you need very high, consistent IOPS beyond gp3 limits.

11. **Cross-region snapshot copy requires a KMS key in the destination region.** If the source is encrypted with a CMK, the destination CMK must exist in the target region.

12. **Performance Insights is being retired July 31, 2026.** Use Database Insights in all current answers. Advanced mode retains SQL data for 1 year; Standard mode for 7 days.

13. **DB subnet group ≠ security.** You need subnet group + correct security group rules + no IGW route from DB subnets + `PubliclyAccessible=false` for a properly secured database.

14. **Failover time for Multi-AZ DB cluster (~35s) is faster than DB instance (~60s)** because a reader is already caught up via semi-synchronous replication.

### Common Exam Scenarios

| Scenario | Correct Answer |
|---|---|
| Need production HA with automatic failover | Multi-AZ DB instance or Multi-AZ DB cluster |
| Need to scale reads for a reporting workload | Add read replica(s) |
| Lambda function getting "too many connections" | RDS Proxy |
| Need cross-region DR with manageable RPO | Cross-region read replica + promotion runbook |
| Encrypt existing unencrypted RDS instance | Snapshot → copy with encryption → restore |
| Minimize failover disruption for microservices | RDS Proxy (absorbs failover reconnect) |
| Upgrade MySQL version with rollback option | Blue/green deployment |
| Application needs fresh reads after a write | Route to writer endpoint, not reader |
| Backups not working after Multi-AZ failover | Check backup window; Multi-AZ uses standby for backups |
| Storage IOPS degraded on gp2 | BurstBalance depleted; migrate to gp3 |

### Red Flags in Exam Questions

- ❌ **"Multi-AZ standby serves reads"** → Only true for Multi-AZ *DB cluster* readers, not the DB *instance* standby
- ❌ **"Read replica provides HA"** → No automatic failover; promotion is manual
- ❌ **"Encrypt existing RDS in-place"** → Not possible; requires snapshot-copy-restore
- ❌ **"RDS Proxy makes queries faster"** → It manages connections, not query execution
- ❌ **"Snapshot = continuous backup"** → Snapshots are point-in-time; PITR requires automated backups + transaction logs
- ❌ **"Multi-AZ protects against accidental data deletion"** → Synchronous replication replicates deletes too; use PITR
- ❌ **"Performance Insights for new architecture"** → Use Database Insights (PI end-of-life July 2026)
- ❌ **"Replica lag is zero"** → Async replication always has some lag; only synchronous Multi-AZ standby has near-zero lag
- ❌ **"RDS restore to same instance"** → Restore always creates a new DB instance

---

## 13. Interview Questions and Model Answers

**Q1: What is the difference between Multi-AZ and a Read Replica?**

Multi-AZ is a high-availability feature. The standby instance (in a DB instance deployment) receives synchronous replication from the primary and exists solely for failover — it never serves reads. Failover is automatic in ~60 seconds. A Read Replica uses asynchronous replication, serves read queries, and requires manual promotion for DR. They solve different problems: Multi-AZ = availability, Read Replica = read throughput. You can use both simultaneously.

The Multi-AZ DB *cluster* blurs this line slightly — its two reader instances serve reads and also participate in semi-synchronous replication, giving both HA and read scaling. But even there, the terminology is distinct from a read replica.

---

**Q2: When does RDS Proxy help, and when doesn't it?**

RDS Proxy helps when you have many short-lived connections (Lambda, ECS tasks, serverless patterns) that would exhaust the DB's `max_connections`. It multiplexes thousands of app connections into a smaller pool of persistent DB connections. It also smooths over failover — the proxy reconnects to the new primary while applications see a brief pause rather than a hard connection error.

RDS Proxy does NOT help with: slow queries (the proxy just passes SQL through), write bottlenecks, missing indexes, or lock contention. It also does not help if session pinning reduces multiplexing (SET statements, open transactions, temp tables).

---

**Q3: How do you encrypt an existing unencrypted RDS instance?**

You cannot do it in-place. The procedure is:
1. Take a manual snapshot of the unencrypted instance
2. Copy the snapshot, specifying a KMS key during the copy — this produces an encrypted snapshot
3. Restore the encrypted snapshot to a new DB instance
4. Validate data integrity and application functionality on the new instance
5. Update connection strings to point to the new (encrypted) instance
6. Decommission the old unencrypted instance

Plan for downtime during the cutover step, or use blue/green deployment to reduce it.

---

**Q4: What are RPO and RTO, and how do RDS backup mechanisms map to them?**

RPO (Recovery Point Objective) is the maximum acceptable data loss — "how much data can we afford to lose?" RTO (Recovery Time Objective) is the maximum acceptable downtime — "how long can we be down?"

- **Multi-AZ failover:** RPO ≈ 0 (synchronous), RTO ≈ 35–60 seconds
- **Automated backups + PITR:** RPO = seconds (continuous log capture), RTO = 20 min to hours (depends on DB size and warmup)
- **Manual snapshot:** RPO = time since last snapshot, RTO = 20 min to hours
- **Cross-region replica promotion:** RPO = replication lag at time of failure, RTO = minutes (promotion time)

"We have daily snapshots" means your RPO could be up to 24 hours and your RTO could be hours. Always test your actual restore time; API completion does not mean the database is production-ready.

---

**Q5: Why does replica lag grow?**

Replica lag grows when the replica cannot process changes as fast as the primary generates them. Common causes:

- **Long-running transactions on primary:** A large `UPDATE` or `DELETE` produces a huge binary log event; the replica must replay it in full
- **Replica instance undersized:** If the replica is a smaller instance class than the primary, it may not have enough CPU/IO to keep up with write volume
- **High write throughput:** Bulk loads or batch jobs overwhelm the replica's ability to replay changes
- **DDL statements:** `ALTER TABLE` on a large table can block replication on the replica
- **Replication errors:** A constraint violation or error stops replication entirely (lag grows to infinity until resolved)

Investigate with `ReplicaLag` CloudWatch metric + DB error logs + Database Insights.

---

**Q6: When would you use a blue/green deployment for schema changes?**

Use blue/green when:
- You need the ability to roll back quickly if the schema change causes issues
- The engine version upgrade or schema migration is risky enough that you want to test the green environment under production-like conditions before cutting over
- Your business requires zero or near-zero downtime for a major version upgrade

How it works: RDS creates a copy (green) of your production DB (blue), then uses logical replication to keep green in sync. You can run tests on green. When ready, RDS switches over (quiesces blue, flushes replication, promotes green). Blue stays around for up to 14 days as a rollback option.

Not suitable for: simple minor version patches (in-place is fine), or when logical replication is not supported for your engine version.

---

**Q7: How does IAM database authentication differ from regular password auth?**

With IAM DB auth, instead of storing a database password, the application calls the AWS API (`generate-db-auth-token`) to get a 15-minute signed token. That token is used as the database password. The RDS instance validates it against IAM.

Benefits: no long-lived passwords in Secrets Manager or config files; credential rotation is handled by IAM; token expiry limits blast radius of leaked tokens; full CloudTrail audit trail of token generation.

Limitations: supported on MySQL and PostgreSQL only; max 200 IAM auth connections per second; token must be regenerated before it expires (though the open connection persists); the DB user still needs SQL `GRANT` privileges — IAM auth is authentication only.

---

**Q8: What happens to existing connections during a Multi-AZ failover?**

Existing open connections to the primary instance are terminated. The DNS endpoint name stays the same, but its target changes to point at the new primary (former standby) after ~30–60 seconds.

Applications must handle this with:
1. **Short DNS TTL:** Ensure the application re-resolves DNS on reconnect (don't cache DNS indefinitely)
2. **Connection validation:** Connection pool should test connections before using them (e.g., `testOnBorrow=true`)
3. **Retry logic:** Bounded retry with exponential backoff on connection failures
4. **Transaction handling:** Detect that an in-flight transaction was interrupted; retry with idempotency logic

RDS Proxy significantly improves this: the proxy reconnects to the new primary while keeping the application's TCP connection alive (applications see a brief pause, not a hard failure).

---

**Q9: How is Multi-AZ DB cluster different from Multi-AZ DB instance?**

| | Multi-AZ DB Instance | Multi-AZ DB Cluster |
|---|---|---|
| Instances | 1 writer + 1 standby | 1 writer + 2 readers |
| Replication | Synchronous | Semi-synchronous |
| Readers serve traffic? | No (standby is passive) | Yes (both readers active) |
| Failover time | ~60 seconds | ~35 seconds |
| Engines | All supported engines | MySQL 8.0.28+, PostgreSQL 13.4+ |

The cluster is better for workloads needing both HA and read scaling. It's also faster to fail over because a reader is already current with the writer via semi-sync replication. The DB instance standby is purely passive — it exists only for failover.

---

**Q10: How would you design regional DR for an RDS database?**

1. **Define RPO and RTO** first — these drive all subsequent decisions
2. **Replication mechanism:** Cross-region read replica for near-continuous replication (RPO = replication lag). Cross-region snapshot copies for lower frequency, higher RPO tolerance
3. **Provision dependencies in DR region:** VPC, subnets, security groups, KMS keys, Secrets Manager secrets, IAM roles — all must exist before you need them
4. **Promotion runbook:** Document and automate the promotion procedure. Test it regularly (quarterly at minimum)
5. **Application failover:** Update DNS (Route 53 health checks + failover routing), connection strings (Secrets Manager), or use a global load balancer
6. **Failback plan:** After the primary region recovers, decide whether to fail back or make DR the new primary
7. **Key operational detail:** The KMS key for encrypted replicas must exist in the DR region. Snapshot copies require a destination-region KMS key

---

**Q11: Why would you recommend Database Insights over Performance Insights?**

Performance Insights is being retired by AWS on July 31, 2026. Database Insights is the current service and its successor. For any production architecture designed today, you should use Database Insights.

Beyond lifecycle, Database Insights offers Advanced mode with 1-year SQL data retention (vs PI's maximum), anomaly detection, and tighter CloudWatch integration. Standard mode provides 7-day retention at lower cost. Always verify which mode is configured and what data is actually retained before promising specific query history depth.

---

**Q12: A developer says "we can just add a read replica to fix our slow checkout." How do you respond?**

Ask whether checkout writes data or requires read-after-write consistency. Checkout almost certainly writes (order creation, payment processing, inventory updates) and needs immediate consistency on read-after-write. A read replica uses asynchronous replication, so reading from a replica immediately after a write may return stale data.

Also, the replica would need to run the same slow queries that are slow on the writer — if the problem is a missing index or a poorly written query, the replica won't fix it. It would just have the same problem with less load.

The right order: investigate the actual slow query, check for missing indexes or lock contention, add caching for read-heavy idempotent operations, then consider a read replica for genuine read-only workloads (reporting, analytics) that can tolerate eventual consistency.

---

## 14. AWS Sources

- [Multi-AZ deployment types](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html)
- [Multi-AZ DB cluster overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/multi-az-db-clusters-concepts.html)
- [Multi-AZ cluster read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_MultiAZDBCluster_ReadRepl.html)
- [Working with read replicas](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ReadRepl.html)
- [Backups overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.html)
- [Backup retention](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_WorkingWithAutomatedBackups.BackupRetention.html)
- [Point-in-time restore](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIT.html)
- [RDS Proxy overview](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html)
- [IAM database authentication](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.IAMDBAuth.html)
- [Encrypting RDS resources](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)
- [Storage types for RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_Storage.html)
- [Storage autoscaling](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_PIOPS.StorageTypes.html#USER_PIOPS.Autoscaling)
- [Enhanced Monitoring](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_Monitoring.OS.html)
- [CloudWatch Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html)
- [Blue/green deployments for RDS](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/blue-green-deployments.html)

**Related:** [Aurora](aws-aurora-cheatsheet.md) · [KMS](aws-kms-cheatsheet.md) · [Lambda](aws-lambda-cheatsheet.md) · [IAM](aws-iam-cheatsheet.md)
