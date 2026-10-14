# Amazon EC2 Cheat Sheet

**Interview preparation: compute, storage, networking, scaling, and failure recovery.**

**Verified:** 2026-10-13 against the AWS sources linked below. Instance availability and adjustable quotas vary by Region and account; verify them for a concrete design.

[All AWS cheatsheets](aws-core-services-interview-index.md) · [VPC networking](aws-vpc-networking-cheatsheet.md) · [IAM](aws-iam-cheatsheet.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Choosing compute](#2-choosing-compute)
- [3. Purchasing options and capacity](#3-purchasing-options-and-capacity)
- [4. Storage and lifecycle](#4-storage-and-lifecycle)
- [5. Networking and security](#5-networking-and-security)
- [6. Scaling and resilience](#6-scaling-and-resilience)
- [7. Observability and troubleshooting](#7-observability-and-troubleshooting)
- [8. Practical inspection](#8-practical-inspection)
- [9. Common patterns](#9-common-patterns)
- [10. Quick reference card](#10-quick-reference-card)
- [11. Exam tips summary](#11-exam-tips-summary)
- [12. Interview Q&A](#12-interview-qa)
- [13. Quick revision](#13-quick-revision)
- [14. AWS sources and further reading](#14-aws-sources-and-further-reading)

---

## 1. Fundamentals and mental model

EC2 provides virtual machines. AWS operates the physical infrastructure; you normally own the guest OS, patches, application, identity configuration, and recovery design.

```text
Region → Availability Zone → subnet → network interface → EC2 instance
                                         │                  ├── AMI + instance type
                                         ├── security group ├── EBS / instance store
                                         └── private IP     └── IAM instance profile
```

An instance lives in one AZ. High availability comes from multiple instances across AZs, normally behind a load balancer with an Auto Scaling group (ASG).

| Component | What it answers | Interview trap |
|---|---|---|
| AMI | Which boot image and software? | An image is not a running server |
| Instance type | Which CPU, memory, network, and storage capabilities? | Bigger CPU does not fix storage bottlenecks |
| Launch template | How should replacement instances launch? | Update the template and roll out replacements |
| Instance profile | Which IAM role can application code use? | Security groups grant no AWS API permissions |
| User data | What initial bootstrap should run? | Do not embed secrets; make bootstrap repeatable |
| ASG | How many instances and how are failures replaced? | Scaling and traffic distribution are separate |

## 2. Choosing compute

| Workload | Starting point | Validate |
|---|---|---|
| General web/application servers | General-purpose family | CPU, memory, p95 latency |
| CPU-heavy encoding or calculations | Compute-optimized | Sustained CPU, instruction set |
| Large in-memory working set | Memory-optimized | Resident memory, garbage collection |
| Local high-throughput scratch data | Storage-optimized | Data-loss tolerance and I/O pattern |
| Model training/inference | Accelerated computing | GPU/accelerator compatibility and quotas |
| Low average CPU with short bursts | Burstable family | Credit balance and surplus-credit costs |

Benchmark Graviton/Arm versus x86 with the actual binaries and dependencies. Select a family and size from measured constraints; do not memorize one instance as the universal best choice.

## 3. Purchasing options and capacity

| Option | Main benefit | Main caveat |
|---|---|---|
| On-Demand | No long-term usage commitment | Capacity is not guaranteed |
| Savings Plans | Discount for an hourly spend commitment | Does not reserve capacity |
| Regional Reserved Instance | Billing discount for matching usage | Does not reserve AZ capacity |
| Zonal Reserved Instance | Discount and matching AZ capacity reservation | More constrained placement |
| On-Demand Capacity Reservation | Reserve matching capacity in an AZ | Unused reservation can still cost money |
| Spot | Discounted spare capacity | Work must tolerate interruption |
| Dedicated Instance | Hardware tenancy isolation | Not the same host-level control as Dedicated Host |
| Dedicated Host | Physical host visibility/control for licensing | Capacity and utilization need planning |

**Answer pattern:** separate *discount commitment* from *capacity assurance*. For a critical launch, discuss capacity reservations; for steady usage, discuss discounts. Compatible discounts can apply to reserved capacity. [AWS purchasing guide](https://docs.aws.amazon.com/decision-guides/latest/decision-guides/ec2-purchasing-options-aws-how-to-choose.html)

## 4. Storage and lifecycle

| Storage | Persistence and scope | Typical use |
|---|---|---|
| EBS | Persistent block volume in one AZ | Boot disks and databases |
| Instance store | Host-local; lost on stop/termination or host loss | Scratch/cache with rebuildable data |
| EFS | Shared managed file storage | Shared files across instances |
| S3 | Regional object storage | Objects, backups, data lakes |

EBS snapshots enable recovery or volume creation elsewhere; they do not make an attached volume multi-AZ. Multi-Attach is a specialized capability, not permission to mount an ordinary filesystem concurrently without coordination.

| Operation | Key consequence |
|---|---|
| Reboot | Same host; instance-store data normally remains |
| Stop/start | EBS remains; instance-store data is lost; auto-assigned public IPv4 can change |
| Hibernate | RAM saved to EBS on supported configurations; instance store still lost |
| Terminate | Instance cannot restart; volume fate follows `DeleteOnTermination` |

Private addresses generally survive stop/start. Stopped instances can still incur EBS and public-IP-related charges. Check volume attributes instead of assuming every disk is deleted. [Instance lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)

## 5. Networking and security

- A public subnet has a route to an internet gateway. An IPv4 instance also needs a public address and permitted traffic to use that path directly.
- A private instance can use NAT for outbound internet access or VPC endpoints for supported services. A NAT gateway does not accept unsolicited inbound application connections.
- Security groups are stateful allow rules attached to network interfaces. Network ACLs are stateless subnet controls; consider return traffic and ephemeral ports.
- Prefer Session Manager for administration where suitable. Confirm agent, IAM, and network access before removing other access paths.
- Give workloads a role through an instance profile; use SDK credential providers instead of embedding access keys.
- Require IMDSv2 where compatible. Its session token protects metadata access, but does not replace least privilege or application security. [IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

## 6. Scaling and resilience

1. Put stateless application instances in at least two AZs.
2. Store shared state outside replaceable instances.
3. Use an ALB/NLB and meaningful application health checks.
4. Configure ASG health evaluation, minimum/desired/maximum capacity, warmup, and scaling policy.
5. Deploy through versioned launch templates and instance refresh; keep rollback images.
6. Drain traffic and finish/checkpoint work during scale-in or interruption.

Target tracking tries to hold a utilization target. Scheduled scaling handles known demand; predictive scaling can prepare for patterns. Scaling on CPU alone is weak when the bottleneck is queue age or downstream capacity.

For Spot, diversify eligible types and AZs and use interruption-aware workers. Do not make correctness depend on receiving an interruption warning in time.

## 7. Observability and troubleshooting

| Symptom | Investigate first |
|---|---|
| Cannot connect | Instance state, routes, address, SG/NACL, OS firewall, service binding |
| System status check failure | Underlying infrastructure; recovery/replacement path |
| Instance status check failure | Guest OS, disk, memory, network configuration |
| Slow despite low CPU | EBS latency/throughput, network, locks, dependency latency |
| ASG replaces healthy-looking instances | Health-check source, grace period, app startup time |
| Scale-out cannot launch | vCPU quota, AZ capacity, launch template, AMI and IAM errors |
| Works until stop/start | Ephemeral data or auto-assigned public-IP assumptions |

EC2 platform metrics do not automatically include guest memory and filesystem utilization; collect them with an agent. Monitor application latency and error rate alongside infrastructure saturation.

## 8. Practical inspection

Read-only examples; replace sample identifiers and set the intended Region/profile.

```bash
aws ec2 describe-instances --instance-ids i-0123456789abcdef0
aws ec2 describe-instance-status --instance-ids i-0123456789abcdef0 --include-all-instances
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names app-asg
aws ec2 describe-volumes --filters Name=attachment.instance-id,Values=i-0123456789abcdef0
```

Check launch-template version, subnet/AZ distribution, role, metadata options, disk deletion flags, and the reason in scaling activities.

---

## 9. Common Patterns

### Pattern 1: Auto Scaling with Mixed Instances (Spot + On-Demand)
```bash
# Create launch template
aws ec2 create-launch-template \
  --launch-template-name web-app \
  --version-description "v1" \
  --launch-template-data '{
    "ImageId": "ami-0c55b159cbfafe1f0",
    "InstanceType": "t3.medium",
    "SecurityGroupIds": ["sg-12345"],
    "IamInstanceProfile": {"Arn": "arn:aws:iam::account:instance-profile/WebAppRole"},
    "UserData": "<base64-encoded-script>",
    "MetadataOptions": {"HttpTokens": "required"}
  }'

# Create ASG with mixed instances policy
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-app-asg \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 4 \
  --vpc-zone-identifier "subnet-a,subnet-b,subnet-c" \
  --target-group-arns arn:aws:elasticloadbalancing:region:account:targetgroup/web-app/abc123 \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --mixed-instances-policy '{
    "LaunchTemplate": {
      "LaunchTemplateSpecification": {
        "LaunchTemplateName": "web-app",
        "Version": "$Latest"
      },
      "Overrides": [
        {"InstanceType": "t3.medium"},
        {"InstanceType": "t3a.medium"},
        {"InstanceType": "t2.medium"}
      ]
    },
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 2,
      "OnDemandPercentageAboveBaseCapacity": 25,
      "SpotAllocationStrategy": "price-capacity-optimized"
    }
  }'
```

**Result:** 2 On-Demand (baseline), rest 75% Spot / 25% On-Demand → ~60-70% cost savings

---

### Pattern 2: Blue/Green Deployment with Instance Refresh
```bash
# Step 1: Update launch template with new AMI
aws ec2 create-launch-template-version \
  --launch-template-name web-app \
  --source-version $Latest \
  --version-description "v2-blue-green" \
  --launch-template-data '{"ImageId": "ami-NEW-VERSION"}'

# Step 2: Start instance refresh (gradual rollout)
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name web-app-asg \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [25, 50, 75],
    "CheckpointDelay": 600
  }'

# Step 3: Monitor progress
aws autoscaling describe-instance-refreshes \
  --auto-scaling-group-name web-app-asg

# Step 4: Rollback if issues (cancel refresh + revert template)
aws autoscaling cancel-instance-refresh --auto-scaling-group-name web-app-asg
aws ec2 modify-launch-template --launch-template-name web-app --default-version v1
```

**Timeline:**
```
T+0:    Start refresh (4 instances on v1)
T+5min: 25% replaced (1 on v2, 3 on v1)
T+15min: 50% replaced (2 on v2, 2 on v1)
T+25min: 75% replaced (3 on v2, 1 on v1)
T+35min: 100% replaced (4 on v2)
```

---

### Pattern 3: Scheduled Scaling for Predictable Load
```bash
# Scale up for business hours
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name web-app-asg \
  --scheduled-action-name scale-up-morning \
  --recurrence "0 8 * * MON-FRI" \
  --min-size 10 \
  --max-size 50 \
  --desired-capacity 20

# Scale down after hours
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name web-app-asg \
  --scheduled-action-name scale-down-evening \
  --recurrence "0 18 * * MON-FRI" \
  --min-size 2 \
  --max-size 10 \
  --desired-capacity 4

# Weekend minimal capacity
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name web-app-asg \
  --scheduled-action-name scale-down-weekend \
  --recurrence "0 0 * * SAT" \
  --min-size 1 \
  --max-size 5 \
  --desired-capacity 2
```

**Cost Savings:** ~50-70% (4 instances vs 20 during off-hours)

---

### Pattern 4: Target Tracking with Multiple Metrics
```python
# Scale based on ALB request count per target
import boto3

autoscaling = boto3.client('autoscaling')

autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-app-asg',
    PolicyName='scale-on-request-count',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ALBRequestCountPerTarget',
            'ResourceLabel': 'app/my-alb/abc123/targetgroup/my-tg/def456'
        },
        'TargetValue': 1000.0,  # 1000 requests per instance
        'ScaleInCooldown': 300,
        'ScaleOutCooldown': 60
    }
)

# Additionally scale on CPU
autoscaling.put_scaling_policy(
    AutoScalingGroupName='web-app-asg',
    PolicyName='scale-on-cpu',
    PolicyType='TargetTrackingScaling',
    TargetTrackingConfiguration={
        'PredefinedMetricSpecification': {
            'PredefinedMetricType': 'ASGAverageCPUUtilization'
        },
        'TargetValue': 70.0
    }
)
```

**Behavior:** ASG scales based on whichever metric needs more capacity

---

### Pattern 5: Spot Instance Interruption Handling
```python
# EC2 user data script to handle Spot interruption
#!/bin/bash

# Poll for spot interruption notice
while true; do
    TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" 2>/dev/null)
    SPOT_TERMINATION=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/spot/instance-action 2>/dev/null)
    
    if [ -n "$SPOT_TERMINATION" ]; then
        echo "Spot interruption detected at $(date)"
        
        # Graceful shutdown
        # 1. Drain from load balancer
        INSTANCE_ID=$(curl -H "X-aws-ec2-metadata-token: $TOKEN" -s http://169.254.169.254/latest/meta-data/instance-id)
        ALB_TG_ARN="arn:aws:elasticloadbalancing:region:account:targetgroup/web-app/abc123"
        
        aws elbv2 deregister-targets \
            --target-group-arn $ALB_TG_ARN \
            --targets Id=$INSTANCE_ID
        
        sleep 30  # Allow in-flight requests to complete
        
        # 2. Save state / checkpoint work
        /app/scripts/checkpoint-state.sh
        
        # 3. Stop application gracefully
        systemctl stop myapp
        
        exit 0
    fi
    
    sleep 5
done &
```

---

### Pattern 6: Cross-AZ Disaster Recovery with ASG
```bash
# Primary region ASG (us-east-1)
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name primary-asg \
  --launch-template LaunchTemplateName=app,Version=$Latest \
  --min-size 3 \
  --max-size 10 \
  --desired-capacity 6 \
  --vpc-zone-identifier "subnet-us-east-1a,subnet-us-east-1b,subnet-us-east-1c" \
  --health-check-type ELB \
  --health-check-grace-period 300

# DR region ASG (eu-west-1) - minimal warm standby
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name dr-asg \
  --launch-template LaunchTemplateName=app-eu,Version=$Latest \
  --min-size 1 \
  --max-size 10 \
  --desired-capacity 1 \
  --vpc-zone-identifier "subnet-eu-west-1a,subnet-eu-west-1b" \
  --region eu-west-1

# Failover: Update DR ASG capacity
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name dr-asg \
  --min-size 3 \
  --max-size 10 \
  --desired-capacity 6 \
  --region eu-west-1

# Update Route 53 to point to DR ALB
```

---

### Pattern 7: Custom AMI Pipeline with Automation
```bash
# Build AMI from running instance
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "web-app-v2.1.0-$(date +%Y%m%d-%H%M%S)" \
  --description "Web app release 2.1.0" \
  --no-reboot  # or omit to ensure consistency

# Copy AMI to DR region
NEW_AMI_ID=$(aws ec2 describe-images --owners self --filters "Name=name,Values=web-app-v2.1.0*" --query 'Images[0].ImageId' --output text)

aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id $NEW_AMI_ID \
  --name "web-app-v2.1.0-dr" \
  --region eu-west-1

# Automate with EventBridge + Lambda
# Lambda triggers on new AMI creation → copies to DR region → updates launch template
```

---

## 10. Quick Reference Card

### Instance Type Selection Guide
```
┌──────────────────────────────────────────────────────┐
│ Workload                   Family                    │
├──────────────────────────────────────────────────────┤
│ Web servers, small DBs     T3, T3a (burstable)       │
│ General apps               M6i, M6a, M6g (Graviton)   │
│ CPU-intensive              C6i, C6a, C6g (compute)    │
│ Memory-intensive (cache)   R6i, R6a, R6g, X2 (memory)│
│ Storage-intensive (data)   I4i, D3 (storage)          │
│ GPU (ML training)          P4, P5 (accelerated)       │
│ GPU (inference)            G5, Inf2 (accelerated)     │
│ Arm workloads              M6g, C6g, R6g (Graviton)   │
└──────────────────────────────────────────────────────┘
```

---

### Purchasing Options Comparison
| Option | Discount | Capacity Guarantee | Commitment | Interruptible |
|--------|----------|-------------------|------------|---------------|
| **On-Demand** | 0% | ❌ | None | ❌ |
| **Savings Plan** | 40-60% | ❌ | 1 or 3 years | ❌ |
| **Reserved (Regional)** | 40-60% | ❌ | 1 or 3 years | ❌ |
| **Reserved (Zonal)** | 40-60% | ✅ | 1 or 3 years | ❌ |
| **Capacity Reservation** | 0% (can combine with SP/RI) | ✅ | On-demand or commitment | ❌ |
| **Spot** | 70-90% | ❌ | None | ✅ |

---

### EBS Volume Types
| Type | IOPS | Throughput | Use Case | Cost |
|------|------|------------|----------|------|
| **gp3** | 3K-16K | 125-1000 MB/s | General (cheaper than gp2) | $$  |
| **gp2** | 3K-16K (burst) | 250 MB/s | Legacy general | $$  |
| **io2 Block Express** | 256K | 4000 MB/s | Ultra-high performance | $$$$ |
| **io2** | 64K | 1000 MB/s | High IOPS databases | $$$  |
| **st1** | 500 | 500 MB/s | Throughput-optimized (HDD) | $   |
| **sc1** | 250 | 250 MB/s | Cold HDD (infrequent access) | $   |

**Rule of Thumb:** Use gp3 for most workloads (better cost/performance than gp2)

---

### Instance Lifecycle States
```
    Pending → Running → Stopping → Stopped → Terminated
                |         ↓                      ↑
                |     (Hibernated)               |
                |_______________________________|
                        (Terminate)

Key Points:
- Reboot: Same host, instance store preserved
- Stop/Start: New host, instance store LOST, new public IP
- Hibernate: RAM → EBS, instance store LOST
- Terminate: Cannot restart, DeleteOnTermination applies
```

---

### Auto Scaling Metrics
```python
# Best Metrics for Target Tracking (in priority order):
1. Application-level: Request count per target, queue depth
2. Resource saturation: CPU, memory (with agent)
3. Network: Bytes in/out (for network-bound apps)

# Avoid:
❌ Fixed schedule only (can't handle unexpected load)
❌ CPU alone (doesn't capture queue backlog or external bottlenecks)
```

---

### Security Best Practices Checklist
```
✅ Use IMDSv2 (HttpTokens: required)
✅ IAM instance profile (not access keys in userdata)
✅ Security groups: least-privilege, named rules
✅ Private subnets for app tier (public only for bastion/ALB)
✅ Systems Manager Session Manager (no SSH keys)
✅ Encrypted EBS volumes (KMS)
✅ CloudWatch Logs agent (centralized logging)
✅ Patch management (Systems Manager Patch Manager)
✅ No public IPs on database/backend instances
✅ VPC endpoints for AWS services (avoid NAT Gateway cost)
```

---

### Cost Optimization Quick Wins
```python
# 1. Right-size: use Compute Optimizer recommendations
aws compute-optimizer get-ec2-instance-recommendations

# 2. Stop unused instances (dev/test after hours)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# 3. Use Spot for fault-tolerant workloads (70-90% savings)
# 4. Graviton instances (20% better price/performance vs Intel)
# 5. gp3 instead of gp2 (20% cheaper, better baseline)
# 6. Delete unused EBS snapshots
# 7. Use Reserved Instances/Savings Plans for steady workloads
```

**Monthly Cost Example (us-east-1):**
- m6i.large On-Demand: ~$70
- m6i.large Spot: ~$10-20 (depends on availability)
- m6g.large (Graviton): ~$55
- m6i.large 1-year RI: ~$40

---

## 11. Exam Tips Summary

### Key Concepts for AWS Exams

1. **Instance Lifecycle**
   - **Stop/Start:** EBS persists, instance store lost, new public IP
   - **Reboot:** Everything persists, same host, same IPs
   - **Terminate:** Deleted, DeleteOnTermination for volumes
   - **Hibernate:** RAM→EBS, faster startup, still loses instance store

2. **Purchasing Options**
   - **Savings Plans:** Flexibility (any region, any size), no capacity guarantee
   - **Reserved (Zonal):** Capacity reserved in specific AZ
   - **Spot:** Can be interrupted with 2-minute warning
   - **Dedicated Host:** Physical server control (for licensing)

3. **Auto Scaling**
   - ASG scales instances; ELB distributes traffic (separate concerns)
   - Health check types: EC2 (instance status) vs ELB (application health)
   - Grace period: Time before health checks start (for app startup)
   - Scaling policies: Target tracking (recommended), step, simple

4. **Storage**
   - **EBS:** Block storage, persists, single AZ, snapshots to S3
   - **Instance Store:** Host-local, ephemeral, lost on stop
   - **EFS:** Shared NFS, multi-AZ, concurrent access
   - **S3:** Object storage, 11 9s durability

5. **Networking**
   - **Public subnet:** Route to IGW, public IP needed for internet access
   - **Private subnet:** No IGW route, use NAT Gateway for outbound
   - **Security Groups:** Stateful, allow rules only, instance-level
   - **NACLs:** Stateless, allow+deny rules, subnet-level

6. **IMDSv2 (Instance Metadata Service v2)**
   - Requires session token (HttpTokens: required)
   - Protects against SSRF attacks
   - Recommended for all new instances

7. **Placement Groups**
   - **Cluster:** Low latency (same AZ, same rack) - HPC
   - **Spread:** Separate hardware (max 7 per AZ) - HA
   - **Partition:** Groups on separate racks - Big Data (Hadoop, Kafka)

8. **Elastic IP**
   - Static public IPv4
   - Free when attached to running instance
   - Charged when unattached or attached to stopped instance

9. **EC2 Status Checks**
   - **System:** AWS infrastructure (power, network, host)
   - **Instance:** Guest OS, disk, memory, network config

10. **Tenancy**
    - **Shared (default):** Multiple customers on same hardware
    - **Dedicated Instance:** Your instances isolated on hardware
    - **Dedicated Host:** Visibility/control of physical host

---

### Scenario-Based Exam Questions

**Scenario 1:** Application needs to survive single AZ failure.
- **Solution:** ASG across 3 AZs, ALB, min capacity ≥ 2
- **Why:** Single instance in one AZ = single point of failure

**Scenario 2:** Batch processing can be interrupted and resumed.
- **Solution:** Spot Instances with checkpointing
- **Why:** 70-90% cost savings, work tolerates interruption

**Scenario 3:** Need capacity guarantee for product launch.
- **Solution:** On-Demand Capacity Reservation in target AZ
- **Why:** Savings Plans don't reserve capacity

**Scenario 4:** Instance fails status checks, application unavailable.
- **Solution:** ASG with ELB health checks, auto-recovery
- **Why:** ASG replaces unhealthy instances automatically

**Scenario 5:** Application slow after stop/start, was fast before.
- **Solution:** Check instance store usage (data lost on stop)
- **Why:** Stop/start loses instance store, may need to rebuild cache

**Scenario 6:** Need to patch instances without downtime.
- **Solution:** Instance refresh with MinHealthyPercentage=90
- **Why:** Gradual rollout maintains availability during updates

**Scenario 7:** High cost from unused instances overnight.
- **Solution:** Scheduled scaling actions to scale down after hours
- **Why:** Pay only for capacity when needed

---

## 12. Interview Q&A

## 12. Interview Q&A

### Q1: Explain the difference between Stop/Start vs Reboot vs Terminate
**Answer:**

| Operation | Host | Instance Store | EBS | Public IP | Use Case |
|-----------|------|----------------|-----|-----------|----------|
| **Reboot** | Same | ✅ Preserved | ✅ Preserved | ✅ Same | Kernel update, config reload |
| **Stop/Start** | New | ❌ Lost | ✅ Preserved | ❌ New (auto-assigned) | Resize, troubleshoot, save cost |
| **Hibernate** | New | ❌ Lost | ✅ + RAM saved | ❌ New | Fast startup with state |
| **Terminate** | - | ❌ Deleted | Depends on DeleteOnTermination | - | Permanent removal |

**Interview Key Point:** "Stop/Start loses instance store and public IP, but EBS persists. Applications must handle these changes gracefully."

---

### Q2: Design a highly available web application architecture
**Answer:**

**Requirements:**
- 99.95% availability (max 4.4 hours downtime/year)
- Handle 10k requests/second
- Survive AZ failure

**Architecture:**
```
[Users] → [Route 53] → [CloudFront (optional CDN)]
                            ↓
                    [ALB in 3 AZs]
                            ↓
        ┌──────────────┬─────────────┬──────────────┐
        ↓              ↓             ↓              ↓
   [us-east-1a]  [us-east-1b]  [us-east-1c]     [RDS Multi-AZ]
   ASG: 2-10      ASG: 2-10      ASG: 2-10      Primary + Standby
   instances      instances      instances
```

**Components:**
1. **ALB:** Distributes traffic across AZs, health checks every 30s
2. **ASG:** Min=6 (2 per AZ), Max=30, Target tracking on request count
3. **Launch Template:** IMDSv2, IAM role, gp3 volumes, encrypted
4. **RDS Multi-AZ:** Automatic failover <2 minutes
5. **ElastiCache:** Shared session state

**Configuration:**
```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name web-asg \
  --launch-template LaunchTemplateName=web-app,Version=$Latest \
  --min-size 6 \
  --max-size 30 \
  --desired-capacity 9 \
  --vpc-zone-identifier "subnet-a,subnet-b,subnet-c" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/web/... \
  --health-check-type ELB \
  --health-check-grace-period 300

# Target tracking policy
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name web-asg \
  --policy-name scale-on-request-count \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "..."
    },
    "TargetValue": 1000.0
  }'
```

**Availability Math:**
- Single AZ: ~99.5% (43.8h downtime/year)
- Multi-AZ (3): 99.95%+ (infrastructure failure independent)

**Interview Tip:** "Distribute across 3+ AZs, use ALB for health-based routing, stateless application tier, external state layer."

---

### Q3: Compare Savings Plans vs Reserved Instances
**Answer:**

**Savings Plans:**
- **Commitment:** Hourly spend ($10/hour for 1-3 years)
- **Flexibility:** Any region, any instance family/size (Compute SP)
- **Discount:** 40-60% vs On-Demand
- **Capacity:** NO capacity guarantee
- **Use Case:** Variable workloads, multi-region

**Reserved Instances:**
- **Commitment:** Specific instance type/family
- **Flexibility:** Regional RI = any AZ, any size (same family); Zonal RI = specific AZ only
- **Discount:** 40-60% vs On-Demand
- **Capacity:** Zonal RI reserves capacity; Regional RI doesn't
- **Use Case:** Predictable, steady workloads

**Comparison Table:**
| Feature | Savings Plan | Regional RI | Zonal RI |
|---------|--------------|-------------|----------|
| Flexibility | ✅ High (any family) | Medium (same family) | ❌ Low (exact type) |
| Capacity Guarantee | ❌ | ❌ | ✅ |
| Discount | 40-60% | 40-60% | 40-60% |
| Best For | Dynamic workloads | Predictable workloads | Capacity-sensitive launches |

**Interview Scenario:**
"For a product launch, we bought a Savings Plan but launch failed."
**Answer:** "Savings Plans don't reserve capacity. For launch day, you need an On-Demand Capacity Reservation (can be covered by SP discount after launch)."

---

### Q4: How would you optimize costs for a batch processing workload?
**Answer:**

**Workload Characteristics:**
- Runs 4 hours daily (8 PM - midnight)
- Interruptible (checkpoints every 15 minutes)
- CPU-intensive
- 50 instances needed during processing

**Optimization Strategy:**

**1. Use Spot Instances (Primary)**
```bash
# Spot Fleet with multiple instance types
aws ec2 create-fleet \
  --launch-template-configs '{
    "LaunchTemplateSpecification": {...},
    "Overrides": [
      {"InstanceType": "c6i.2xlarge", "SubnetId": "subnet-a"},
      {"InstanceType": "c6a.2xlarge", "SubnetId": "subnet-b"},
      {"InstanceType": "c5.2xlarge", "SubnetId": "subnet-c"}
    ]
  }' \
  --target-capacity-specification TotalTargetCapacity=50,OnDemandTargetCapacity=5,SpotTargetCapacity=45 \
  --spot-options AllocationStrategy=price-capacity-optimized
```

**2. Scheduled Scaling**
```bash
# Scale up at 7:45 PM (warm-up before job)
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name batch-asg \
  --scheduled-action-name scale-up \
  --recurrence "45 19 * * *" \
  --desired-capacity 50

# Scale down at 12:15 AM
aws autoscaling put-scheduled-action \
  --auto-scaling-group-name batch-asg \
  --scheduled-action-name scale-down \
  --recurrence "15 0 * * *" \
  --desired-capacity 0
```

**3. Checkpointing for Spot Interruptions**
```python
# Worker polls for Spot interruption
import boto3
import time

def process_with_checkpointing():
    while has_work():
        work_batch = get_next_batch()
        
        # Check for Spot termination notice
        if is_spot_terminating():
            save_checkpoint(current_state)
            deregister_from_queue()
            break
        
        process_batch(work_batch)
        
        # Checkpoint every 15 minutes
        if time_since_last_checkpoint() > 900:
            save_checkpoint(current_state)
```

**Cost Breakdown:**
```
On-Demand baseline (5 instances):
5 × c6i.2xlarge × 4 hours × 30 days × $0.34/hour = $204/month

Spot instances (45 instances):
45 × c6i.2xlarge × 4 hours × 30 days × $0.07/hour (avg Spot) = $378/month

Total: $582/month vs $2,040 On-Demand = 71% savings
```

**Interview Tip:** "Spot + checkpointing for fault-tolerant batch workloads gives 70-90% savings. Diversify instance types to reduce interruption risk."

---

### Q5: Explain IMDSv2 and why it's recommended
**Answer:**

**IMDS (Instance Metadata Service):**
Accessible at `http://169.254.169.254/latest/meta-data/` from EC2 instances.

**IMDSv1 (Legacy):**
```bash
# Simple HTTP GET (no authentication)
curl http://169.254.169.254/latest/meta-data/instance-id
```

**Vulnerability:** SSRF (Server-Side Request Forgery) attacks can trick application into revealing metadata/credentials.

**IMDSv2 (Recommended):**
```bash
# Step 1: Get session token (PUT request)
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Step 2: Use token in header
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id
```

**Protection:**
- **PUT method:** Blocks simple GET-based SSRF
- **TTL header:** Token expires (6 hours max)
- **Hop limit:** Prevents forwarding through proxies (default 1 hop)

**Enforcement:**
```bash
# Launch template with IMDSv2 required
aws ec2 create-launch-template \
  --launch-template-name secure-app \
  --launch-template-data '{
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1,
      "HttpEndpoint": "enabled"
    }
  }'
```

**Interview Tip:** "IMDSv2 protects against SSRF attacks by requiring session tokens. Always use HttpTokens: required for new instances."

---

### Q6: How do you perform zero-downtime deployments with ASG?
**Answer:**

**Strategy: Instance Refresh with Health Checks**

**Step 1: Prepare New AMI**
```bash
# Build and test new AMI
aws ec2 create-image \
  --instance-id i-test-instance \
  --name "app-v2.0-$(date +%Y%m%d)" \
  --no-reboot
```

**Step 2: Update Launch Template**
```bash
aws ec2 create-launch-template-version \
  --launch-template-name app \
  --source-version $Latest \
  --launch-template-data '{"ImageId": "ami-new-version"}'

# Set as default version
aws ec2 modify-launch-template \
  --launch-template-name app \
  --default-version $Latest
```

**Step 3: Start Instance Refresh**
```bash
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name app-asg \
  --preferences '{
    "MinHealthyPercentage": 90,
    "InstanceWarmup": 300,
    "CheckpointPercentages": [50],
    "CheckpointDelay": 600,
    "ScaleInProtectedInstances": "Ignore"
  }'
```

**What Happens:**
```
T+0:    Refresh starts (10 instances on v1)
T+1m:   Terminate 1 old instance (9 old, 0 new)
T+2m:   Launch 1 new instance (9 old, 1 new launching)
T+7m:   New instance healthy (9 old, 1 new) ← InstanceWarmup=300s
T+8m:   Terminate next old instance (8 old, 1 new)
...
T+50m:  50% replaced (5 old, 5 new) ← Checkpoint, pause 10 minutes
T+60m:  Resume after checkpoint delay
...
T+90m:  100% replaced (0 old, 10 new)
```

**Rollback If Issues:**
```bash
# Cancel refresh
aws autoscaling cancel-instance-refresh --auto-scaling-group-name app-asg

# Revert launch template
aws ec2 modify-launch-template \
  --launch-template-name app \
  --default-version <previous-version>

# Start new refresh to rollback
```

**Interview Key Points:**
- "MinHealthyPercentage=90 ensures 9/10 instances always healthy"
- "InstanceWarmup gives app time to start before health checks"
- "Checkpoints allow validation before continuing"

---

### Q7: Troubleshoot: ASG keeps terminating healthy instances
**Answer:**

**Common Causes:**

**1. Health Check Grace Period Too Short**
```bash
# Check current grace period
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names app-asg \
  --query 'AutoScalingGroups[0].HealthCheckGracePeriod'

# Application takes 3 minutes to start, but grace period = 60s
# → Instance marked unhealthy before fully started

# Fix: Increase grace period
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name app-asg \
  --health-check-grace-period 300  # 5 minutes
```

**2. ELB Health Check Misconfigured**
```bash
# Check target group health check
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:...

# Common issues:
# - Wrong path (checking /health but app serves /healthz)
# - Wrong expected status (checking for 200 but app returns 204)
# - Interval too short (5s, app needs 10s to respond)

# Fix target group health check
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:... \
  --health-check-path /healthz \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3
```

**3. Application Crashes After Launch**
```bash
# Check CloudWatch Logs for application errors
aws logs tail /aws/ec2/app-logs --follow

# Common issues:
# - Missing environment variables
# - Cannot connect to database (security group)
# - Insufficient memory (OOMKilled)
```

**4. Instance Status Checks Failing**
```bash
# Check system/instance status
aws ec2 describe-instance-status --instance-ids i-...

# System check failure: AWS infrastructure issue → replace
# Instance check failure: Guest OS issue → investigate logs
```

**Debugging Checklist:**
```
✅ Health check grace period ≥ application startup time
✅ ELB health check path returns 200 OK
✅ Security group allows health check traffic
✅ Application binds to 0.0.0.0 (not 127.0.0.1)
✅ CloudWatch Logs show successful startup
✅ Instance has correct IAM role for dependencies
```

**Interview Tip:** "Check health check grace period first - most common cause. Then verify ALB target health and application logs."

---

### Q8: Compare EBS vs Instance Store
**Answer:**

| Feature | EBS | Instance Store |
|---------|-----|----------------|
| **Persistence** | ✅ Survives stop/start | ❌ Lost on stop/terminate/host failure |
| **Scope** | AZ-level (can snapshot to S3) | Host-local |
| **Performance** | gp3: 16K IOPS, 1000 MB/s | NVMe: 100K+ IOPS, GB/s throughput |
| **Backup** | Snapshots to S3 | Must replicate externally |
| **Flexibility** | Attach/detach, resize | Fixed at instance type |
| **Cost** | $0.08-0.125/GB-month | Included in instance price |
| **Latency** | ~1ms | ~100μs |
| **Use Case** | Boot volumes, databases | Cache, scratch, temporary data |

**When to Use Instance Store:**
```python
# Good use cases:
✅ Distributed data (HDFS, Cassandra) - replicated elsewhere
✅ Cache (Redis) - can rebuild from primary DB
✅ Temporary processing (video encoding) - results saved to S3
✅ High IOPS workloads (>16K IOPS)

# Bad use cases:
❌ Boot volumes (lost on stop)
❌ Databases without replication
❌ Critical application state
❌ Workloads requiring snapshots
```

**Instance Store Performance:**
```
i4i.4xlarge: 4 × 937GB NVMe SSD
- 400,000 IOPS
- 6,250 MB/s throughput
- ~50μs latency

vs

EBS io2 Block Express (max):
- 256,000 IOPS
- 4,000 MB/s
- ~1ms latency
```

**Interview Tip:** "EBS for persistence and flexibility; instance store for extreme performance with ephemeral data."

---

### Q9: Design a disaster recovery strategy across regions
**Answer:**

**RTO/RPO Requirements:**
- RTO: 1 hour (max downtime)
- RPO: 5 minutes (max data loss)

**Strategy: Warm Standby**

**Primary Region (us-east-1):**
```
[Route 53] → [Primary: Weighted 100%]
                ↓
        [ALB + ASG (10 instances)]
                ↓
        [RDS Multi-AZ Primary]
                ↓
        [S3 Replication to DR region]
```

**DR Region (eu-west-1):**
```
[Route 53] → [DR: Weighted 0%]
                ↓
        [ALB + ASG (2 instances, scaled down)]
                ↓
        [RDS Read Replica (promote on failover)]
                ↓
        [S3 (replicated)]
```

**Setup:**

**1. Database Replication**
```bash
# Create cross-region read replica
aws rds create-db-instance-read-replica \
  --db-instance-identifier db-dr-eu \
  --source-db-instance-identifier arn:aws:rds:us-east-1:...:db:db-primary \
  --db-instance-class db.r6g.xlarge \
  --region eu-west-1

# Replication lag typically <5 seconds = RPO ~5s
```

**2. AMI Replication**
```bash
# Copy AMI to DR region (automated with Lambda)
aws ec2 copy-image \
  --source-region us-east-1 \
  --source-image-id ami-primary \
  --name app-v1.0-dr \
  --region eu-west-1
```

**3. Route 53 Failover**
```bash
# Primary record (health checked)
aws route53 change-resource-record-sets --change-batch '{
  "Changes": [{
    "Action": "CREATE",
    "ResourceRecordSet": {
      "Name": "api.example.com",
      "Type": "A",
      "SetIdentifier": "primary-us-east-1",
      "Failover": "PRIMARY",
      "AliasTarget": {...},
      "HealthCheckId": "hc-primary"
    }
  }]
}'

# DR record (no health check, becomes active when primary fails)
# Similar with "Failover": "SECONDARY"
```

**4. S3 Replication**
```bash
aws s3api put-bucket-replication --bucket primary-bucket --replication-configuration '{
  "Role": "arn:aws:iam::account:role/S3ReplicationRole",
  "Rules": [{
    "Status": "Enabled",
    "Priority": 1,
    "Destination": {"Bucket": "arn:aws:s3:::dr-bucket", "ReplicationTime": {"Status": "Enabled", "Time": {"Minutes": 15}}}
  }]
}'
```

**Failover Procedure:**
```bash
# 1. Promote RDS read replica to standalone
aws rds promote-read-replica \
  --db-instance-identifier db-dr-eu \
  --region eu-west-1

# 2. Scale up DR ASG
aws autoscaling update-auto-scaling-group \
  --auto-scaling-group-name dr-asg \
  --desired-capacity 10 \
  --region eu-west-1

# 3. Route 53 automatically fails over (primary health check fails)
# OR manually update weights if primary region degraded but health check still passes
```

**Cost:**
```
Primary: $1,000/month (10 instances + RDS primary)
DR: $200/month (2 instances + RDS replica)
Total: $1,200/month (20% overhead for DR capability)
```

**Interview Tip:** "Warm standby balances cost (minimal DR capacity) with RTO (scale up in minutes). Read replica gives RPO <5 minutes."

---

### Q10: How do Spot Instances work and when should you use them?
**Answer:**

**How Spot Works:**
1. AWS sells spare EC2 capacity at up to 90% discount
2. Price fluctuates based on supply/demand
3. AWS can interrupt with 2-minute warning when capacity needed
4. Best effort to provide warning (not guaranteed)

**Interruption Handling:**
```bash
# Poll for interruption notice (every 5 seconds)
while true; do
    HTTP_CODE=$(curl -s -o /dev/null -w "%{http_code}" \
      http://169.254.169.254/latest/meta-data/spot/instance-action)
    
    if [ "$HTTP_CODE" == "200" ]; then
        echo "Spot interruption detected"
        # Graceful shutdown: save state, deregister from LB, exit
        /app/graceful-shutdown.sh
        break
    fi
    
    sleep 5
done
```

**When to Use Spot:**

**✅ Good Use Cases:**
- Batch processing (checkpointed)
- Data analysis (can resume)
- CI/CD workers (stateless)
- Rendering farms
- Web crawling
- Machine learning training (checkpointed)

**❌ Bad Use Cases:**
- Databases (state loss)
- Long-running transactions (>2 hours)
- Real-time APIs without On-Demand fallback
- Workloads requiring 99.95%+ availability

**Best Practices:**
```bash
# 1. Diversify instance types (reduces interruption probability)
aws ec2 request-spot-fleet --spot-fleet-request-config '{
  "AllocationStrategy": "price-capacity-optimized",
  "LaunchTemplateConfigs": [{
    "LaunchTemplateSpecification": {...},
    "Overrides": [
      {"InstanceType": "m5.large"},
      {"InstanceType": "m5a.large"},
      {"InstanceType": "m5n.large"},
      {"InstanceType": "m6i.large"}
    ]
  }]
}'

# 2. Use capacity-optimized allocation (not lowest-price)
# 3. Mix with On-Demand (e.g., 70% Spot, 30% On-Demand baseline)
# 4. Implement checkpointing (save state every 10-15 minutes)
# 5. Use Spot Instance Advisor to check interruption rates
```

**Spot Instance Advisor:**
- Shows historical interruption frequency (<5%, 5-10%, 10-15%, >20%)
- Helps choose instance types with low interruption rates

**Interview Decision Matrix:**
| Scenario | Spot? | Justification |
|----------|-------|---------------|
| Web servers with ASG | ✅ Mix 70/30 | ASG replaces interrupted instances |
| Database primary | ❌ | State loss unacceptable |
| Batch analytics | ✅ 100% | Checkpointed, interruptible |
| Real-time API | ⚠️ Partial | On-Demand baseline, Spot for burst |
| CI/CD workers | ✅ 100% | Stateless, job requeued on interruption |

**Interview Tip:** "Spot for fault-tolerant workloads saves 70-90%. Diversify instance types, checkpoint state, handle interruptions gracefully."

---

**Additional Quick Answers:**

**One server must survive an AZ outage:** A single instance cannot. Deploy redundant instances across AZs with an external state layer, health-based routing, and enough surviving capacity.

**Nightly processing can restart from a checkpoint:** Consider Spot workers consuming a durable queue, with checkpoints in persistent storage. Compare deadline risk against savings.

**A database is slow after increasing instance size:** Measure IOPS, throughput, latency, locks, and working set; compute scaling alone may not address storage or query design.

**We bought a Savings Plan but launch failed. Why?** It changes billing, not available capacity. Check AZ capacity, quotas, and whether a matching reservation is needed.

**How do you patch without an outage?** Bake/test a new image, roll replacements across AZs, gate on health and latency, drain old instances, and keep rollback capacity.

---

## 13. Quick Revision

- Instance = one AZ; availability = architecture across AZs
- EBS persistence ≠ backups and cross-AZ recovery
- IAM role controls API access; security group controls network traffic
- Stop ≠ terminate, and stop ≠ zero cost (still pay for EBS, Elastic IPs)
- Savings Plans are NOT capacity reservations
- ASG replaces/scales instances; ELB distributes traffic
- Stop/Start loses instance store and auto-assigned public IP
- IMDSv2 (HttpTokens: required) protects against SSRF
- Spot saves 70-90% but can be interrupted (2-minute warning)
- gp3 is better cost/performance than gp2 (use for new volumes)

---

## 14. AWS sources and further reading

- [EC2 purchasing options](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-purchasing-options.html)
- [Purchasing decision guide](https://docs.aws.amazon.com/decision-guides/latest/decision-guides/ec2-purchasing-options-aws-how-to-choose.html)
- [Instance lifecycle](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-lifecycle.html)
- [Instance metadata and IMDSv2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html)

**Related:** [ELB](aws-elb-cheatsheet.md), [ECS](aws-ecs-cheatsheet.md), [EKS](aws-eks-cheatsheet.md).
