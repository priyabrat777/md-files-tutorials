# Amazon Route 53 Cheat Sheet

**Interview preparation: DNS, routing policies, health checks, private resolution, and failover.**

**Verified:** 2026-10-13. Routing-policy and alias-target support differ between public and private hosted zones.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Record types and aliases](#2-record-types-and-aliases)
- [3. Routing policies](#3-routing-policies)
- [4. Health checks and failover](#4-health-checks-and-failover)
- [5. TTL and realistic recovery time](#5-ttl-and-realistic-recovery-time)
- [6. Private hosted zones and split DNS](#6-private-hosted-zones-and-split-dns)
- [7. Hybrid DNS and VPC Resolver](#7-hybrid-dns-and-vpc-resolver)
- [8. Security and operating practices](#8-security-and-operating-practices)
- [9. Troubleshooting and practical inspection](#9-troubleshooting-and-practical-inspection)
- [10. Common patterns](#10-common-patterns)
- [11. Quick reference card](#11-quick-reference-card)
- [12. Exam tips summary](#12-exam-tips-summary)
- [13. Interview Q&A](#13-interview-qa)
- [14. Quick revision](#14-quick-revision)
- [15. AWS sources](#15-aws-sources)

---

## 1. Fundamentals and mental model

Route 53 provides authoritative DNS, domain registration capabilities, health checks, and VPC DNS resolution integrations. DNS answers tell clients where to connect; DNS is not the application's packet-forwarding path.

```text
Client → recursive resolver/cache → authoritative hosted zone → DNS answer
Client ─────────────────────────────────────────────────────→ application endpoint
```

| Term | Meaning |
|---|---|
| Registrar | Manages domain registration/delegation relationship |
| Hosted zone | Authoritative records for a DNS namespace |
| NS delegation | Identifies authoritative name servers |
| TTL | Caching lifetime supplied with an answer |
| Public zone | Internet-resolvable namespace |
| Private zone | Namespace resolved through associated VPC/hybrid setup |

Buying a domain, creating a hosted zone, and correctly delegating it are separate steps. Multiple hosted zones with the same name do not automatically share records or delegation.

## 2. Record types and aliases

| Record | Use |
|---|---|
| A / AAAA | IPv4 / IPv6 address |
| CNAME | Another DNS name; unsuitable at zone apex |
| Alias A/AAAA | Route 53 extension for supported targets, including apex |
| MX | Mail delivery targets |
| TXT | Text verification and policy records |
| NS / SOA | Delegation and zone authority metadata |

An alias is not a universal CNAME replacement for arbitrary external names. It supports specified AWS resources or suitable records in the same zone. Alias responses use the selected record type, such as A/AAAA. [Alias versus non-alias](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)

For an ALB-backed root domain, an alias is normally appropriate. Do not copy transient ALB IP addresses into static A records.

## 3. Routing policies

| Policy | Intent | Important qualification |
|---|---|---|
| Simple | One logical destination | No health-based policy selection |
| Weighted | Relative distribution, rollout | DNS cache/resolver behavior prevents exact per-request percentages |
| Latency | Choose among configured Regions using latency data | Not a geographic-distance guarantee |
| Failover | Primary/secondary | Health and caching determine observed cutover |
| Geolocation | Policy by query origin location | Configure default behavior for unmatched locations |
| Geoproximity | Geographic proximity with bias | Bias changes relative routing areas |
| Multivalue answer | Return several healthy records | Not a full substitute for a load balancer |
| IP-based | Route by configured client CIDR mapping | Check public/private support |

[Routing policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)

Use policy combinations only when their failure behavior remains understandable. More layers make diagnosis and validation harder.

## 4. Health checks and failover

Route 53 can evaluate endpoint checks, calculated checks, and supported CloudWatch-alarm-based checks. Alias `EvaluateTargetHealth` uses supported target health behavior and is not the same as adding an arbitrary HTTP probe.

Direct endpoint health checkers cannot probe private/non-routable IPs. For private services, use supported target-health integration or publish internal health to an appropriate alarm-based check. [Health-check behavior](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-simple-configs.html)

**Failure caveat:** when all eligible records are unhealthy, Route 53 can still return an answer under its policy-specific fallback rules. For primary/secondary failover with both unhealthy, the primary can be returned. A health check is not a guaranteed network kill switch. [Unhealthy-record selection](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-how-route-53-chooses-records.html)

## 5. TTL and realistic recovery time

DNS failover includes health detection, authoritative answer changes, recursive/client caches, connection reuse, and application reconnect time.

Lowering TTL before a migration helps only after existing cached answers expire. Existing TCP connections do not automatically move when DNS changes. Some applications cache longer than intended, so validate actual client behavior.

**Interview answer:** “Failover in 30 seconds” cannot be justified solely by setting TTL to 30 seconds. Explain detection and cache behavior, then give a tested recovery objective.

## 6. Private hosted zones and split DNS

Associate private zones with intended VPCs and enable the required VPC DNS settings. A private zone can share a name with a public zone to provide split-view DNS.

If a matching private zone exists but the requested record is missing, resolution does not simply fall back to the identically named public zone. Overlapping zones and Resolver rules must be planned deliberately.

Private hosted zones support several routing policies, including failover; it is outdated to say “private DNS cannot fail over.” Direct public health probes still cannot reach private addresses. [Private hosted-zone considerations](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.html)

DNS visibility also does not grant network access: routes, firewalls, and endpoint authorization are separate.

## 7. Hybrid DNS and VPC Resolver

| Component | Direction/purpose |
|---|---|
| Inbound Resolver endpoint | On-premises/connected clients query AWS-resolvable names |
| Outbound Resolver endpoint + rules | AWS forwards selected queries to external DNS |
| Conditional forwarding rule | Select destination resolvers by domain suffix |
| DNS Firewall | Domain-level DNS filtering under configured rules |

Deploy resilient endpoint capacity across AZs and permit required DNS traffic. Avoid forwarding loops between on-premises and AWS. A private hosted zone alone does not expose its names to every corporate resolver. [VPC Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)

## 8. Security and operating practices

Protect domain registration, record changes, delegation, and certificate-validation records with least privilege and change review. Losing DNS control can redirect users even if the application itself is secure.

DNSSEC provides integrity/authenticity of DNS data when the validation chain is configured; it does not encrypt DNS queries or replace TLS. Query logging and CloudTrail answer different questions: resolution activity versus configuration/API changes.

For an important migration, test the new target first, reduce TTL early, change controlled records, observe traffic/errors, and keep rollback ready. Include negative caching when debugging recently added records.

## 9. Troubleshooting and practical inspection

| Symptom | Investigate |
|---|---|
| Domain resolves to old address | Resolver/client cache, authoritative record, long-lived connection |
| Public DNS works, VPC fails | Private zone overlap, missing record, Resolver rule precedence |
| Zone edits have no effect | Wrong hosted-zone ID or registrar delegation |
| Private name fails on-premises | Inbound endpoint, forwarding, network/SG, VPC association |
| Health check fails but app works locally | Public reachability, checker permissions, path/Host/TLS configuration |
| Weighted split looks uneven | Cached answers, shared recursive resolvers, sample size |

Read-only examples:

```bash
aws route53 list-hosted-zones-by-name --dns-name example.com
aws route53 list-resource-record-sets --hosted-zone-id HOSTED_ZONE_ID
aws route53 get-health-check-status --health-check-id HEALTH_CHECK_ID
dig example.com A
dig example.com NS
```

Compare authoritative answers with recursive answers and test from the network where the failure occurs.

---

## 10. Common Patterns

### Pattern 1: Active-Active Multi-Region with Weighted Routing
```bash
# Create health checks for each region
aws route53 create-health-check \
  --caller-reference $(date +%s)-us-east \
  --health-check-config \
    IPAddress=52.1.2.3,Port=443,Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=api.example.com

aws route53 create-health-check \
  --caller-reference $(date +%s)-eu-west \
  --health-check-config \
    IPAddress=34.240.5.6,Port=443,Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=api-eu.example.com

# Create weighted records with health checks
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "us-east-1",
        "Weight": 70,
        "TTL": 60,
        "ResourceRecords": [{"Value": "52.1.2.3"}],
        "HealthCheckId": "abc123-us"
      }
    }, {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "eu-west-1",
        "Weight": 30,
        "TTL": 60,
        "ResourceRecords": [{"Value": "34.240.5.6"}],
        "HealthCheckId": "xyz789-eu"
      }
    }]
  }'
```

**Use Case:** 70% traffic to US, 30% to EU; automatic failover if region unhealthy

---

### Pattern 2: Blue/Green Deployment with Weighted Routing
```bash
# Start: 100% blue, 0% green
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "SetIdentifier": "blue",
        "Weight": 100,
        "TTL": 60,
        "AliasTarget": {
          "HostedZoneId": "Z215337",
          "DNSName": "blue-alb-123.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }, {
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "www.example.com",
        "Type": "A",
        "SetIdentifier": "green",
        "Weight": 0,
        "TTL": 60,
        "AliasTarget": {
          "HostedZoneId": "Z215337",
          "DNSName": "green-alb-456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'

# Gradual shift: 90/10 → 50/50 → 0/100
# Final: flip weights to 0% blue, 100% green
```

**Timeline:**
```
T+0:  Blue=100%, Green=0%   (current production)
T+10: Blue=90%,  Green=10%  (canary testing)
T+20: Blue=50%,  Green=50%  (split traffic)
T+30: Blue=0%,   Green=100% (full cutover)
```

---

### Pattern 3: Disaster Recovery with Failover Routing
```bash
# Create health check for primary
aws route53 create-health-check \
  --caller-reference $(date +%s)-primary \
  --health-check-config \
    Type=HTTPS_STR_MATCH,ResourcePath=/health,FullyQualifiedDomainName=api.example.com,Port=443,SearchString='"status":"ok"'

# Primary record (us-east-1)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "primary",
        "Failover": "PRIMARY",
        "TTL": 60,
        "HealthCheckId": "hc-primary",
        "AliasTarget": {
          "HostedZoneId": "Z215337",
          "DNSName": "primary-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }, {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "secondary",
        "Failover": "SECONDARY",
        "TTL": 60,
        "AliasTarget": {
          "HostedZoneId": "Z1234567",
          "DNSName": "dr-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

**Behavior:** Serves primary while healthy, fails over to secondary when health check fails (typically 30-60 seconds)

---

### Pattern 4: Geolocation Routing for GDPR Compliance
```bash
# EU users → EU datacenter
aws route53 change-resource-record-sets \
  --hosted-zone-id Z123456 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "eu-users",
        "GeoLocation": {"ContinentCode": "EU"},
        "TTL": 300,
        "AliasTarget": {
          "HostedZoneId": "Z1234567",
          "DNSName": "eu-alb.eu-west-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }, {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "SetIdentifier": "default",
        "GeoLocation": {"ContinentCode": "*"},
        "TTL": 300,
        "AliasTarget": {
          "HostedZoneId": "Z215337",
          "DNSName": "us-alb.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

**Use Case:** Keep EU customer data in EU region for compliance

---

### Pattern 5: Hybrid DNS with Route 53 Resolver
```bash
# Create inbound endpoint (on-premises queries AWS)
aws route53resolver create-resolver-endpoint \
  --name corp-inbound \
  --security-group-ids sg-12345 \
  --direction INBOUND \
  --ip-addresses SubnetId=subnet-a,Ip=10.0.1.10 SubnetId=subnet-b,Ip=10.0.2.10

# Create outbound endpoint (AWS queries on-premises)
aws route53resolver create-resolver-endpoint \
  --name corp-outbound \
  --security-group-ids sg-12345 \
  --direction OUTBOUND \
  --ip-addresses SubnetId=subnet-a SubnetId=subnet-b

# Forward corp.internal queries to on-premises DNS
aws route53resolver create-resolver-rule \
  --name forward-to-corp \
  --rule-type FORWARD \
  --domain-name corp.internal \
  --target-ips Ip=192.168.1.10,Port=53 Ip=192.168.1.11,Port=53 \
  --resolver-endpoint-id rslvr-out-abc123

# Associate rule with VPC
aws route53resolver associate-resolver-rule \
  --resolver-rule-id rslvr-rr-123 \
  --vpc-id vpc-abc123
```

**Architecture:**
```
[On-Premises] ←→ [Route 53 Resolver Endpoints] ←→ [VPC Private Zones]
                         (Inbound + Outbound)
```

---

### Pattern 6: Private Hosted Zone with Split-View DNS
```bash
# Public zone (internet-facing)
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)-public

# Add public record
aws route53 change-resource-record-sets \
  --hosted-zone-id Z-PUBLIC \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "internal.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "203.0.113.10"}]
      }
    }]
  }'

# Private zone (VPC-internal)
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference $(date +%s)-private \
  --vpc VPCRegion=us-east-1,VPCId=vpc-12345 \
  --hosted-zone-config PrivateZone=true

# Add private record (same name, different IP)
aws route53 change-resource-record-sets \
  --hosted-zone-id Z-PRIVATE \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "internal.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "10.0.1.50"}]
      }
    }]
  }'
```

**Result:**
- Internet resolves `internal.example.com` → 203.0.113.10 (public IP)
- VPC resolves `internal.example.com` → 10.0.1.50 (private IP)

---

### Pattern 7: DNS Firewall for Security
```bash
# Create firewall rule group
aws route53resolver create-firewall-rule-group \
  --name block-malicious-domains \
  --creator-request-id $(date +%s)

# Add blocking rule
aws route53resolver create-firewall-rule \
  --firewall-rule-group-id rslvr-frg-123 \
  --firewall-domain-list-id rslvr-fdl-456 \
  --priority 100 \
  --action BLOCK \
  --block-response NODATA \
  --name block-malware

# Associate with VPC
aws route53resolver associate-firewall-rule-group \
  --firewall-rule-group-id rslvr-frg-123 \
  --vpc-id vpc-12345 \
  --priority 101 \
  --name prod-vpc-firewall
```

**Use Cases:**
- Block malware C&C domains
- Enforce acceptable use policy
- Data exfiltration prevention

---

## 11. Quick Reference Card

### Routing Policies Comparison
| Policy | Use Case | Health Check | Multi-Value | Failover |
|--------|----------|--------------|-------------|----------|
| **Simple** | Single destination | ❌ | ❌ | ❌ |
| **Weighted** | A/B testing, canary | ✅ | ✅ | Removes unhealthy |
| **Latency** | Global apps | ✅ | ✅ | Routes to healthy+low-latency |
| **Failover** | DR (primary/secondary) | ✅ Required | ❌ | ✅ |
| **Geolocation** | Compliance, localization | ✅ | ✅ | Removes unhealthy |
| **Geoproximity** | Fine-grained geo routing | ✅ | ✅ | Bias-adjusted routing |
| **Multivalue** | Simple load balancing | ✅ | ✅ (up to 8) | Returns healthy only |
| **IP-based** | Route by client IP | ✅ | ✅ | Removes unhealthy |

---

### Alias vs CNAME Decision
```
┌──────────────────────────────────────────────────┐
│ Scenario                    Use                  │
├──────────────────────────────────────────────────┤
│ Zone apex (example.com)     Alias (CNAME illegal)│
│ ALB/CloudFront target       Alias (free queries) │
│ External domain             CNAME                 │
│ S3 website endpoint         Alias                 │
│ Another Route 53 record     Alias                 │
│ Non-AWS service             CNAME                 │
└──────────────────────────────────────────────────┘
```

---

### Health Check Types
| Type | Monitors | Use Case |
|------|----------|----------|
| **Endpoint** | HTTP/HTTPS/TCP probe | Public endpoints |
| **Calculated** | Logical AND/OR of other health checks | Composite health (DB + app) |
| **CloudWatch Alarm** | Alarm state | Internal resources, custom metrics |

---

### TTL Selection Guide
```
┌─────────────────────────────────────────────────┐
│ Scenario              Recommended TTL           │
├─────────────────────────────────────────────────┤
│ Static content        3600-86400s (1-24 hours) │
│ API endpoints         60-300s (1-5 minutes)     │
│ Active migration      30-60s (during cutover)   │
│ Failover config       60s (fast recovery)       │
│ Dev/test             5-30s (rapid iteration)    │
└─────────────────────────────────────────────────┘
```

**Trade-off:** Lower TTL = faster change propagation, higher query volume to Route 53

---

### Resolver Endpoint Architecture
```
[On-Premises Network]
        ↓ (Forward corp.aws.internal)
[Inbound Resolver Endpoint] (10.0.1.10, 10.0.2.10)
        ↓
[Route 53 Private Hosted Zone: corp.aws.internal]
        ↓
[VPC Resources]

[VPC Resources]
        ↓ (Query corp.onprem.local)
[Outbound Resolver Endpoint]
        ↓ (Conditional forwarding rule)
[On-Premises DNS Servers] (192.168.1.10)
```

---

### Cost Breakdown
```
Hosted Zones:
- Public/Private: $0.50/zone/month
- First 25 zones: $0.50 each
- 25+: volume discounts

Queries:
- Alias queries (AWS resources): FREE
- Standard queries: $0.40 per million
- Latency/Geo/Failover: $0.60 per million
- GeoProximity: $0.70 per million

Health Checks:
- AWS endpoint: $0.50/month
- Non-AWS endpoint: $0.75/month (optional)
- HTTPS: +$1.00/month
- String matching: +$1.00/month
- Fast interval (10s): +$1.00/month

Domain Registration:
- .com: $13/year
- .io: $39/year
- .ai: $109/year
```

---

## 12. Exam Tips Summary

### Key Concepts for AWS Exams

1. **Alias Records**
   - Can be used at zone apex (unlike CNAME)
   - Free queries to AWS resources
   - Supports: ALB, CloudFront, S3 website, another Route 53 record, API Gateway
   - Not supported: EC2 instance, on-premises server

2. **Routing Policies**
   - **Simple:** No health checks, multiple IPs returned in random order
   - **Weighted:** Distribute traffic by percentage (not exact due to DNS caching)
   - **Latency:** Routes to lowest-latency region for user
   - **Failover:** Active-passive DR (primary/secondary)
   - **Geolocation:** Match user location (continent, country, state)
   - **Geoproximity:** Distance-based with bias adjustment
   - **Multivalue:** Returns up to 8 healthy records

3. **Health Checks**
   - Endpoint checks: must be publicly accessible (HTTP/HTTPS/TCP)
   - Calculated checks: combine multiple health checks (AND/OR/NOT)
   - CloudWatch alarms: for private resources
   - Default: 30s interval, 3 consecutive failures = unhealthy

4. **Private Hosted Zones**
   - Associated with VPCs
   - Can have same name as public zone (split-view DNS)
   - Supports most routing policies (including failover)
   - DNS records not automatically shared across VPCs

5. **Route 53 Resolver**
   - **Inbound:** On-premises → AWS (queries to private hosted zones)
   - **Outbound:** AWS → on-premises (conditional forwarding rules)
   - Endpoints deployed in VPC subnets (2+ for HA)

6. **TTL Behavior**
   - Cached by resolvers for TTL seconds
   - Lowering TTL doesn't immediately propagate changes
   - Negative caching (NXDOMAIN) also has TTL
   - Alias records use target's TTL

7. **DNSSEC**
   - Route 53 supports DNSSEC signing (public hosted zones)
   - Protects against DNS spoofing/cache poisoning
   - Does NOT encrypt queries (use DNS over HTTPS for that)

8. **Traffic Flow**
   - Visual policy editor for complex routing
   - Version control for routing policies
   - Reusable configurations across hosted zones

9. **Query Logging**
   - Logs DNS queries to CloudWatch Logs
   - Useful for debugging, security, compliance
   - Only available for public hosted zones (2024+: private zones too)

10. **Domain Registration vs Hosted Zones**
    - Registering domain ≠ creating hosted zone
    - NS records at registrar must match hosted zone's name servers
    - Can register domain with Route 53 but use external DNS

---

### Scenario-Based Exam Questions

**Scenario 1:** Need to route example.com (zone apex) to an ALB.
- **Solution:** Use Alias record (A or AAAA type) pointing to ALB
- **Why:** CNAME not allowed at zone apex; Alias solves this

**Scenario 2:** Gradual rollout of new version (10% → 50% → 100%).
- **Solution:** Weighted routing policy with adjustable weights
- **Why:** Allows traffic shifting over time

**Scenario 3:** Route users to nearest region for low latency.
- **Solution:** Latency-based routing policy with health checks
- **Why:** Route 53 measures latency from user to each region

**Scenario 4:** EU users must access EU-hosted data (GDPR).
- **Solution:** Geolocation routing policy (ContinentCode: EU)
- **Why:** Forces EU users to EU endpoints

**Scenario 5:** Failover to DR site when primary region fails.
- **Solution:** Failover routing (primary + secondary with health checks)
- **Why:** Automatic failover when health check fails

**Scenario 6:** Query internal AWS services from on-premises.
- **Solution:** Route 53 Resolver inbound endpoint
- **Why:** Allows on-premises DNS to resolve private hosted zones

**Scenario 7:** Monitor internal application health for DNS failover.
- **Solution:** CloudWatch alarm-based health check
- **Why:** Can't probe private IPs from internet-based health checkers

---

## 13. Interview Q&A

### Q1: Explain how Route 53 Alias records differ from CNAME records
**Answer:**

**CNAME Limitations:**
- Cannot be used at zone apex (example.com)
- Charged for queries
- Adds DNS lookup latency (extra hop)
- Cannot coexist with other record types (NS, MX)

**Alias Benefits:**
- ✅ Works at zone apex
- ✅ Free queries to AWS resources
- ✅ No extra lookup hop
- ✅ Can coexist with other record types
- ✅ Automatic IP updates (ALB IP changes handled transparently)

**Interview Example:**
"To route `example.com` to an ALB, you must use an Alias record because CNAME is illegal at the zone apex. Alias also saves cost (free queries) and reduces latency (no extra DNS hop)."

---

### Q2: Design a multi-region active-active architecture with Route 53
**Answer:**

Users routed to nearest healthy region with automatic failover. Use weighted routing + health checks for 70/30 traffic distribution between us-east-1 and eu-west-1.

**Interview Key Point:** "Weighted routing distributes traffic; health checks provide failover. Combine policies carefully and test all failure modes."

---

### Q3: How does TTL affect DNS failover time?
**Answer:**

**DNS Failover Timeline:**
```
T+0:    Primary fails
T+30s:  Health check detects failure
T+30s:  Route 53 returns secondary
T+30-90s: Resolvers cache expired (TTL=60s)
T+90s:  Most users recovered
```

**Interview Tip:** "Don't claim '30-second failover' just because TTL=30. Account for health detection (30s), cache expiry (TTL), and connection drain. Realistic RTO: 2-5 minutes."

---

### Q4: Explain Route 53 Resolver inbound vs outbound endpoints
**Answer:**

**Inbound:** Allow on-premises to query AWS private hosted zones
**Outbound:** Allow AWS VPCs to query on-premises DNS

**Interview Tip:** "Inbound = queries come IN to AWS; Outbound = queries go OUT from AWS."

---

### Q5: How do you implement split-view DNS in AWS?
**Answer:**

Create separate public and private hosted zones with same name. Public returns internet-facing IPs; private returns internal IPs for VPC clients.

**Interview Tip:** "Private zone wins for associated VPCs. Plan record overlap carefully or queries fail unexpectedly."

---

### Q6: Explain weighted routing and DNS caching effects
**Answer:**

Weighted routing affects DNS responses, not HTTP requests. Resolver caching creates variance from target percentages.

**Observed Distribution:**
- Expected: 70/30
- Reality: 60/40 or 75/25 (depends on resolver distribution)

**Interview Tip:** "Use weighted routing for gradual rollouts, not precise load balancing. For precision, use ALB target groups."

---

### Q7: How do health checks work for private resources?
**Answer:**

Route 53 health checkers CANNOT probe private IPs. Solutions:
1. CloudWatch alarm-based health checks
2. EvaluateTargetHealth (for ALB/NLB)
3. Calculated health checks
4. Public health endpoint proxy

**Interview Tip:** "Private resources need CloudWatch alarm-based health checks or EvaluateTargetHealth."

---

### Q8: Design a blue/green deployment strategy using Route 53
**Answer:**

Use weighted routing with gradual shift: 100% blue → 10% green → 50% green → 100% green. Monitor at each phase. Keep blue warm for fast rollback.

**Interview Key Points:**
- "Gradual shift reduces blast radius"
- "TTL=60s balances recovery speed vs query load"
- "Monitor application metrics, not just DNS queries"

---

### Q9: How does Route 53 handle all health checks failing?
**Answer:**

Route 53 STILL returns unhealthy records (degraded service better than no service).

**Exam Trap:** "When all health checks fail, Route 53 stops responding" → **FALSE**

**Interview Tip:** "Route 53 doesn't blackhole traffic when all checks fail. Design monitoring to alert on 'all unhealthy' state."

---

### Q10: Compare Route 53 with ALB for traffic distribution
**Answer:**

**Route 53:** Multi-region, DNS-level, coarse-grained (~10% variance)
**ALB:** Single-region, request-level, fine-grained (<1% variance)

**Hybrid Approach:** Route 53 chooses region; ALB chooses target within region.

**Interview Tip:** "Route 53 for global distribution, ALB for regional balancing. Use together."

---

**Additional Quick Answers:**

**Route 53 versus ELB?** Route 53 selects DNS answers, often across Regions. ELB routes application connections/requests to targets within its architecture. They are commonly used together.

**Can we use a CNAME at `example.com`?** Not at the zone apex under normal DNS constraints; use a supported Route 53 alias for an eligible target.

**How do private services fail over?** Use supported private-zone routing and a health signal that can represent internal health; public endpoint checkers cannot directly probe a private IP.

**Does latency routing always select the physically nearest Region?** No; selection uses latency information, and resolver/client location behavior matters.

**Does changing DNS ensure zero data loss?** No. Database replication, writer fencing, and application recovery determine data correctness.

---

## 14. Quick Revision

- DNS returns destinations; it does not carry application traffic
- Alias can work at zone apex for supported targets; CNAME cannot
- Health detection + TTL + client behavior determine failover time
- Private zones can support failover, but direct probes need reachable endpoints
- Inbound = external queries into AWS; outbound = AWS queries to external DNS
- Validate the actual delegation and hosted-zone ID
- Weighted routing is approximate due to resolver caching
- When all health checks fail, Route 53 still returns answers

---

## 15. AWS sources

- [Routing policies](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Aliases](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [Health checks](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover-simple-configs.html)
- [Record selection when unhealthy](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-how-route-53-chooses-records.html)
- [Private hosted zones](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zone-private-considerations.html)
- [VPC Resolver](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resolver.html)

**Related:** [ELB](aws-elb-cheatsheet.md), [Aurora](aws-aurora-cheatsheet.md), [VPC networking](aws-vpc-networking-cheatsheet.md).
