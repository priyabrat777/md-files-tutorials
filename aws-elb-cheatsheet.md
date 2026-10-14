# AWS Elastic Load Balancing Cheat Sheet

**Interview preparation: ALB, NLB, GWLB, routing, health checks, and traffic failures.**

**Verified:** 2026-10-13 using AWS documentation. Protocol features and quotas depend on load-balancer type; name the type before quoting a limit.

[All AWS cheatsheets](aws-core-services-interview-index.md) · [EC2](aws-ec2-cheatsheet.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Select the correct load balancer](#2-select-the-correct-load-balancer)
- [3. Routing and target types](#3-routing-and-target-types)
- [4. AZs and cross-zone routing](#4-azs-and-cross-zone-routing)
- [5. Health checks, draining, and fail-open](#5-health-checks-draining-and-fail-open)
- [6. Security and current details](#6-security-and-current-details)
- [7. Performance and cost](#7-performance-and-cost)
- [8. Common patterns](#8-common-patterns)
- [9. Troubleshooting](#9-troubleshooting)
- [10. Quick reference card](#10-quick-reference-card)
- [11. Exam tips summary](#11-exam-tips-summary)
- [12. Interview questions and detailed answers](#12-interview-questions-and-detailed-answers)
- [13. Practical inspection](#13-practical-inspection)
- [14. AWS sources](#14-aws-sources)

---

## 1. Fundamentals and mental model

Elastic Load Balancing distributes traffic to targets. It does not create application capacity; combine it with EC2 Auto Scaling, ECS service scaling, or Kubernetes scaling.

```text
Client → DNS name → load-balancer node → listener → rule → target group → healthy target
                                                        └── health-check configuration
```

Listeners define accepted protocols/ports. Target groups define backends, protocol/port, health checks, and routing attributes. A target can fail health checks even when the instance is running.

## 2. Select the correct load balancer

| Type | Primary job | Typical choice |
|---|---|---|
| Application Load Balancer (ALB) | Layer 7 HTTP/HTTPS routing | Web apps, host/path routing, HTTP APIs, gRPC |
| Network Load Balancer (NLB) | Layer 4 connection/flow routing | TCP/UDP/TLS, static zonal addresses, PrivateLink services |
| Gateway Load Balancer (GWLB) | Transparent traffic distribution to appliances | Firewalls, intrusion inspection, virtual network appliances |
| Classic Load Balancer | Previous-generation load balancing | Existing systems; evaluate migration |

ALB supports features such as redirects, fixed responses, authentication integrations, and WAF association. NLB is useful when application-layer inspection is unnecessary or a transport requirement dictates it. GWLB uses GENEVE for appliance traffic; it is not a replacement for a web ALB.

## 3. Routing and target types

| Requirement | Design |
|---|---|
| `/orders` and `/catalog` to different services | ALB listener rules and separate target groups |
| TLS termination with managed certificates | HTTPS ALB or TLS NLB listener with ACM |
| End-to-end TLS | Encrypt the target connection too; define trust requirements |
| Backend handles the TLS handshake | NLB TCP pass-through |
| Static frontend IP requirement | NLB; ALB addresses can change |
| Private service across VPCs/accounts | Evaluate NLB-backed PrivateLink and permissions |
| Container has its own IP | IP targets; common with ECS `awsvpc` |

ALB target groups support instances, IPs, and Lambda with type-specific restrictions. NLB can target instances, IPs, and an ALB under supported configurations. Never assume every target type supports every listener protocol.

Use the load balancer's DNS name or Route 53 alias. Do not pin discovered ALB IP addresses in client configuration. [How ELB works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)

## 4. AZs and cross-zone routing

| Type | Cross-zone default | Detail |
|---|---|---|
| ALB | Enabled at load-balancer level | Target groups can override to disable where supported |
| NLB | Disabled | Enable deliberately and evaluate inter-AZ cost |
| GWLB | Disabled | Account for appliance placement and flow symmetry |

Cross-zone distribution and multi-AZ redundancy solve different problems. Have healthy targets and sufficient capacity in surviving AZs. A deployment with targets in only one AZ still has a failure concentration.

[Cross-zone behavior](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html), [ALB attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)

## 5. Health checks, draining, and fail-open

- Use a health endpoint that accurately represents readiness to serve traffic.
- Separate liveness from readiness. Restarting every container because a shared database is briefly unavailable can worsen an outage.
- Configure healthy/unhealthy thresholds and startup grace periods for the real boot time.
- Deregistration delay lets in-flight requests complete during removal; application shutdown timeout must align.
- Check load-balancer-specific fail-open behavior. An unhealthy target set must not be treated as a guaranteed traffic block.
- Sticky sessions can help legacy sessions, but external session storage improves replaceability and balancing.

**Deployment question:** a rollout is safe only if new targets become ready before old capacity drains, rollback is possible, and enough capacity remains during an AZ failure.

## 6. Security and current details

For ALB, allow client traffic to the load-balancer security group and allow target traffic from that security group to the application's port. Include the health-check path and port. [ALB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html)

**Outdated claim:** “NLB does not support security groups.” NLB supports them. Associate at least one when creating the NLB if you need the feature: an NLB created without any cannot have them added later. After initial association, the groups can be changed. Health checks are controlled by outbound rules. [NLB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html)

Distinguish TLS encryption from user authorization. An encrypted request can still be unauthorized. ALB authentication and WAF also do not replace authorization inside the application.

## 7. Performance and cost

Load-balancer charges depend on type, running time, and capacity units/traffic dimensions; downstream data-transfer charges also matter. Long-lived connections, new-connection rates, bytes, and ALB rule evaluation can affect cost differently.

Common tuning questions:

- Is latency in the target, connection setup, TLS, or load balancer?
- Do idle timeouts align with client/server keep-alive settings?
- Are request sizes and connection patterns representative of production?
- Is one target overloaded because of stickiness or skewed connections?
- Is the health-check route fast and independent of unnecessary dependencies?

---

## 8. Common patterns

### Pattern 1: Multi-tier web application with ALB path-based routing

**Architecture:**
```
Internet → ALB (internet-facing)
         ├→ /api/* → API Target Group (EC2/ECS)
         ├→ /admin/* → Admin Target Group (EC2)
         ├→ /images/* → S3 Redirect (static assets)
         └→ /* → Frontend Target Group (EC2/ECS)
```

**CLI Setup:**

```bash
# Create ALB
aws elbv2 create-load-balancer \
  --name web-application-alb \
  --subnets subnet-abc123 subnet-def456 \
  --security-groups sg-alb123 \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4

ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names web-application-alb \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# Create target groups
aws elbv2 create-target-group \
  --name api-targets \
  --protocol HTTP \
  --port 8080 \
  --vpc-id vpc-123456 \
  --health-check-path /api/health \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3

API_TG_ARN=$(aws elbv2 describe-target-groups \
  --names api-targets \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Similar for other target groups...

# Create HTTPS listener with certificate
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/abc \
  --default-actions Type=forward,TargetGroupArn=$FRONTEND_TG_ARN

LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query 'Listeners[0].ListenerArn' --output text)

# Add path-based routing rules
# Rule 1: /api/* → API target group (priority 10)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 10 \
  --conditions Field=path-pattern,Values='/api/*' \
  --actions Type=forward,TargetGroupArn=$API_TG_ARN

# Rule 2: /admin/* → Admin target group (priority 20)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 20 \
  --conditions Field=path-pattern,Values='/admin/*' \
  --actions Type=forward,TargetGroupArn=$ADMIN_TG_ARN

# Rule 3: /images/* → Redirect to S3 (priority 30)
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 30 \
  --conditions Field=path-pattern,Values='/images/*' \
  --actions Type=redirect,RedirectConfig='{
    "Protocol":"HTTPS",
    "Host":"static-assets.s3.amazonaws.com",
    "Path":"/#{path}",
    "StatusCode":"HTTP_301"
  }'

# HTTP to HTTPS redirect listener
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=redirect,RedirectConfig='{
    "Protocol":"HTTPS",
    "Port":"443",
    "StatusCode":"HTTP_301"
  }'
```

**Key features:**
- ✅ Path-based routing to different services
- ✅ HTTPS with ACM certificate
- ✅ HTTP→HTTPS redirect
- ✅ Static asset redirect to S3

### Pattern 2: NLB for TCP application with static IPs

**Use case:** Database proxy, MQTT broker, or TCP service requiring static IPs for firewall whitelisting.

```bash
# Create NLB with Elastic IPs
EIP1=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
EIP2=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

aws elbv2 create-load-balancer \
  --name tcp-service-nlb \
  --type network \
  --scheme internet-facing \
  --subnet-mappings \
    SubnetId=subnet-abc123,AllocationId=$EIP1 \
    SubnetId=subnet-def456,AllocationId=$EIP2

NLB_ARN=$(aws elbv2 describe-load-balancers \
  --names tcp-service-nlb \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

# Create TCP target group
aws elbv2 create-target-group \
  --name tcp-backend \
  --protocol TCP \
  --port 5432 \
  --vpc-id vpc-123456 \
  --health-check-protocol TCP \
  --health-check-port 5432 \
  --health-check-interval-seconds 10 \
  --healthy-threshold-count 3 \
  --unhealthy-threshold-count 3

TCP_TG_ARN=$(aws elbv2 describe-target-groups \
  --names tcp-backend \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Create TCP listener
aws elbv2 create-listener \
  --load-balancer-arn $NLB_ARN \
  --protocol TCP \
  --port 5432 \
  --default-actions Type=forward,TargetGroupArn=$TCP_TG_ARN

# Register targets (instances or IPs)
aws elbv2 register-targets \
  --target-group-arn $TCP_TG_ARN \
  --targets Id=i-abc123 Id=i-def456

# Enable cross-zone load balancing (optional, incurs data transfer)
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $NLB_ARN \
  --attributes Key=load_balancing.cross_zone.enabled,Value=true
```

**Benefits:**
- ✅ Static IP addresses (EIPs) for firewall whitelisting
- ✅ Layer 4 (TCP) routing
- ✅ Preserve source IP
- ✅ High throughput, low latency

### Pattern 3: ALB with WAF for DDoS protection

**Architecture:**
```
Internet → CloudFront (optional) → WAF → ALB → Targets
```

**Setup:**

```bash
# Create WAF WebACL
aws wafv2 create-web-acl \
  --name web-app-protection \
  --scope REGIONAL \
  --default-action Block={} \
  --rules file://waf-rules.json \
  --visibility-config \
    SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=WebAppACL

WEBACL_ARN=$(aws wafv2 list-web-acls --scope REGIONAL \
  --query "WebACLs[?Name=='web-app-protection'].ARN" --output text)

# Associate WAF with ALB
aws wafv2 associate-web-acl \
  --web-acl-arn $WEBACL_ARN \
  --resource-arn $ALB_ARN

# Example WAF rules file (waf-rules.json)
```

```json
[
  {
    "Name": "RateLimitRule",
    "Priority": 1,
    "Statement": {
      "RateBasedStatement": {
        "Limit": 2000,
        "AggregateKeyType": "IP"
      }
    },
    "Action": {"Block": {}},
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "RateLimitRule"
    }
  },
  {
    "Name": "AWSManagedRulesCommonRuleSet",
    "Priority": 2,
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesCommonRuleSet"
      }
    },
    "OverrideAction": {"None": {}},
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "AWSManagedRulesCommonRuleSet"
    }
  }
]
```

### Pattern 4: Blue/Green deployment with weighted target groups

**Use case:** Canary or blue/green deployments with traffic shifting.

```bash
# Create two target groups: blue (current) and green (new version)
aws elbv2 create-target-group --name app-blue --protocol HTTP --port 80 --vpc-id vpc-123
aws elbv2 create-target-group --name app-green --protocol HTTP --port 80 --vpc-id vpc-123

BLUE_TG_ARN=$(aws elbv2 describe-target-groups --names app-blue \
  --query 'TargetGroups[0].TargetGroupArn' --output text)
GREEN_TG_ARN=$(aws elbv2 describe-target-groups --names app-green \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Initial: 100% to blue, 0% to green
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups":[
      {"TargetGroupArn":"'"$BLUE_TG_ARN"'","Weight":100},
      {"TargetGroupArn":"'"$GREEN_TG_ARN"'","Weight":0}
    ],
    "TargetGroupStickinessConfig":{"Enabled":false}
  }'

# Phase 1: Canary 10% to green
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups":[
      {"TargetGroupArn":"'"$BLUE_TG_ARN"'","Weight":90},
      {"TargetGroupArn":"'"$GREEN_TG_ARN"'","Weight":10}
    ]
  }'

# Phase 2: 50/50 split
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups":[
      {"TargetGroupArn":"'"$BLUE_TG_ARN"'","Weight":50},
      {"TargetGroupArn":"'"$GREEN_TG_ARN"'","Weight":50}
    ]
  }'

# Phase 3: Full cutover to green
aws elbv2 modify-listener \
  --listener-arn $LISTENER_ARN \
  --default-actions Type=forward,ForwardConfig='{
    "TargetGroups":[
      {"TargetGroupArn":"'"$BLUE_TG_ARN"'","Weight":0},
      {"TargetGroupArn":"'"$GREEN_TG_ARN"'","Weight":100}
    ]
  }'

# After validation, deregister blue targets
```

### Pattern 5: Internal ALB for microservices communication

**Architecture:**
```
VPC
├─ Service A → Internal ALB (service-discovery.local)
│              ├→ /service-b → Service B Target Group
│              ├→ /service-c → Service C Target Group
│              └→ /service-d → Service D Target Group
└─ Private subnets only (no internet access)
```

```bash
# Create internal ALB
aws elbv2 create-load-balancer \
  --name internal-services-alb \
  --subnets subnet-private1 subnet-private2 \
  --security-groups sg-internal \
  --scheme internal \
  --type application

# Register with Route 53 private hosted zone
ZONE_ID=$(aws route53 create-hosted-zone \
  --name internal.example.com \
  --vpc VPCRegion=us-east-1,VPCId=vpc-123456 \
  --caller-reference $(date +%s) \
  --query 'HostedZone.Id' --output text)

# Create alias record
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "services.internal.example.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "'"$ALB_HOSTED_ZONE_ID"'",
          "DNSName": "'"$ALB_DNS_NAME"'",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### Pattern 6: NLB with TLS termination and Proxy Protocol

**Use case:** Preserve client IP while terminating TLS at NLB.

```bash
# Create NLB target group with Proxy Protocol v2
aws elbv2 create-target-group \
  --name app-with-proxy-protocol \
  --protocol TCP \
  --port 8080 \
  --vpc-id vpc-123456 \
  --target-type instance \
  --proxy-protocol-v2 \
  --preserve-client-ip

# Create TLS listener on NLB
aws elbv2 create-listener \
  --load-balancer-arn $NLB_ARN \
  --protocol TLS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:...:certificate/abc \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --alpn-policy HTTP2Preferred

# Enable Proxy Protocol on target (nginx example)
```

**/etc/nginx/nginx.conf:**
```nginx
http {
    server {
        listen 8080 proxy_protocol;
        
        # Extract client IP from Proxy Protocol header
        real_ip_header proxy_protocol;
        set_real_ip_from 10.0.0.0/8;  # VPC CIDR
        
        location / {
            proxy_pass http://backend;
            proxy_set_header X-Real-IP $proxy_protocol_addr;
            proxy_set_header X-Forwarded-For $proxy_protocol_addr;
        }
    }
}
```

### Pattern 7: ALB with Lambda target for serverless API

**Use case:** Route specific paths to Lambda functions.

```bash
# Create Lambda function (assume already exists)
LAMBDA_ARN="arn:aws:lambda:us-east-1:123456789012:function:api-handler"

# Grant ALB permission to invoke Lambda
aws lambda add-permission \
  --function-name api-handler \
  --statement-id alb-invoke \
  --action lambda:InvokeFunction \
  --principal elasticloadbalancing.amazonaws.com \
  --source-arn $TARGET_GROUP_ARN

# Create target group with Lambda target
aws elbv2 create-target-group \
  --name lambda-targets \
  --target-type lambda

LAMBDA_TG_ARN=$(aws elbv2 describe-target-groups --names lambda-targets \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# Register Lambda as target
aws elbv2 register-targets \
  --target-group-arn $LAMBDA_TG_ARN \
  --targets Id=$LAMBDA_ARN

# Add listener rule
aws elbv2 create-rule \
  --listener-arn $LISTENER_ARN \
  --priority 40 \
  --conditions Field=path-pattern,Values='/lambda/*' \
  --actions Type=forward,TargetGroupArn=$LAMBDA_TG_ARN
```

**Lambda handler receives ALB event:**

```python
def lambda_handler(event, context):
    # ALB event structure
    method = event['httpMethod']
    path = event['path']
    headers = event['headers']
    body = event['body']
    
    # Process request
    result = process_request(method, path, body)
    
    # Return ALB-compatible response
    return {
        'statusCode': 200,
        'statusDescription': '200 OK',
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps(result),
        'isBase64Encoded': False
    }
```

---

## 9. Troubleshooting

| Symptom | Likely investigation |
|---|---|
| ALB 502 | Target closes/resets connection, malformed response, TLS/backend mismatch |
| ALB 503 | No usable registered targets; inspect target group state |
| ALB 504 | Target response/connect timeout; network or dependency latency |
| High target response time | Application traces, database calls, target saturation |
| Unhealthy targets | Port/path/matcher, SG/NACL, service binding, startup time |
| Uneven traffic | Stickiness, long connections, cross-zone setting, target capacity |
| Client IP appears wrong | ALB forwarded headers versus NLB preservation/Proxy Protocol |

Use load-balancer and target 5xx metrics separately. Correlate access logs with target logs using request/trace identifiers; average latency can hide tail failures.

---

## 10. Quick reference card

### Load balancer type comparison

| Feature | ALB | NLB | GWLB | CLB (Legacy) |
|---------|-----|-----|------|--------------|
| **OSI Layer** | Layer 7 (Application) | Layer 4 (Transport) | Layer 3 (Network) | Layer 4/7 Hybrid |
| **Protocols** | HTTP, HTTPS, gRPC | TCP, UDP, TLS | GENEVE | TCP, SSL, HTTP, HTTPS |
| **Routing** | Host, path, headers, query | IP, port | Transparent | Basic port |
| **Target types** | Instance, IP, Lambda | Instance, IP, ALB | Instance, IP (appliances) | Instance only |
| **Static IP** | No (DNS only) | Yes (EIP support) | Yes | No |
| **WebSockets** | Yes | Yes | No | Yes |
| **PrivateLink** | No | Yes (endpoint services) | Yes (inspection) | No |
| **TLS termination** | Yes (ACM) | Yes (ACM) | No | Yes |
| **WAF** | Yes | No | No | No |
| **Cognito/OIDC auth** | Yes | No | No | No |
| **Lambda targets** | Yes | No | No | No |
| **Preserve source IP** | Via headers | Native (optional Proxy Protocol) | Yes | Via Proxy Protocol |
| **Cross-zone default** | Enabled | Disabled | Disabled | Disabled (EC2-Classic: enabled) |
| **Health check protocols** | HTTP, HTTPS | TCP, HTTP, HTTPS | TCP, HTTP, HTTPS | TCP, SSL, HTTP, HTTPS |
| **Use case** | Web apps, APIs, microservices | TCP/UDP, static IP, PrivateLink | Firewall, IDS/IPS, deep packet inspection | Legacy (migrate to ALB/NLB) |

### ALB routing capabilities

| Routing Type | Condition | Example |
|--------------|-----------|---------|
| **Host-based** | Host header | `api.example.com` → API target group |
| **Path-based** | URL path | `/images/*` → Static assets |
| **HTTP header** | Custom headers | `X-Custom-Header: mobile` → Mobile targets |
| **HTTP method** | GET, POST, etc. | `POST /api/*` → Write API |
| **Query string** | Query parameters | `?version=v2` → V2 targets |
| **Source IP** | Client CIDR | `10.0.0.0/8` → Internal targets |
| **Fixed response** | Return HTTP directly | `503` with custom message |
| **Redirect** | HTTP redirect | HTTP → HTTPS, old domain → new domain |
| **Weighted** | Traffic splitting | 90% blue, 10% green |

### Target group health check parameters

| Parameter | Default | Range | Notes |
|-----------|---------|-------|-------|
| **Protocol** | HTTP | TCP, HTTP, HTTPS | Match target protocol or use HTTP for health |
| **Port** | Traffic port | 1-65535 or "traffic port" | Can differ from traffic port |
| **Path** | `/` | Any valid path | Should be lightweight endpoint |
| **Interval** | 30s | 5-300s | How often to check |
| **Timeout** | 5s | 2-120s | Must be < interval |
| **Healthy threshold** | 5 | 2-10 | Consecutive successes to mark healthy |
| **Unhealthy threshold** | 2 | 2-10 | Consecutive failures to mark unhealthy |
| **Success codes** | 200 | 200-499 | HTTP status codes considered healthy |
| **Matcher** | N/A | HTTP status or gRPC code | For ALB/NLB HTTP checks |

**Formula:** Minimum time to become healthy = Interval × Healthy threshold
**Example:** 30s × 5 = 150 seconds (2.5 minutes)

### Connection draining / deregistration delay

| Parameter | Default | Range | Purpose |
|-----------|---------|-------|---------|
| **Deregistration delay** | 300s | 0-3600s | Time to complete in-flight requests before removing target |
| **Connection draining (CLB)** | 300s | 1-3600s | Legacy equivalent for CLB |

**Best practice:** Set to match longest expected request duration (e.g., 60s for API, 300s for file uploads).

### Key security group patterns

**ALB security groups:**
```
ALB Security Group (sg-alb)
├─ Inbound: 
│  ├─ HTTP (80) from 0.0.0.0/0
│  └─ HTTPS (443) from 0.0.0.0/0
└─ Outbound:
   └─ All traffic to 0.0.0.0/0 (or restrict to target SG)

Target Security Group (sg-targets)
├─ Inbound:
│  ├─ Application port (8080) from sg-alb
│  └─ Health check port from sg-alb
└─ Outbound:
   └─ As needed (databases, APIs)
```

**NLB with security groups:**
```
NLB Security Group (sg-nlb) - Optional
├─ Inbound:
│  └─ TCP (port) from client CIDR
└─ Outbound:
   └─ TCP (port) to target SG

Target Security Group
├─ Inbound:
│  └─ TCP (port) from sg-nlb OR NLB private IPs
└─ Outbound:
   └─ As needed
```

### Client IP preservation

| LB Type | Method | Header/Protocol | Target Sees |
|---------|--------|-----------------|-------------|
| **ALB** | HTTP headers | `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Port` | ALB private IP (use headers) |
| **NLB (default)** | Native preservation | None (source IP preserved) | Original client IP |
| **NLB (Proxy Protocol)** | Proxy Protocol v2 | Binary protocol prepended | Client IP via protocol |
| **NLB (TLS termination)** | Configurable | Preserve or use Proxy Protocol | Depends on config |
| **CLB** | HTTP headers or Proxy Protocol | `X-Forwarded-For` or Proxy Protocol | CLB IP (use headers/protocol) |

### ALB vs NLB decision matrix

| Requirement | Choose | Reason |
|-------------|--------|--------|
| HTTP/HTTPS web application | ALB | Layer 7 routing, WAF, auth, Lambda |
| Path-based or host-based routing | ALB | Native HTTP routing |
| WebSocket support | Both | ALB for HTTP upgrade, NLB for raw TCP |
| gRPC application | ALB | Native gRPC support |
| Need static IP addresses | NLB | Elastic IP support |
| Extreme performance (millions RPS) | NLB | Lower latency, higher throughput |
| PrivateLink service | NLB or GWLB | Only NLB/GWLB support VPC endpoints |
| TCP/UDP non-HTTP protocol | NLB | Layer 4 transport |
| Need to preserve source IP | NLB | Native IP preservation |
| SSL/TLS offload with path routing | ALB | Application layer visibility |
| Firewall/IDS/IPS appliances | GWLB | Transparent packet inspection |

### Access log fields

**Key ALB access log fields:**
- `type`: http, https, h2, grpc, ws, wss
- `time`: Request timestamp
- `elb`: Load balancer resource ID
- `client:port`: Client IP and port
- `target:port`: Target IP and port (or Lambda ARN)
- `request_processing_time`: Time to route request
- `target_processing_time`: Target response time
- `response_processing_time`: Time to send response
- `elb_status_code`: Load balancer status
- `target_status_code`: Target status
- `received_bytes`: Request size
- `sent_bytes`: Response size
- `request`: HTTP method, URI, protocol
- `user_agent`: Client user agent
- `ssl_cipher`, `ssl_protocol`: TLS details
- `target_group_arn`: Which target group handled request
- `trace_id`: X-Amzn-Trace-Id for X-Ray tracing

**Enable access logs:**
```bash
aws elbv2 modify-load-balancer-attributes \
  --load-balancer-arn $ALB_ARN \
  --attributes \
    Key=access_logs.s3.enabled,Value=true \
    Key=access_logs.s3.bucket,Value=my-alb-logs \
    Key=access_logs.s3.prefix,Value=prod-alb
```

### CloudWatch metrics

**Load balancer metrics:**
- `ActiveConnectionCount`: Current connections
- `NewConnectionCount`: New connections per minute
- `ProcessedBytes`: Bytes processed
- `ConsumedLCUs`: Load Balancer Capacity Units consumed
- `ClientTLSNegotiationErrorCount`: TLS handshake failures
- `HTTPCode_ELB_4XX_Count`: 4xx from load balancer
- `HTTPCode_ELB_5XX_Count`: 5xx from load balancer

**Target metrics:**
- `HealthyHostCount`: Healthy targets
- `UnHealthyHostCount`: Unhealthy targets
- `TargetResponseTime`: Average response time
- `HTTPCode_Target_2XX_Count`: 2xx from targets
- `HTTPCode_Target_4XX_Count`: 4xx from targets
- `HTTPCode_Target_5XX_Count`: 5xx from targets
- `RequestCount`: Requests per target group

### Common HTTP status codes

| Code | Source | Meaning |
|------|--------|---------|
| **200** | Target | Success |
| **301/302** | ALB rule | Redirect action |
| **400** | ALB | Malformed request |
| **401** | ALB (auth) or Target | Unauthorized |
| **403** | ALB (WAF), Target | Forbidden |
| **404** | Target | Not found |
| **460** | ALB | Client closed connection before completion |
| **463** | ALB | X-Forwarded-For header >30 IPs |
| **464** | ALB | Protocol mismatch (HTTP/HTTPS) |
| **500** | Target | Internal server error |
| **502** | ALB | Bad gateway (target closed connection, invalid response) |
| **503** | ALB | No healthy targets |
| **504** | ALB | Gateway timeout (target didn't respond) |

### Cost factors

| Factor | ALB | NLB |
|--------|-----|-----|
| **Base charge** | $0.0225/hour | $0.0225/hour |
| **Capacity units** | LCU: $0.008/hour | NLCU: $0.006/hour |
| **LCU dimensions** | New connections, active connections, processed bytes, rule evaluations | New connections/flows, active connections/flows, processed bytes |
| **Data transfer** | Standard AWS rates | Standard AWS rates |
| **Cross-zone** | Free | $0.01/GB inter-AZ |

**LCU calculation:** Pay for highest dimension:
- 25 new connections/sec
- 3,000 active connections/min
- 1 GB/hour (2.5 Mbps) for TCP/UDP/TLS or 1,000 rule evaluations/sec (ALB)

---

## 11. Exam tips summary

### Must-know concepts

1. **ALB = Layer 7, NLB = Layer 4, GWLB = Layer 3.** ALB inspects HTTP; NLB forwards TCP/UDP flows; GWLB routes to appliances.

2. **Only NLB provides static IP addresses.** ALB DNS names resolve to changing IPs. NLB supports Elastic IPs.

3. **Cross-zone default differs by type.** ALB: enabled. NLB/GWLB: disabled. Enable deliberately for even distribution.

4. **NLB DOES support security groups** (since 2023). Must be associated at creation; can be modified after.

5. **Target health ≠ instance health.** EC2 instance can be running but target unhealthy (wrong port, failing health check path).

6. **Deregistration delay allows graceful shutdown.** Set based on longest request duration. Default 300s.

7. **ALB adds headers for client info.** `X-Forwarded-For` (client IP), `X-Forwarded-Proto` (HTTP/HTTPS), `X-Forwarded-Port`.

8. **NLB preserves source IP by default.** No headers needed. Can enable Proxy Protocol v2 for additional metadata.

9. **Only ALB supports Lambda targets.** NLB cannot invoke Lambda directly.

10. **WAF only works with ALB and CloudFront.** Cannot attach WAF to NLB or GWLB.

11. **502 = target problem, 503 = no targets, 504 = timeout.** ALB error codes indicate different failure modes.

12. **Health check timing matters.** Healthy = Interval × Healthy threshold. 30s × 5 = 150s minimum.

13. **ALB supports authentication.** Cognito user pools and OIDC providers. NLB does not.

14. **Only NLB supports PrivateLink.** Create VPC endpoint services. ALB cannot.

15. **Sticky sessions can cause imbalance.** Use application-managed sessions (Redis, DynamoDB) for better scaling.

### Common exam scenarios

| Scenario | Correct Answer | Wrong Answers to Avoid |
|----------|----------------|------------------------|
| Route `/api` and `/web` to different services | ALB with path-based routing | ❌ Two load balancers, ❌ NLB |
| Need static IP for firewall whitelist | NLB with Elastic IPs | ❌ ALB (no static IP), ❌ Route 53 |
| Web application with DDoS protection | ALB + WAF | ❌ NLB (no WAF support) |
| Preserve client IP for TCP application | NLB (native preservation) | ❌ ALB (only headers) |
| Blue/green deployment with traffic shifting | ALB weighted target groups | ❌ Route 53 weighted records |
| VPC endpoint service (PrivateLink) | NLB as endpoint service | ❌ ALB (not supported) |
| Authenticate users before reaching app | ALB with Cognito or OIDC | ❌ NLB, ❌ Application-only auth |
| High-performance UDP application | NLB (UDP support) | ❌ ALB (no UDP), ❌ GWLB |
| Route based on HTTP headers | ALB with header-based routing | ❌ NLB (Layer 4 only) |
| Targets spread across 3 AZs evenly | Enable cross-zone load balancing | ❌ Default NLB (disabled) |
| Lambda function behind load balancer | ALB with Lambda target | ❌ NLB (no Lambda support) |
| Firewall appliances in traffic path | GWLB | ❌ ALB, ❌ NLB |

### Red flags (wrong answer indicators)

- ❌ **"Use NLB for HTTP routing by path"** → Layer 4 doesn't see paths; use ALB
- ❌ **"ALB provides static IPs"** → ALB IPs change; use NLB for static IPs
- ❌ **"NLB doesn't support security groups"** → Outdated; NLB supports SGs since 2023
- ❌ **"Attach WAF to NLB"** → WAF only works with ALB and CloudFront
- ❌ **"NLB can invoke Lambda"** → Only ALB supports Lambda targets
- ❌ **"Cross-zone is always enabled"** → Only ALB; NLB/GWLB default is disabled
- ❌ **"ALB preserves client source IP"** → Uses X-Forwarded-For header, not native
- ❌ **"Health check port must match traffic port"** → Can be different
- ❌ **"502 means no targets available"** → 503 means no targets; 502 is bad gateway
- ❌ **"Create PrivateLink with ALB"** → Only NLB/GWLB support VPC endpoints

### Quick decision flowchart

```
Need load balancer?
  │
  ├─ HTTP/HTTPS web application?
  │  └─ YES → ALB
  │     ├─ Need path/host routing? → ALB with rules
  │     ├─ Need WAF/DDoS protection? → ALB + WAF
  │     ├─ Need authentication? → ALB with Cognito/OIDC
  │     └─ Need Lambda targets? → ALB
  │
  ├─ Need static IP addresses?
  │  └─ YES → NLB with Elastic IPs
  │
  ├─ TCP/UDP non-HTTP protocol?
  │  └─ YES → NLB
  │
  ├─ Extreme performance required?
  │  └─ YES → NLB (lower latency)
  │
  ├─ PrivateLink service?
  │  └─ YES → NLB
  │
  ├─ Firewall/IDS/IPS appliances?
  │  └─ YES → GWLB
  │
  └─ Default → ALB (most flexible)
```

---

## 12. Interview questions and detailed answers

Read-only examples; substitute your ARNs.

```bash
aws elbv2 describe-load-balancers --names app-alb
aws elbv2 describe-target-health --target-group-arn TARGET_GROUP_ARN
aws elbv2 describe-listeners --load-balancer-arn LOAD_BALANCER_ARN
aws elbv2 describe-load-balancer-attributes --load-balancer-arn LOAD_BALANCER_ARN
aws elbv2 describe-target-group-attributes --target-group-arn TARGET_GROUP_ARN
```

Inspect the target-health reason code before changing thresholds. Verify rules against the request's actual hostname and path.

---

## 13. Practical inspection

Read-only examples; substitute your ARNs.

```bash
aws elbv2 describe-load-balancers --names app-alb
aws elbv2 describe-target-health --target-group-arn TARGET_GROUP_ARN
aws elbv2 describe-listeners --load-balancer-arn LOAD_BALANCER_ARN
aws elbv2 describe-load-balancer-attributes --load-balancer-arn LOAD_BALANCER_ARN
aws elbv2 describe-target-group-attributes --target-group-arn TARGET_GROUP_ARN
```

Inspect the target-health reason code before changing thresholds. Verify rules against the request's actual hostname and path.

---

## 14. AWS sources

- [ELB architecture and cross-zone behavior](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [ALB configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)
- [ALB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html)
- [NLB configuration](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html)
- [NLB security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html)
- [GWLB architecture](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/getting-started.html)

**Related:** [Route 53](aws-route53-cheatsheet.md), [API Gateway](aws-api-gateway-cheatsheet.md), [ECS](aws-ecs-cheatsheet.md).
