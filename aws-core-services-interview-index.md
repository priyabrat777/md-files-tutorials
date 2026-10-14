# AWS Core Services Interview Cheatsheets

**Research and verification date: October 13, 2026.**

These 16 service guides combine fundamentals, comparison tables, architecture tradeoffs, security, troubleshooting, inspection examples, interview answers, and revision notes. Each guide links the AWS documentation used to check current behavior.

## Service guides

| Area | Guide | Main interview themes |
|---|---|---|
| Compute | [EC2](aws-ec2-cheatsheet.md) | Instance lifecycle, capacity versus discounts, storage, HA, scaling |
| Traffic | [Elastic Load Balancing](aws-elb-cheatsheet.md) | ALB/NLB/GWLB, health checks, cross-zone behavior, draining |
| Relational data | [RDS](aws-rds-cheatsheet.md) | Multi-AZ types, replicas, backups, connections, failover |
| Relational data | [Aurora](aws-aurora-cheatsheet.md) | Shared storage, reader endpoints, Serverless v2, Global Database |
| Containers | [ECS](aws-ecs-cheatsheet.md) | Task/service model, IAM roles, Fargate, Managed Instances |
| Kubernetes | [EKS](aws-eks-cheatsheet.md) | Access entries, Pod Identity, networking, Auto Mode, upgrades |
| Messaging | [SNS](aws-sns-cheatsheet.md) | Fanout, filters, FIFO, subscription DLQs, payload configuration |
| Object storage | [S3](aws-s3-cheatsheet.md) | Consistency, storage classes, versioning, replication, Object Lock |
| Queues | [SQS](aws-sqs-cheatsheet.md) | Visibility, retention, FIFO groups, idempotency, redrive |
| Events | [EventBridge](aws-eventbridge-cheatsheet.md) | Classic/new buses, FIFO subscribers, replay, Pipes, Scheduler |
| Observability | [CloudWatch](aws-cloudwatch-cheatsheet.md) | Metrics/logs/traces, alarms, retention, Database Insights |
| APIs | [API Gateway](aws-api-gateway-cheatsheet.md) | REST/HTTP/WebSocket, auth, private integrations, streaming |
| NoSQL data | [DynamoDB](aws-dynamodb-cheatsheet.md) | Keys/indexes, capacity math, transactions, MREC/MRSC |
| DNS | [Route 53](aws-route53-cheatsheet.md) | Aliases, routing, DNS caching, private zones, hybrid resolution |
| Encryption | [KMS](aws-kms-cheatsheet.md) | Envelope encryption, key policies, rotation, multi-Region keys |
| Serverless compute | [Lambda](aws-lambda-cheatsheet.md) | 15/90-minute distinction, retries, concurrency, durable functions |

## Current facts that older notes often miss

