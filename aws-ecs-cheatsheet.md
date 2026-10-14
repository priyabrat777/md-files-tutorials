# Amazon ECS Cheat Sheet

**Interview preparation: task definitions, Fargate, managed compute, IAM, networking, deployments, and operations.**

**Verified:** 2026-10-13. Capacity-provider, OS, platform-version, and Region support should be checked for the chosen workload.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Compute choices and current features](#2-compute-choices-and-current-features)
- [3. The IAM roles interviewers expect](#3-the-iam-roles-interviewers-expect)
- [4. Networking and service communication](#4-networking-and-service-communication)
- [5. Deployments and health](#5-deployments-and-health)
- [6. Two independent scaling loops](#6-two-independent-scaling-loops)
- [7. Storage, security, and costs](#7-storage-security-and-costs)
- [8. Troubleshooting](#8-troubleshooting)
- [9. Practical inspection](#9-practical-inspection)
- [10. Common patterns](#10-common-patterns)
- [11. Interview questions and model answers](#11-interview-questions-and-model-answers)
- [12. Quick reference card](#12-quick-reference-card)
- [13. Exam tips summary](#13-exam-tips-summary)
- [14. AWS sources](#14-aws-sources)

---

## 1. Fundamentals and mental model

ECS is AWS-native container orchestration. It schedules containers, maintains service task counts, and integrates deeply with networking, IAM, load balancing, auto scaling, and monitoring—with no Kubernetes control plane to operate.

### Full lifecycle mental model

```text
┌─────────────────────────────────────────────────────────────────────┐
│                        CONTAINER LIFECYCLE                          │
│                                                                     │
│  Developer pushes image                                             │
│       │                                                             │
│       ▼                                                             │
│  ┌─────────┐   image URI/digest   ┌──────────────────────────┐     │
│  │   ECR   │ ──────────────────→  │   Task Definition (rev)  │     │
│  │  image  │                      │  cpu, memory, ports,     │     │
│  │  store  │                      │  roles, volumes, logging │     │
│  └─────────┘                      └────────────┬─────────────┘     │
│                                                │ referenced by      │
│                                                ▼                    │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                        Service                           │      │
│  │  desired count • deployment config • health-check grace  │      │
│  │  deployment controller (rolling / CodeDeploy / external) │      │
│  └───────────┬───────────────────────────┬──────────────────┘      │
│              │ places tasks              │ registers targets        │
│              ▼                           ▼                          │
│  ┌───────────────────────┐   ┌──────────────────────────────┐      │
│  │   Task (running)      │   │  Target Group (IP mode)      │      │
│  │  container A          │   │       │                       │      │
│  │  container B (sidecar)│   │       ▼                       │      │
│  │  ENI / awsvpc         │   │  ALB listener rule           │      │
│  └───────────────────────┘   │       │                       │      │
│                               │       ▼                       │      │
│  ┌────────────────────────┐  │  Internet / internal client  │      │
│  │       Cluster          │  └──────────────────────────────┘      │
│  │  (logical boundary)    │                                         │
│  └────────────┬───────────┘                                         │
│               │ capacity from                                        │
│               ▼                                                      │
│  ┌────────────────────────┐                                          │
│  │   Capacity Provider    │                                          │
│  │  FARGATE / FARGATE_SPOT│                                          │
│  │  ASG / Managed Instances│                                         │
│  └────────────────────────┘                                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Core object glossary

| Term | Meaning |
|---|---|
| Cluster | Logical grouping for services, tasks, and capacity |
| Task definition | Versioned configuration: image, CPU/memory, ports, roles, logs, volumes |
| Task | Running instance of a task definition; may contain multiple containers |
| Service | Keeps desired task count; manages rolling/blue-green deployments and load-balancer registration |
| Standalone task | One-off or batch execution without service replacement semantics |
| Capacity provider | Connects placement to available compute and handles scaling |
| Service Connect | Envoy-based mesh for intra-cluster service-to-service traffic |
| ECS Exec | SSM-based shell access into a running container |

### Mental model: task definition vs task vs service

```text
Task Definition  =  Blueprint / class
      │
      │  instantiated by service or manual launch
      ▼
Task             =  Running process
      │
      │  lifecycle managed by
      ▼
Service          =  Controller / desired-state loop
                    (replaces failed tasks, drives deployments)
```

---

## 2. Compute choices and current features

### Comparison table

| Choice | You primarily manage | Use case | Spot/interruption |
|---|---|---|---|
| Fargate | Tasks, resource sizing, application code | Eliminate host operations | Fargate Spot available |
| EC2 with ASG capacity provider | Hosts, AMI, patching, instance type plus tasks | Host control, GPU, custom instance types | EC2 Spot via ASG |
| ECS Managed Instances | Instance requirements and workload config | AWS-managed EC2 capacity with broader instance choice | Varies by configuration |
| Fargate Spot | Same as Fargate | Interruption-tolerant batch/flex work | Always interrupted |

### Fargate vs EC2 cost tradeoffs

```text
Fargate:
  ✔ No idle EC2 capacity costs when tasks are 0
  ✔ No AMI patching, no agent management
  ✔ Per-task billing (vCPU-second, GB-second)
  ✗ Higher per-vCPU rate than equivalent EC2 On-Demand
  ✗ No GPU support
  ✗ Bin-packing many small tasks onto shared hosts not possible

EC2 (via ASG capacity provider):
  ✔ Lower per-vCPU rate; can bin-pack many containers per host
  ✔ GPU, ARM, bare metal, custom instance types
  ✔ Reserved Instance / Savings Plan discounts apply to hosts
  ✗ Must size, patch, and replace EC2 fleet
  ✗ Idle capacity costs when task count is low
  ✗ Scale-out has a lag (host must boot before task can place)

Rule of thumb:
  • Highly variable traffic  →  Fargate (no idle host cost)
  • High, steady utilization →  EC2 Reserved (bin-packing wins)
  • GPU / specialized HW    →  EC2 mandatory
  • Short-lived batch       →  Fargate Spot (often cheapest)
```

### Key current features to know

**ECS Managed Instances** manages compute lifecycle through capacity providers. A capacity-provider strategy uses one provider family—Managed Instances, ASG, or Fargate/Fargate Spot—do not freely mix families in one strategy. [Managed Instances docs](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ManagedInstances.html)

**Express Mode** simplifies web-service setup and provisions Fargate, load-balancing, networking, and scaling configuration. It is a deployment convenience; you still need to understand the underlying resources and the bill they generate. [Express Mode resources](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/express-service-work.html)

**ECS Exec** uses AWS Systems Manager Session Manager to open an interactive shell or run a command inside a running container. Requires the task role to allow `ssmmessages:*` and the execute-command flag on the service/task. Audit all sessions with CloudTrail and SSM session logs.

**Service Connect** provides Envoy-based service mesh inside the cluster. Services register under logical names; clients reference names without DNS lookups. Adds retries, circuit-breaking, and per-route metrics without application code changes.

---

## 3. The IAM roles interviewers expect

### Role summary table

| Role | Used by | Typical permissions | Trust principal |
|---|---|---|---|
| Task role | Application code in containers | Read S3, send SQS, query DynamoDB | `ecs-tasks.amazonaws.com` |
| Task execution role | ECS/Fargate agent operations | Pull ECR image, send logs, retrieve secrets | `ecs-tasks.amazonaws.com` |
| EC2 instance role | ECS agent on self-managed EC2 | Register host, coordinate with ECS | `ec2.amazonaws.com` |
| Infrastructure role | Supported ECS-managed infrastructure features | Manage associated AWS resources | ECS service principal |

### Mental model: which role does what

```text
         ┌─────────────────────────────────┐
         │           ECS/Fargate agent      │
         │  pulls image ──► ECR             │  ← Task Execution Role
         │  sends logs  ──► CloudWatch      │  ← Task Execution Role
         │  fetches secret ► Secrets Manager│  ← Task Execution Role
         └─────────────────────────────────┘

         ┌─────────────────────────────────┐
         │       Your application code     │
         │  reads file  ──► S3             │  ← Task Role
         │  sends msg   ──► SQS            │  ← Task Role
         │  reads table ──► DynamoDB       │  ← Task Role
         └─────────────────────────────────┘

Key insight: adding app permissions to the execution role does NOT
grant them to application code. A failed image pull and an app
AccessDenied always point to DIFFERENT identities.
```

### Task role — trust policy and permission policy

```json
// Trust policy (who can assume this role)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```json
// Permission policy (what the application can do)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AppS3Access",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Sid": "AppSQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
    }
  ]
}
```

### Task execution role — trust policy and permission policy

```json
// Trust policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

```json
// Permission policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ECRImagePull",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/ecs/my-service:*"
    },
    {
      "Sid": "SecretsManager",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/*"
    }
  ]
}
```

### Handling secrets in ECS

ECS supports injecting secrets as environment variables or bind-mounted files at task startup. The execution role must have permission to read the secret; the task role has no involvement.

```json
// Task definition container definition (excerpt)
{
  "secrets": [
    {
      "name": "DB_PASSWORD",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:prod/myapp/db-password-AbCdEf"
    },
    {
      "name": "API_KEY",
      "valueFrom": "arn:aws:ssm:us-east-1:123456789012:parameter/prod/myapp/api-key"
    }
  ]
}
```

**Important caveats:**
- Secrets are injected at container start. A running container does not automatically pick up rotated secret values—you must restart (redeploy) the task.
- Never place secrets in container images or environment variables hardcoded in the task definition.
- Use KMS CMKs with Secrets Manager for envelope encryption; the execution role needs `kms:Decrypt` on the relevant key if using a customer-managed key.

---

## 4. Networking and service communication

### Network mode comparison

| Mode | Supported compute | IP assignment | Load-balancer target type | Security groups |
|---|---|---|---|---|
| `awsvpc` | Fargate, EC2 | Task gets its own ENI and IP | **IP** targets | Per-task security group |
| `bridge` | EC2 only | Host IP + dynamic port mapping | Instance targets | Host security group |
| `host` | EC2 only | Host IP, same namespace | Instance targets | Host security group |

### awsvpc deep dive

With `awsvpc`, every task gets its own elastic network interface. This means:
- Task traffic is isolated at the VPC level (security groups enforce per-task rules)
- Fargate **requires** `awsvpc`—there is no bridge or host mode
- Target groups must use **IP** target type, not instance target type
- ENI density limits apply: each EC2 instance type has a maximum number of ENIs; with many small tasks on one host, you may exhaust ENIs before CPU or memory

### VPC connectivity requirements for Fargate

Private-subnet Fargate tasks need reachability to AWS APIs. You can provide this with:

```text
Option A: NAT Gateway (simplest, egress-only)
  Private subnet → NAT Gateway (public subnet) → Internet

Option B: VPC Interface Endpoints (no internet, lower data transfer cost)
  Required endpoints:
  • com.amazonaws.REGION.ecr.api       (ECR API)
  • com.amazonaws.REGION.ecr.dkr       (ECR image layers)
  • com.amazonaws.REGION.logs          (CloudWatch Logs)
  • com.amazonaws.REGION.secretsmanager (Secrets Manager)
  • com.amazonaws.REGION.ssm           (SSM Parameter Store)
  • com.amazonaws.REGION.s3 (Gateway endpoint, free)
```

### Service Connect vs Cloud Map

| Feature | Service Connect | Cloud Map |
|---|---|---|
| Protocol support | HTTP/1.1, HTTP/2, gRPC, TLS | DNS A/SRV, HTTP health checks |
| Retries / circuit-breaking | Built-in (Envoy) | None (DNS only) |
| Per-route metrics | Yes (CloudWatch) | Limited |
| Sidecar required | Yes (managed Envoy sidecar injected by ECS) | No |
| Discovery scope | Within ECS Service Connect namespace | Any client that can query Route 53 |
| Use case | Intra-cluster east-west service mesh | Broader service discovery (non-ECS workloads included) |

### Key networking rules

- A public subnet alone does not give a task internet connectivity—the task also needs a public IP assigned and a route to the internet gateway, or a NAT path.
- Subnet address exhaustion can prevent scale-out even when vCPU capacity is available. Monitor `SubnetAvailableIPAddresses` via CloudWatch.
- Keep external traffic on an ALB/NLB and restrict task security groups to accept inbound only from the load balancer security group.
- Avoid assigning public IPs to every task just to allow ECR pulls—use VPC endpoints or NAT instead.

---

## 5. Deployments and health

### Deployment controller options

| Controller | Traffic shifting | Rollback | Requires |
|---|---|---|---|
| ECS (rolling) | Gradual, controlled by min/max percent | Automatic or manual | No additional setup |
| CodeDeploy (blue/green) | Linear or canary rules; traffic on ALB/NLB | One-click rollback | CodeDeploy app + deployment group |
| External | Custom logic | Custom logic | Your deployment orchestrator |

### Rolling deployment mechanics

```text
Desired: 4    Min healthy: 100%    Max percent: 200%

Phase 1: Launch 4 new tasks (total 8, all healthy required)
Phase 2: Drain and deregister 4 old tasks
Final:   4 new tasks running

Desired: 4    Min healthy: 50%    Max percent: 150%

Phase 1: Stop 2 old tasks (2 old, 0 new = 50% healthy — at limit)
Phase 2: Start 2 new tasks
Phase 3: Stop remaining 2 old tasks
Phase 4: Start 2 more new tasks
Final:   4 new tasks running (faster, lower cost, brief reduced capacity)
```

A deployment stalls when configured settings prevent both starting a new task (max percent reached) and stopping an old one (min healthy percent reached). Increase max percent or reduce min healthy to unblock.

### Blue/green with CodeDeploy

```text
ALB Listener
    │
    ├── Production rule (100% → Blue TG)    before deployment
    └── Test rule (100% → Green TG, port 8080)

Deployment starts:
  1. ECS launches new tasks into Green TG
  2. Green TG health checks pass
  3. CodeDeploy shifts traffic: production rule → Green TG
  4. Bake time (optional monitoring window)
  5. Old Blue tasks terminated

Rollback:
  Shift production rule back to Blue TG instantly (Blue tasks still running during bake time)
```

### Health signals

| Signal | What it checks | Where configured |
|---|---|---|
| Container health check | Process inside container (e.g., `curl localhost/health`) | Task definition |
| ALB target health check | HTTP response code from the container | Target group |
| ECS service health | Containers are RUNNING and healthy | Service definition |

Configure `healthCheckGracePeriodSeconds` on the service to give containers time to initialize before ECS acts on failing health checks. A value too small causes task churn during cold starts.

### Graceful shutdown

- ECS sends `SIGTERM` to the container when draining. After `stopTimeout` seconds, it sends `SIGKILL`.
- Default `stopTimeout` is 30 seconds (max 120). Increase it if your application needs more time to finish in-flight requests.
- Register a `SIGTERM` handler in your application to stop accepting new connections and drain existing ones.

---

## 6. Two independent scaling loops

| Loop | Scales | Typical signal |
|---|---|---|
| Service Auto Scaling | Number of tasks | CPU/memory utilization, ALB requests per target, custom metric |
| Capacity-provider / host scaling | Compute to place tasks | Pending task placement demand |

### Service Auto Scaling CLI example

```bash
# Register the ECS service as a scalable target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Create a target tracking policy on ALB requests per task
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/my-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name alb-requests-per-target \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 1000,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ALBRequestCountPerTarget",
      "ResourceLabel": "app/my-alb/xxxx/targetgroup/my-tg/yyyy"
    },
    "ScaleInCooldown": 60,
    "ScaleOutCooldown": 30
  }'
```

### Queue-worker scaling guidance

For SQS-based workers, the ideal metric is **backlog per task** = `ApproximateNumberOfMessagesVisible / running task count`. Scale out when that ratio exceeds your processing time target; scale in when it falls. Also watch `ApproximateAgeOfOldestMessage` as a latency SLO signal.

More desired tasks do not help if there is no capacity, no subnet IP space, or a conflicting placement constraint. Fargate removes host scaling concern but account-level task quotas and platform capacity still apply.

---

## 7. Storage, security, and costs

### Storage options

| Option | Scope | Use case |
|---|---|---|
| Ephemeral storage | Single task lifetime | Temp files, build artifacts; lost when task stops |
| EFS (NFS) | Shared across tasks | Shared config, ML models, CMS uploads |
| EBS (task-level volume) | Single task | High-IOPS workloads; attached at task launch, Fargate supported on Linux |
| S3 | Application-level | Durable object storage; app reads/writes via SDK |
| Bind mount / tmpfs | Container within task | Sharing data between containers in the same task |

Fargate ephemeral storage default is 20 GiB; configurable up to 200 GiB (additional cost).

### Security practices

- Run containers as non-root. Set `"user": "1000"` in the container definition.
- Use read-only root filesystems where possible (`"readonlyRootFilesystem": true`).
- Enable image vulnerability scanning in ECR (enhanced scanning via Inspector).
- Restrict ECS Exec access: grant `ssmmessages:CreateControlChannel`, `ssmmessages:CreateDataChannel`, `ssmmessages:OpenControlChannel`, `ssmmessages:OpenDataChannel` only to the task roles that need it, and log all sessions to S3 or CloudWatch Logs.
- Apply least-privilege task roles scoped to specific resource ARNs.
- Fetch or inject secrets through Secrets Manager or SSM Parameter Store at task startup—never embed in images or task definition environment variables.

### Cost components

```text
Fargate pricing:
  vCPU-hour × rate + GB-hour × rate
  (billed from task start to task stop)

EC2 pricing:
  Instance type × hours (regardless of how many tasks run on it)
  → Aim for high bin-packing (>70% CPU utilization per host)

Additional costs to account for:
  • NAT Gateway (data processing + hourly)
  • ALB / NLB (hourly + LCU)
  • ECR storage ($/GB/month) + data transfer
  • CloudWatch Logs ingestion and storage
  • EFS throughput and storage
  • Fargate Spot savings: ~70% discount vs Fargate On-Demand
    but tasks can be interrupted with 2-minute notice
```

---

## 8. Troubleshooting

**Always read the stopped-task reason and container exit code before changing CPU limits or restarting blindly.**

```bash
# Get stopped reason and exit code for stopped tasks
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks TASK_ARN \
  --query 'tasks[0].{stoppedReason:stoppedReason,containers:containers[*].{name:name,exitCode:exitCode,reason:reason}}'
```

### Symptom → root cause table

| Symptom | Likely root causes | How to confirm |
|---|---|---|
| Task stuck in `PENDING` | (a) Insufficient CPU/memory in cluster; (b) No subnets with free ENIs; (c) No free IPs in subnet; (d) Placement constraint or spread rule cannot be satisfied; (e) Capacity provider not linked to cluster | `describe-services` → events; `describe-capacity-providers`; CloudWatch `SubnetAvailableIPAddresses` |
| Image pull failure | (a) Wrong image URI or tag; (b) Image digest mismatch; (c) Execution role missing ECR permissions; (d) No connectivity to ECR (NAT/endpoint missing); (e) Architecture mismatch (arm64 image on x86 host) | CloudWatch task stopped reason; `ecr:GetAuthorizationToken` in execution role; VPC route table |
| Task fails at resource initialization | (a) Execution role cannot read the secret (Secrets Manager/SSM); (b) Missing VPC endpoint for the secrets service; (c) EFS mount timeout (security group, mount target missing in subnet); (d) Log driver cannot reach CloudWatch (execution role or endpoint) | Task stopped reason usually names the failing dependency; check security group rules |
| Tasks continually stop / restart loop | (a) Essential container exits (non-zero exit code = application crash); (b) Container killed by OOM (exit code 137); (c) Health check fails repeatedly; (d) Application throws unhandled exception on startup | Exit code 137 = OOM; exit code 1 = app error; read container logs in CloudWatch |
| Application gets `AccessDenied` for S3/DynamoDB/SQS | (a) Task role missing the permission; (b) Resource-based policy on the bucket/queue/table denies the task role; (c) KMS key policy does not allow the task role; (d) Application accidentally using execution role credentials | `aws sts get-caller-identity` from inside container via ECS Exec; check both task role and resource policy |
| Deployment stuck (no progress) | (a) Min healthy % and max % settings leave no room to place or remove tasks; (b) New tasks fail health checks (health-check grace period too short); (c) No spare capacity for new tasks (EC2 cluster full) | `describe-services` → deployments; increase max percent; increase health-check grace period |
| ALB returns 502 Bad Gateway | (a) Task registered with wrong port (container port ≠ target group port); (b) Container binds to `127.0.0.1` instead of `0.0.0.0`; (c) Task deregistered before finishing in-flight requests; (d) Health check path returns non-2xx | Target group health in ALB console; check container bind address in application startup logs |
| ALB returns 503 Service Unavailable | All registered targets unhealthy or target group empty (tasks still launching / old tasks already drained) | ALB target health tab; check task count vs desired count |
| ECS Exec fails | (a) `--enable-execute-command` not set on service/task; (b) Task role missing `ssmmessages:*` permissions; (c) SSM agent not running in container (Fargate: built-in; EC2: must be in image) | `describe-tasks` → `enableExecuteCommand`; task role policy |
| Scale-out does not place tasks | (a) Fargate: account-level task quota reached; (b) EC2: cluster has capacity but ENIs exhausted; (c) Subnet IP exhaustion; (d) ASG max size reached | Service events; AWS quota console; CloudWatch metrics |

---

## 9. Practical inspection

Read-only inspection commands:

```bash
# Describe a service — check desired/running/pending counts and recent events
aws ecs describe-services \
  --cluster app-cluster \
  --services orders \
  --query 'services[0].{desired:desiredCount,running:runningCount,pending:pendingCount,events:events[:5]}'

# Inspect a specific task definition revision
aws ecs describe-task-definition \
  --task-definition orders:12

# List running tasks for a service
aws ecs list-tasks \
  --cluster app-cluster \
  --service-name orders

# Describe tasks (stopped tasks included if using --desired-status STOPPED)
aws ecs describe-tasks \
  --cluster app-cluster \
  --tasks TASK_ARN

# Check capacity provider configuration
aws ecs describe-capacity-providers \
  --capacity-providers FARGATE FARGATE_SPOT

# Open a shell into a running container (ECS Exec)
aws ecs execute-command \
  --cluster app-cluster \
  --task TASK_ARN \
  --container app \
  --interactive \
  --command "/bin/sh"

# Check what image digest is actually running
aws ecs describe-tasks \
  --cluster app-cluster \
  --tasks TASK_ARN \
  --query 'tasks[0].containers[*].{name:name,image:image,imageDigest:imageDigest}'

# Verify the effective task role
aws ecs describe-tasks \
  --cluster app-cluster \
  --tasks TASK_ARN \
  --query 'tasks[0].overrides.taskRoleArn'
```

Compare desired/running/pending tasks and recent service events. Verify the deployed image digest matches what was pushed. Check that the task role ARN is the one you expect.

---

## 10. Common patterns

### Pattern 1: Web service on Fargate with ALB

**Scenario:** A stateless HTTP API needs to handle variable traffic, be highly available across two AZs, and not require you to manage EC2 instances.

```text
Architecture:
  Internet → ALB (public subnets, AZ-a and AZ-b)
           → Target Group (IP mode, port 8080)
           → ECS Service (Fargate, 2 AZs, desired 2 tasks, max 10)
           → Tasks in private subnets
           → NAT Gateway or VPC endpoints for outbound
```

```bash
# 1. Create the cluster
aws ecs create-cluster \
  --cluster-name web-cluster \
  --capacity-providers FARGATE FARGATE_SPOT

# 2. Register task definition (excerpt — use full JSON in production)
aws ecs register-task-definition \
  --family web-api \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 512 --memory 1024 \
  --execution-role-arn arn:aws:iam::123456789012:role/ecsTaskExecutionRole \
  --task-role-arn arn:aws:iam::123456789012:role/web-api-task-role \
  --container-definitions '[{
    "name": "api",
    "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/web-api:1.2.3",
    "portMappings": [{"containerPort": 8080}],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/web-api",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "ecs"
      }
    },
    "essential": true
  }]'

# 3. Create the service
aws ecs create-service \
  --cluster web-cluster \
  --service-name web-api \
  --task-definition web-api:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-aaa", "subnet-bbb"],
      "securityGroups": ["sg-tasks"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --load-balancers '[{
    "targetGroupArn": "arn:aws:elasticloadbalancing:...",
    "containerName": "api",
    "containerPort": 8080
  }]' \
  --health-check-grace-period-seconds 30 \
  --deployment-configuration '{
    "minimumHealthyPercent": 100,
    "maximumPercent": 200
  }'
```

**Key decisions:**
- `assignPublicIp: DISABLED` — keep tasks private; ALB in public subnet handles internet traffic
- `minimumHealthyPercent: 100` — no reduction in capacity during rolling deploys
- `healthCheckGracePeriodSeconds: 30` — prevents health-check churn during JVM/Node cold start
- Use image digest (`sha256:…`) instead of mutable tag (`latest`) for reproducible deploys

---

### Pattern 2: Background worker consuming SQS

**Scenario:** A worker service polls an SQS queue, processes messages, and deletes them. Throughput must auto-scale with queue depth.

```text
Architecture:
  SQS Queue → ECS Service (Fargate, no ALB)
            → Tasks poll queue, process messages, delete

Scaling signal: ApproximateNumberOfMessagesVisible / running task count
```

```json
// Task definition container definition (key parts)
{
  "name": "worker",
  "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/queue-worker:2.0.1",
  "environment": [
    { "name": "QUEUE_URL", "value": "https://sqs.us-east-1.amazonaws.com/123456789012/my-queue" },
    { "name": "BATCH_SIZE", "value": "10" }
  ],
  "essential": true,
  "stopTimeout": 120
}
```

```json
// Task role permission policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:DeleteMessageBatch",
        "sqs:GetQueueAttributes",
        "sqs:ChangeMessageVisibility"
      ],
      "Resource": "arn:aws:sqs:us-east-1:123456789012:my-queue"
    }
  ]
}
```

```bash
# Custom CloudWatch metric for scaling: messages per task
# Publish from application or use a Lambda poller:
aws cloudwatch put-metric-data \
  --namespace "ECS/SQSWorker" \
  --metric-data '[{
    "MetricName": "MessagesPerTask",
    "Value": 42,
    "Unit": "Count",
    "Dimensions": [{"Name": "ServiceName", "Value": "queue-worker"}]
  }]'

# Step scaling policy on that metric
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/my-cluster/queue-worker \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name scale-on-queue-depth \
  --policy-type StepScaling \
  --step-scaling-policy-configuration '{
    "AdjustmentType": "ChangeInCapacity",
    "StepAdjustments": [
      {"MetricIntervalLowerBound": 0, "MetricIntervalUpperBound": 100, "ScalingAdjustment": 1},
      {"MetricIntervalLowerBound": 100, "ScalingAdjustment": 3}
    ],
    "Cooldown": 60
  }'
```

**Key decisions:**
- `stopTimeout: 120` — give worker 2 minutes to finish the current message batch before SIGKILL
- No ALB needed — workers are not receiving inbound HTTP requests
- Scale on backlog per task, not just queue depth (avoids over-provisioning at low message volumes)
- Handle poison messages: use a dead-letter queue with a redrive policy so stuck messages don't block workers indefinitely

---

### Pattern 3: Scheduled and one-off batch tasks

**Scenario:** A nightly report needs to run at 02:00 UTC; analysts also trigger ad-hoc data exports on demand.

```bash
# Scheduled task via EventBridge rule
aws events put-rule \
  --name nightly-report \
  --schedule-expression "cron(0 2 * * ? *)" \
  --state ENABLED

aws events put-targets \
  --rule nightly-report \
  --targets '[{
    "Id": "ecs-batch",
    "Arn": "arn:aws:ecs:us-east-1:123456789012:cluster/batch-cluster",
    "RoleArn": "arn:aws:iam::123456789012:role/EventBridgeECSRole",
    "EcsParameters": {
      "TaskDefinitionArn": "arn:aws:ecs:us-east-1:123456789012:task-definition/nightly-report:5",
      "LaunchType": "FARGATE",
      "NetworkConfiguration": {
        "awsvpcConfiguration": {
          "Subnets": ["subnet-aaa"],
          "SecurityGroups": ["sg-batch"],
          "AssignPublicIp": "DISABLED"
        }
      },
      "TaskCount": 1
    }
  }]'

# Ad-hoc / on-demand run (standalone task, not a service)
aws ecs run-task \
  --cluster batch-cluster \
  --task-definition data-export:3 \
  --launch-type FARGATE \
  --network-configuration '{
    "awsvpcConfiguration": {
      "subnets": ["subnet-aaa"],
      "securityGroups": ["sg-batch"],
      "assignPublicIp": "DISABLED"
    }
  }' \
  --overrides '{
    "containerOverrides": [{
      "name": "exporter",
      "environment": [
        {"name": "EXPORT_DATE", "value": "2025-01-15"},
        {"name": "OUTPUT_BUCKET", "value": "my-exports-bucket"}
      ]
    }]
  }'
```

**Key decisions:**
- Standalone tasks (no service) are appropriate for finite jobs—ECS does not restart them on completion
- Use container overrides to parameterize the same task definition for different runs
- Use Fargate Spot for batch to reduce cost by ~70% when interruptions are acceptable
- Implement idempotency and checkpointing so retried tasks do not duplicate work
- ECS does not natively retry failed standalone tasks—handle retries in EventBridge (configure retry attempts) or the calling application

---

### Pattern 4: Blue/green deployment with CodeDeploy

**Scenario:** A production API needs zero-downtime deployments with a validation window and instant rollback capability.

```text
Setup:
  1. ECS Service with deployment controller = CODE_DEPLOY
  2. Two target groups: Blue TG and Green TG
  3. ALB production listener → Blue TG (initially)
  4. ALB test listener (optional, port 8080) → Green TG

Deployment flow:
  CodeDeploy creates new task set in Green TG
       ↓
  Green TG health checks pass
       ↓
  (Optional) Run validation scripts against test listener
       ↓
  Traffic shifts: production listener → Green TG
  (Linear: 10% per minute | Canary: 10% then 90% | All-at-once)
       ↓
  Bake time (e.g., 5 minutes — monitor error rates/latency)
       ↓
  Blue tasks terminated
       ↓
  Rollback: available throughout bake time — shifts traffic back to Blue instantly
```

```bash
# Create the CodeDeploy application and deployment group
aws deploy create-application \
  --application-name my-ecs-app \
  --compute-platform ECS

aws deploy create-deployment-group \
  --application-name my-ecs-app \
  --deployment-group-name my-ecs-dg \
  --deployment-config-name CodeDeployDefault.ECSAllAtOnce \
  --service-role-arn arn:aws:iam::123456789012:role/CodeDeployECSRole \
  --ecs-services '[{"serviceName": "web-api", "clusterName": "web-cluster"}]' \
  --load-balancer-info '{
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "blue-tg"},
        {"name": "green-tg"}
      ],
      "prodTrafficRoute": {"listenerArns": ["arn:aws:elasticloadbalancing:..."]},
      "testTrafficRoute": {"listenerArns": ["arn:aws:elasticloadbalancing:...:listener/..."]}
    }]
  }'

# Trigger a deployment
aws deploy create-deployment \
  --application-name my-ecs-app \
  --deployment-group-name my-ecs-dg \
  --revision '{
    "revisionType": "AppSpecContent",
    "appSpecContent": {
      "content": "{\"version\":0,\"Resources\":[{\"TargetService\":{\"Type\":\"AWS::ECS::Service\",\"Properties\":{\"TaskDefinition\":\"arn:aws:ecs:us-east-1:123456789012:task-definition/web-api:15\",\"LoadBalancerInfo\":{\"ContainerName\":\"api\",\"ContainerPort\":8080}}}}]}"
    }
  }'
```

**When to choose rolling vs blue/green:**

| Criterion | Rolling | Blue/Green (CodeDeploy) |
|---|---|---|
| Setup complexity | Low (built into ECS) | Higher (CodeDeploy app, two target groups) |
| Rollback speed | Slow (re-deploy previous revision) | Instant (traffic shift back) |
| Validation window | No built-in bake time | Configurable bake time |
| Cost during deployment | No extra tasks (min/max % controls) | Double tasks during bake window |
| Traffic splitting (canary) | Not supported natively | Supported (linear, canary configs) |
| Best for | Low-risk updates, cost-sensitive | Production APIs, zero-downtime requirement |

---

## 11. Interview questions and model answers

**Q1: ECS or EKS — how do you decide?**

ECS is the right choice when the team is AWS-focused and wants tighter native integration with ALB, IAM, CloudWatch, and Secrets Manager without a Kubernetes learning curve or control-plane operations cost. EKS fits when you need Kubernetes-specific APIs (Custom Resource Definitions, admission webhooks, Helm ecosystem), have multi-cloud portability requirements, or your team already operates Kubernetes in production.

---

**Q2: What is the difference between a task and a service?**

A task is a single execution of a task definition—a running set of containers. A service is a long-running controller that maintains a desired task count, handles rolling or blue/green deployments, registers tasks with load balancers, and replaces unhealthy tasks. Use standalone tasks for finite jobs (batch, migration scripts) and services for any long-running workload that must stay running.

---

**Q3: Why does the container pull successfully but S3 access fail?**

Image pulls use the **task execution role**; S3 access uses the **task role**. These are two different IAM identities. A common mistake is adding `s3:GetObject` to the execution role—the ECS agent uses it, not the application. The task role also needs to pass the resource policy and KMS key policy checks on the target bucket if encryption is configured.

---

**Q4: How do you deploy without dropping traffic?**

Use a rolling deployment with `minimumHealthyPercent: 100` and `maximumPercent: 200` so ECS launches new tasks before draining old ones. Configure a `healthCheckGracePeriodSeconds` appropriate for the app startup time. Implement graceful shutdown (handle `SIGTERM`, drain in-flight requests). For zero-downtime with instant rollback, switch to CodeDeploy blue/green and set a bake time window.

---

**Q5: How do you run an hour-long batch job on ECS?**

Use a standalone task (no service) with Fargate or Fargate Spot. Schedule it via EventBridge or trigger it via `run-task` from a CI/CD pipeline. Increase `stopTimeout` so the container has time to checkpoint if interrupted. Implement idempotency so retries are safe. For very long jobs (> 4 hours), consider checkpointing state to S3 or DynamoDB and splitting into stages.

---

**Q6: Explain the Fargate vs EC2 cost tradeoff for ECS.**

Fargate bills per vCPU-second and GB-second from task start to stop with no idle host cost. This is cost-efficient for variable workloads where tasks are frequently at zero. EC2 bills per instance-hour regardless of task density, but per-vCPU rates are lower—so high and steady utilization (bin-packing many containers per host) typically costs less on EC2. EC2 also benefits from Reserved Instances and Savings Plans applied to the host layer. As a rule: spiky traffic → Fargate; steady high utilization → EC2 Reserved; batch/flexible → Fargate Spot.

---

**Q7: How does ECS Service Connect work, and how does it differ from Cloud Map?**

Service Connect injects a managed Envoy sidecar into each task. Services register under a logical DNS name in a Service Connect namespace. Inter-service calls use that name; Envoy handles load-balancing, retries, circuit-breaking, and emits per-route CloudWatch metrics automatically. Cloud Map is a lower-level service discovery mechanism: services register their IP/port; clients resolve them via DNS or API. Cloud Map covers non-ECS workloads and is simpler but provides no built-in retries, circuit-breaking, or traffic metrics.

---

**Q8: What are ECS task placement strategies and constraints?**

**Strategies** (where ECS prefers to place tasks):
- `spread` — distribute evenly across attributes (AZ, instance ID) for fault isolation
- `binpack` — pack tasks as densely as possible on available hosts to minimize host count
- `random` — no preference

**Constraints** (hard rules that must be satisfied):
- `distinctInstance` — every task must run on a different host
- `memberOf` — task must run on hosts matching a custom attribute expression (e.g., instance type, AMI)

Strategies can be combined (e.g., spread across AZs first, then binpack within each AZ). Constraints that cannot be satisfied prevent the task from placing at all.

---

**Q9: How do you securely handle secrets in ECS?**

Use AWS Secrets Manager or SSM Parameter Store as the source of truth. Reference them in the task definition `secrets` block—ECS injects them as environment variables at container start using the execution role credentials. The execution role needs `secretsmanager:GetSecretValue` or `ssm:GetParameters`. Use KMS customer-managed keys for at-rest encryption; the execution role also needs `kms:Decrypt`. Never hardcode secrets in images, Dockerfiles, or task definition environment variables. Rotate secrets in Secrets Manager and redeploy tasks to pick up rotated values.

---

**Q10: Rolling deployment vs blue/green — when do you choose each?**

Rolling is simpler to operate (no extra infrastructure, no CodeDeploy setup) and is fine for lower-risk services where brief mixed-version traffic is acceptable and a re-deploy is a fast enough rollback. Blue/green is appropriate for production APIs with zero-downtime requirements: it allows a validation/bake window, enables instant traffic rollback (no re-deploy needed), and supports canary or linear traffic shifting. The tradeoff is double the task count during the bake window and additional CodeDeploy configuration complexity.

---

**Q11: What are ECS task placement strategies vs capacity provider strategies?**

Task placement strategies (spread, binpack, random) control which host within available EC2 capacity receives a new task. Capacity provider strategies control which compute source (FARGATE, FARGATE_SPOT, an ASG) provides the underlying capacity and what weights/base values determine how tasks are distributed across providers. They operate at different layers: capacity providers determine what compute is available; placement strategies determine where on that compute tasks land.

---

**Q12: What are the security implications of ECS Exec?**

ECS Exec opens an SSM session into a running container, bypassing network firewalls and load-balancer restrictions. This is powerful for debugging but represents an elevated-access path. Security implications: (1) Anyone with `ecs:ExecuteCommand` IAM permission can execute arbitrary commands in the container as whatever OS user the container runs as. (2) All ECS Exec sessions should be logged to CloudWatch Logs or S3 via SSM session logging. (3) Disable `--enable-execute-command` on production services when not actively debugging; treat access like SSH access. (4) The task role needs `ssmmessages:*`—scope that permission to task roles that genuinely need it via separate roles for debug vs production.

---

## 12. Quick reference card

### Valid Fargate CPU and memory combinations

| vCPU | Valid memory range |
|---|---|
| 0.25 (256) | 512 MB, 1 GB, 2 GB |
| 0.5 (512) | 1 GB – 4 GB (1 GB increments) |
| 1 (1024) | 2 GB – 8 GB (1 GB increments) |
| 2 (2048) | 4 GB – 16 GB (1 GB increments) |
| 4 (4096) | 8 GB – 30 GB (1 GB increments) |
| 8 (8192) | 16 GB – 60 GB (4 GB increments) |
| 16 (16384) | 32 GB – 120 GB (8 GB increments) |

### Other key limits

| Resource | Limit | Notes |
|---|---|---|
| Tasks per cluster | 5,000 (default) | Soft limit, can be increased |
| Task definitions per account | 1,000,000 | Hard limit |
| Containers per task definition | 10 | Soft limit |
| Services per cluster | 5,000 | Soft limit |
| Fargate ephemeral storage | 20 GB default, up to 200 GB | Additional GB charged separately |
| `stopTimeout` | 2–120 seconds | Default 30 s |
| Health check grace period | 0–2,147,483,647 seconds | Practical range 0–300 s |
| ALB target group: IP targets | Requires `awsvpc` network mode | Instance targets use `bridge`/`host` |

### Compute choice decision tree

```text
Do you need GPU or specialized hardware?
  Yes → EC2 capacity provider (mandatory)
  No  ↓

Do you want zero EC2 management (patching, sizing)?
  Yes → Fargate
  No  ↓

Is the workload steady and highly utilized (>60% avg CPU)?
  Yes → EC2 with Reserved Instances (lower $/vCPU)
  No  → Fargate (no idle host cost)

Is the workload interruption-tolerant (batch/worker)?
  Yes → Fargate Spot (~70% discount)
  No  → Fargate On-Demand
```

### Networking decision table

| Question | awsvpc | bridge |
|---|---|---|
| Required for Fargate? | **Yes** | No |
| Task gets dedicated security group? | **Yes** | No (shares host SG) |
| Load balancer target type | **IP** | Instance |
| Port conflicts on same host? | No (each task has own ENI) | Possible (dynamic host ports needed) |
| ENI limit risk on dense EC2? | **Yes** (one ENI per task) | No |
| Supports Service Connect? | **Yes** | No |
| Recommended for new services? | **Yes** | Only for legacy EC2 workloads |

### Common CLI one-liners

```bash
# Force a new deployment (picks up latest task definition revision)
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --force-new-deployment

# Scale a service manually
aws ecs update-service \
  --cluster my-cluster \
  --service my-service \
  --desired-count 5

# Stop a specific task
aws ecs stop-task \
  --cluster my-cluster \
  --task TASK_ARN \
  --reason "manual stop for debugging"

# Deregister a task definition revision
aws ecs deregister-task-definition \
  --task-definition my-task:42

# List task definition families
aws ecs list-task-definition-families \
  --status ACTIVE

# Get the current image running in a service
aws ecs describe-tasks \
  --cluster my-cluster \
  --tasks $(aws ecs list-tasks --cluster my-cluster --service-name my-service --query 'taskArns[0]' --output text) \
  --query 'tasks[0].containers[*].image' \
  --output text
```

---

## 13. Exam tips summary

### Must-know concepts

1. **Task role vs execution role** — execution role is used by the ECS/Fargate agent (ECR pull, logs, secrets); task role is used by application code. They are independent identities.
2. **awsvpc is mandatory for Fargate** — tasks get a dedicated ENI; use IP target groups, not instance target groups.
3. **Service scaling and host scaling are independent** — more desired tasks cannot place if there is no compute capacity or no free subnet IPs.
4. **Deployment stall cause** — when `minimumHealthyPercent` and `maximumPercent` settings leave no room to launch new tasks or drain old ones, the deployment freezes. Check service events.
5. **Blue/green requires CodeDeploy** — ECS native rolling does not support canary or linear traffic splitting; only CodeDeploy does.
6. **Secrets injection is at task start** — rotating a secret requires a task restart to take effect; running containers do not auto-refresh environment variables.
7. **Health check grace period** — set this on the ECS service to prevent tasks from being killed during slow application startup. Applies to ALB health checks.
8. **Fargate task limits** — only specific CPU/memory combinations are valid. CPU 0.25 with 16 GB memory is not a valid combination.
9. **ECS Exec needs `ssmmessages:*` on the task role** — and `--enable-execute-command` must be set on the service or task.
10. **Capacity provider families must not be mixed** — a strategy using Managed Instances and Fargate together is not supported; stick to one provider family.
11. **Subnet IP exhaustion** — `SubnetAvailableIPAddresses` dropping to zero prevents new tasks from placing even if compute is available.
12. **Image digests over mutable tags** — using `latest` means two deployments of the same revision can pull different images; always pin to a specific digest or immutable tag in production.

### Common exam scenarios

- **EC2-backed ECS tasks can't reach S3** → Check task role (not execution role); also check VPC endpoint or NAT path and bucket policy
- **Fargate task stuck in PENDING** → Subnet has no free IPs, or Fargate quota reached, or capacity provider not linked to the cluster
- **ALB returns 502** → Container binds to `127.0.0.1` (fix: bind to `0.0.0.0`), or container port in task definition doesn't match the target group port
- **New task definition deployed but no change visible** → Service still uses old task definition revision; run `update-service --force-new-deployment`
- **App running on EC2 cluster can't get permissions** → Permissions are on the EC2 instance role, not the task role; task role overrides instance role for application code
- **Deployment stuck with no new tasks launching** → `maximumPercent` too low — raise it to allow extra tasks during the rollout
- **Blue/green deployment can't roll back** → Bake time already expired and blue task set was already terminated; rollback is only available during the bake window
- **Worker not scaling despite long queue** → Scaling policy target metric is `ApproximateNumberOfMessagesVisible` not backlog-per-task; at high task counts the absolute number looks small

### Red flags (anti-patterns to avoid)

- ❌ **Using `latest` as the image tag in production** → Use immutable tags or image digests; `latest` means a re-deploy can silently change the image
- ❌ **Adding application permissions to the execution role** → Application code uses the task role, not the execution role; this is a common access-denied trap
- ❌ **Placing secrets in environment variables in the task definition** → They are stored as plain text in the task definition; use `secrets` block with Secrets Manager/SSM instead
- ❌ **Assigning public IPs to every task for ECR connectivity** → Use NAT Gateway or VPC interface endpoints; public IPs expose tasks to direct internet traffic
- ❌ **Setting `minimumHealthyPercent: 0` in production** → Allows all tasks to be stopped simultaneously, causing a full service outage during deployment
- ❌ **Using bridge mode for new services** → `awsvpc` gives better isolation, supports Service Connect, and is required for Fargate
- ❌ **Ignoring stopped-task reasons and increasing CPU instead** → The stopped reason tells you exactly what failed; OOM kills (exit code 137) need memory tuning, not CPU
- ❌ **No `stopTimeout` configuration for message-processing workers** → Default 30 seconds may be too short to finish a long message batch; set `stopTimeout` to match your max processing time
- ❌ **Enabling ECS Exec permanently on production services** → Treat it like SSH; enable only when actively debugging, and audit session logs
- ❌ **Using a single task definition for all environments** → Separate task definitions (or environment-specific overrides) prevent accidental prod config in staging

---

## 14. AWS sources

- [Managed Instances](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ManagedInstances.html)
- [Managed Instances task parameters](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters-managed-instances.html)
- [Express Mode resources](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/express-service-work.html)
- [Task IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)
- [Execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)
- [Managed scaling behavior](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/managed-scaling-behavior.html)
- [Service deployment behavior](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html)
- [Deployment controllers and strategies](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs_service-options.html)
- [Service Connect overview](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html)
- [ECS Exec](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ecs-exec.html)
- [Task placement strategies](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-strategies.html)
- [Task placement constraints](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement-constraints.html)
- [Application Auto Scaling for ECS](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-auto-scaling.html)
- [Secrets in task definitions](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/specifying-sensitive-data.html)
- [Blue/green deployments with CodeDeploy](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html)
- [Fargate task storage](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/fargate-task-storage.html)

**Related:** [EKS](aws-eks-cheatsheet.md), [ELB](aws-elb-cheatsheet.md), [SQS](aws-sqs-cheatsheet.md).