| Older shorthand | Current interview answer | Verification |
|---|---|---|
| Lambda always stops at 15 minutes | Eligible asynchronous/ESM invocations on Managed Instances support 90 minutes; exclusions and other invocation limits remain | [Lambda quotas](https://docs.aws.amazon.com/lambda/latest/dg/gettingstarted-limits.html) |
| S3 objects max out at 5 TB | AWS now advertises 50 TB; see the guide for precise units | [S3 announcement](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-maximum-object-size-50-tb/) |
| SQS maximum message is 256 KiB | Maximum is 1 MiB | [SQS quotas](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/quotas-messages.html) |
| SNS maximum is always 256 KiB | Default is 256 KiB; configurable 1 MiB with endpoint/subscription restrictions | [SNS feature guide](https://docs.aws.amazon.com/sns/latest/dg/large-message-payloads.html) |
| DynamoDB global tables are always eventual | Choose MREC or MRSC; capabilities and latency differ | [Read consistency](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html) |
| EventBridge cannot offer FIFO or native retention | New Custom Event Bus adds these; Classic differs | [Bus comparison](https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-custom-bus-what-is.html) |
| API Gateway always has a 29-second ceiling | API type and response mode matter; REST streaming and approved Regional/private timeout increases change the answer | [Streaming](https://docs.aws.amazon.com/apigateway/latest/developerguide/response-transfer-mode.html) |
| Aurora Serverless v2 cannot reach zero | Supported versions/configurations can auto-pause at zero ACU | [Aurora auto-pause](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2-auto-pause.html) |
| Multi-AZ standby never serves reads | True for Multi-AZ DB instance standby; Multi-AZ DB cluster has readable instances | [RDS deployment types](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html) |
| NLB has no security groups | Supported, with a creation-time association constraint | [NLB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html) |
| Customer KMS rotation is only annual | Eligible automatic rotation periods are configurable; on-demand support depends on key type/origin | [Rotation](https://docs.aws.amazon.com/kms/latest/developerguide/rotate-keys.html) |
| Performance Insights is the current standalone monitoring answer | Use Database Insights and check the selected mode | [Database Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Database-Insights.html) |

## Suggested study order

1. **Foundation:** existing IAM/VPC notes, then EC2, ELB, Route 53, and KMS.
2. **Data:** S3, RDS, Aurora, and DynamoDB. Compare consistency, availability, backups, and RPO/RTO.
3. **Messaging:** SQS, SNS, and EventBridge. Explain delivery versus processing and design retry-safe effects.
4. **Application platforms:** Lambda, API Gateway, ECS, and EKS. Compare execution model, scaling, security, and cost.
5. **Operations:** CloudWatch, then revisit troubleshooting in each service guide.

For each service, practice a 60-second explanation, one comparison, one failure scenario, and one design tradeoff. Use the questions near the end of each file as oral prompts.

## Cross-service design drills

| Scenario | Services to connect | Questions to answer |
|---|---|---|
| Resilient checkout API | Route 53 → API Gateway/ALB → Lambda/ECS → Aurora/RDS | How do retries avoid duplicate orders? What happens during DB failover? |
| Image processing | S3 → SQS/EventBridge → Lambda/ECS → S3 | How are duplicates, large objects, and poison inputs handled? |
| Independent order consumers | SNS/EventBridge → separate SQS queues → workers | Which layer owns retries, ordering, and replay? |
| Multi-Region application | Route 53 + regional compute + Aurora Global/DynamoDB global tables | What consistency, RPO, RTO, and writer-fencing guarantees exist? |
| Private container platform | ECS/EKS + VPC endpoints + ELB + IAM/KMS | Which identity calls each API, and which network path carries it? |
| Production incident | CloudWatch + service events/logs + CloudTrail | Is the cause deployment, permissions, saturation, or dependency failure? |

## How to handle quotas and changing documentation

- Name the exact service mode, API, engine version, Region, and invocation path before quoting a number.
- Distinguish a fixed maximum from a default or adjustable account quota.
- Recheck linked live feature/API pages before implementation or an interview that depends on a recent launch.
- Where old overview/quota pages and a new feature guide differ, retain the configuration-specific distinction instead of silently picking one number. The SNS guide explicitly notes this case.
- CLI blocks are study/inspection examples, not evidence of execution against an AWS account. Replace identifiers and confirm profile/Region before use.
- These are interview guides, not an assertion that every feature appears on a particular certification exam. Exam coverage can lag current service behavior.

## Existing companion notes

- [IAM cheatsheet](aws-iam-cheatsheet.md)
- [VPC networking cheatsheet](aws-vpc-networking-cheatsheet.md)
- [Encryption and decryption cheatsheet](aws-encryption-decryption-cheatsheet.md)
- [Architecture decision cheatsheet](aws-saa-c03-architecture-decision-cheat-sheet.md)
- [Scenario practice questions](aws-saa-c03-tricky-scenario-practice-questions.md)
- [Six-week interview preparation kit](interview-preparation/README.md)

The existing companion notes were not re-audited as part of this addition. For overlapping recent service changes, use the dated new guides and their AWS sources.
