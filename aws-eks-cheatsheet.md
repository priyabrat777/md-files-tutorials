# Amazon EKS Cheat Sheet

**Complete Guide to Kubernetes on AWS — workload identity, networking, scaling, upgrades, and operations.**

**Verified:** 2026-10-13. Check current Kubernetes versions, add-on compatibility, and compute-specific limitations before implementation.

[All AWS cheatsheets](aws-core-services-interview-index.md)

---

## Table of Contents

1. [Fundamentals and mental model](#1-fundamentals-and-mental-model)
2. [Compute and responsibility choices](#2-compute-and-responsibility-choices)
3. [Two authorization paths](#3-two-authorization-paths)
4. [Networking and ingress](#4-networking-and-ingress)
5. [Scheduling and scaling](#5-scheduling-and-scaling)
6. [Health, deployments, and state](#6-health-deployments-and-state)
7. [Versions and upgrades](#7-versions-and-upgrades)
8. [Common patterns](#8-common-patterns)
9. [Troubleshooting](#9-troubleshooting)
10. [Quick reference card](#10-quick-reference-card)
11. [Practical inspection](#11-practical-inspection)
12. [Exam tips summary](#12-exam-tips-summary)
13. [Interview questions and model answers](#13-interview-questions-and-model-answers)
14. [AWS sources](#14-aws-sources)

---

## 1. Fundamentals and mental model

EKS supplies a managed Kubernetes control plane. You choose the compute model and own workload configuration, application security, data, and the operating practices not delegated to AWS.

### The full identity-to-service flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       HUMAN / AUTOMATION ACCESS                         │
│                                                                         │
│  IAM Identity (User / Role / CI-CD)                                     │
│       │                                                                 │
│       ▼                                                                 │
│  EKS Access Entry  ──── maps IAM principal → cluster access             │
│       │                                                                 │
│       ▼                                                                 │
│  Kubernetes RBAC  ──── ClusterRole / Role → binding → namespace actions │
│       │                                                                 │
│       ▼                                                                 │
│  Kubernetes API Server  (control plane, AWS-managed)                    │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          WORKLOAD IDENTITY                              │
│                                                                         │
│  Pod                                                                    │
│   ├── Kubernetes Service Account  ←── annotated with IAM role ARN       │
│   │        │                                                            │
│   │        ├── Pod Identity (agent-injected credential)                 │
│   │        │        └── IAM Role → AWS API calls (S3, DDB, SQS …)       │
│   │        └── IRSA (OIDC token + STS AssumeRoleWithWebIdentity)        │
│   │                 └── IAM Role → AWS API calls                        │
│   │                                                                     │
│   └── Node IAM Role  ←── for node infrastructure only, NOT apps        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                        TRAFFIC FLOW                                     │
│                                                                         │
│  Internet → AWS Load Balancer (ALB/NLB)                                 │
│                  │                                                      │
│                  ▼                                                      │
│            Kubernetes Service (ClusterIP / NodePort / LoadBalancer)     │
│                  │                                                      │
│                  ▼                                                      │
│            Pod (selected by label)  ←── readiness probe must pass      │
│                  │                                                      │
│                  ▼                                                      │
│            Container  →  downstream AWS service (via workload role)     │
└─────────────────────────────────────────────────────────────────────────┘
```

### Core Kubernetes resource quick reference

| Kubernetes resource | Job | Common trap |
|---|---|---|
| Pod | Smallest scheduling unit; one or more cooperating containers | Never run a bare Pod in production — use a controller |
| Deployment | Replicated application rollout and desired state | Missing resource requests breaks scheduling |
| StatefulSet | Stable identities and persistent-volume associations | Does not make your DB HA by itself |
| DaemonSet | One Pod per node (agents, log collectors) | Tolerations needed for control-plane nodes |
| Job / CronJob | Run-to-completion or scheduled workloads | Back-off limits and completion tracking matter |
| Service | Stable discovery and traffic abstraction over Pods | Service type controls whether a load balancer is created |
| Ingress | HTTP routing intent interpreted by a controller | Requires a running Ingress controller; AWS LBC is not automatic |
| ConfigMap / Secret | Configuration; a Secret is not secure merely because it is base64 | Mount as env vars or volume; rotate via external secrets operator |
| Namespace | Logical organization and policy boundary | Not a hard tenant boundary; network policy enforces isolation |
| ServiceAccount | Kubernetes identity for workloads | Annotate with IAM role for Pod Identity / IRSA |

---

## 2. Compute and responsibility choices

### Comparison: all four compute modes

| | Managed node groups | Self-managed nodes | Fargate | EKS Auto Mode |
|---|---|---|---|---|
| **Node lifecycle** | AWS handles OS patching and node replacement | You own AMI, patching, replacement | AWS-managed per Pod | AWS-managed, immutable nodes |
| **Node OS access** | SSH possible (not recommended) | Full SSH access | No node access | No SSH access |
| **Scaling** | You configure node group min/max; add Cluster Autoscaler or Karpenter | Same as managed | Unlimited (within quotas) | AWS scales compute automatically |
| **Spot support** | Yes, with interruption handling | Yes | No | Yes (cost optimization built in) |
| **Graviton/Arm** | Yes — choose instance type | Yes | Yes — use `arm64` Fargate profile | Yes — Auto Mode selects Graviton where compatible |
| **Custom AMI** | Supported (bring your own) | Required | No | No — uses AWS-managed immutable AMI |
| **GPU workloads** | Yes | Yes | No | Limited — check current support |
| **DaemonSets** | Supported | Supported | Not supported on Fargate | Supported on managed compute |
| **What you still own** | Upgrade planning, sizing, add-ons, workload resilience, disruption budgets | AMIs, upgrades, lifecycle, security, all of the above | Profile design, scheduling constraints, CPU/memory limits (required), workload compatibility | Workload design, resource requests, disruption settings, IAM and RBAC policy, budgets |

### Managed node groups — key detail

- AWS provisions EC2 instances into an Auto Scaling Group, handles OS patching via optimized AMIs, and drains nodes during updates.
- You still choose the instance type family, set min/max sizes, manage add-on upgrades, and plan Pod Disruption Budgets before node cycling.
- Node group updates use a rolling strategy: new nodes are added, old nodes drained, then terminated.

### EKS Auto Mode — what AWS manages vs what you still own

```
AWS manages in Auto Mode:
├── Compute provisioning (selects instance type, size, AZ)
├── Node OS updates (immutable, replaced rather than patched)
├── Integrated VPC CNI networking
├── Built-in Pod Identity agent (no manual DaemonSet install)
├── EBS / EFS CSI drivers
├── AWS Load Balancer Controller equivalent
├── CoreDNS and kube-proxy equivalents
└── Node group scaling (no Cluster Autoscaler needed)

You still own in Auto Mode:
├── Workload correctness (replicas, probes, anti-affinity)
├── Resource requests and limits (required for bin-packing)
├── Pod Disruption Budgets (voluntary disruption protection)
├── IAM policies and RBAC (access control unchanged)
├── Kubernetes application manifests
├── Application secrets, ConfigMaps, and config strategy
├── Cost visibility (review usage; Auto Mode optimizes, not minimizes)
└── Cluster-level policies and namespaces
```

**Do not design operations around SSH to Auto Mode nodes.** They are replaced, not patched in place.

### Graviton / Arm architecture considerations

- Graviton3 (m7g, c7g, r7g) offers better price/performance for most general workloads.
- Container images must be built for `linux/arm64`. Multi-arch manifests (Docker buildx) let you ship one image tag for both architectures.
- Verify all third-party add-ons and sidecars publish arm64 images before scheduling on Graviton nodes.
- Use `nodeSelector` or node affinity with `kubernetes.io/arch: arm64` to target Graviton explicitly, or let Karpenter/Auto Mode select the best instance.
- Mixed-arch clusters: label node pools and use affinity rules to keep arch-incompatible workloads on `amd64` nodes.

### Fargate — scheduling constraints

- Every Pod that runs on Fargate must match an active Fargate profile (namespace + optional label selectors).
- CPU and memory **must** be specified in the Pod spec; Fargate bins to the next supported vCPU/memory combination.
- DaemonSets, hostNetwork Pods, and privileged containers are not supported.
- Use for serverless, bursty, or security-sensitive workloads where node-level access is undesirable.

---

## 3. Two authorization paths

### Mental model

```
Path A — People and automation accessing Kubernetes
─────────────────────────────────────────────────
  IAM Identity
      │  authenticate
      ▼
  EKS Access Entry  (maps IAM ARN → Kubernetes username/groups)
      │  authorize
      ▼
  Kubernetes RBAC  (ClusterRole / Role → binding → subject)
      │
      ▼
  Kubernetes API action (get pods, deploy, delete, etc.)

Path B — Pods accessing AWS services
─────────────────────────────────────
  Pod
   └── Kubernetes ServiceAccount
         │  annotated with IAM role ARN
         ├── Pod Identity (agent injects credential into Pod environment)
         │        └── STS  →  IAM Role  →  AWS service
         └── IRSA (OIDC token projected into Pod, exchanged at STS)
                  └── STS AssumeRoleWithWebIdentity  →  IAM Role  →  AWS service
```

### Mechanism comparison

| Mechanism | How it works | When to prefer |
|---|---|---|
| EKS access entry | Maps an IAM ARN to a Kubernetes username and groups; replaces the legacy `aws-auth` ConfigMap | Always — access entries are the current recommended approach |
| Kubernetes RBAC | ClusterRole/Role + binding controls who can do what on which Kubernetes resources | Required for namespace-scoped permissions |
| EKS Pod Identity | Agent (DaemonSet or built-in with Auto Mode) injects role credentials into matched Pods | Preferred for new workloads; simpler setup, no OIDC endpoint per cluster |
| IRSA | Cluster OIDC provider + annotated service account + STS token exchange | Existing IRSA setups; required on Fargate |
| Node role | EC2 instance profile for node-level infrastructure (VPC CNI, kubelet, image pull from ECR) | Node infrastructure only — never use it as an app role |

### Access entry — create via CLI

```bash
# Create an access entry mapping an IAM role to Kubernetes RBAC groups
aws eks create-access-entry \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevTeamRole \
  --kubernetes-groups "developers" \
  --type STANDARD

# Optionally attach an EKS access policy (cluster-admin, view, etc.)
aws eks associate-access-policy \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::123456789012:role/DevTeamRole \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=dev-team
```

### Kubernetes RBAC — ClusterRole and binding examples

```yaml
# ClusterRole: read-only view of all namespaces
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: developer-read-only
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps", "events"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets", "statefulsets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["batch"]
    resources: ["jobs", "cronjobs"]
    verbs: ["get", "list", "watch"]
---
# ClusterRoleBinding: grant the ClusterRole to the 'developers' group
# (Group name must match kubernetes-groups in the access entry above)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: developer-read-only-binding
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: developer-read-only
  apiGroup: rbac.authorization.k8s.io
---
# Namespace-scoped Role: deploy access in a single namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: orders
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: orders
subjects:
  - kind: Group
    name: ci-pipeline
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

### Pod Identity — association CLI

```bash
# 1. Create the IAM role (trust policy must allow pods.eks.amazonaws.com)
# See trust policy below, then:

aws eks create-pod-identity-association \
  --cluster-name prod-cluster \
  --namespace orders \
  --service-account order-processor-sa \
  --role-arn arn:aws:iam::123456789012:role/OrderProcessorRole

# The agent automatically injects credentials into matching Pods.
# No annotation needed on the service account for Pod Identity
# (unlike IRSA, which requires the annotation).
```

IAM trust policy for Pod Identity:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "pods.eks.amazonaws.com"
      },
      "Action": [
        "sts:AssumeRole",
        "sts:TagSession"
      ]
    }
  ]
}
```

### IRSA — full setup steps

```bash
# Step 1: Associate an OIDC provider with the cluster (once per cluster)
eksctl utils associate-iam-oidc-provider \
  --cluster prod-cluster \
  --approve

# Or via AWS CLI:
OIDC_URL=$(aws eks describe-cluster --name prod-cluster \
  --query "cluster.identity.oidc.issuer" --output text)

aws iam create-open-id-connect-provider \
  --url "${OIDC_URL}" \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list <thumbprint>

# Step 2: Create an IAM role with OIDC trust policy
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(echo $OIDC_URL | cut -d '/' -f 5)

cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}:sub": "system:serviceaccount:orders:order-processor-sa",
          "oidc.eks.us-east-1.amazonaws.com/id/${OIDC_ID}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

aws iam create-role \
  --role-name OrderProcessorIRSARole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy \
  --role-name OrderProcessorIRSARole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Step 3: Annotate the Kubernetes service account
kubectl annotate serviceaccount order-processor-sa \
  -n orders \
  eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT_ID}:role/OrderProcessorIRSARole
```

### Pod Identity vs IRSA comparison

| | Pod Identity | IRSA |
|---|---|---|
| Setup complexity | Low — one CLI call per namespace/SA | Higher — OIDC provider + per-role trust condition |
| Trust policy target | `pods.eks.amazonaws.com` | OIDC URL + sub claim per service account |
| Credential mechanism | Agent DaemonSet injects credentials | SDK fetches projected service account token, calls STS |
| Fargate support | No | Yes |
| Cross-account | Yes (trust policy in target account) | Yes |
| Session tags | Yes (`sts:TagSession`) | Limited |
| Prefer for new clusters | Yes | Existing workloads or Fargate requirement |

### The node role pitfall — why using it for apps is wrong

The node IAM role is attached to the EC2 instances (or used by managed compute) so the Kubernetes node itself can:
- Pull images from ECR
- Call the VPC CNI (EC2 network APIs)
- Write node-level metrics and logs

Every Pod running on that node can reach the instance metadata service (IMDS) at `169.254.169.254`. If you add application permissions to the node role, **every Pod on every node in that group inherits those permissions** — even unrelated workloads and compromised containers.

```
Wrong approach:
  Node role → s3:GetObject on my-data-bucket
  Result: ALL Pods on ALL nodes can read the bucket

Correct approach:
  Node role → only VPC CNI + ECR + CloudWatch (minimal)
  Order processor ServiceAccount → Pod Identity → s3:GetObject on my-data-bucket
  Result: Only order-processor Pods get S3 access
```

Block IMDS access for Pods that do not need node credentials:
```yaml
# In the Pod spec, set hostNetwork: false and consider blocking IMDS
# via a network policy or IMDSv2 hop-limit of 1 (so Pods can't reach it)
```

---

## 4. Networking and ingress

### VPC CNI overview

The default CNI assigns real VPC IP addresses directly to Pods. Each node gets one or more ENIs, and each ENI can hold multiple IP addresses.

```
Node (m5.large example — 3 ENIs × 10 IPs = 30 IPs)
├── ENI 0 (primary): node IP + up to 9 Pod IPs
├── ENI 1 (secondary): up to 10 Pod IPs
└── ENI 2 (secondary): up to 10 Pod IPs
    Total capacity: ~29 Pods (1 IP reserved per ENI)

Problem: large clusters exhaust subnet IP space quickly.
Solution: prefix delegation (below).
```

### VPC CNI prefix delegation

Without prefix delegation, each secondary IP slot holds one Pod IP. With prefix delegation, each slot holds a `/28` prefix (16 IPs), dramatically increasing Pod density per node.

```bash
# Enable prefix delegation (set ENABLE_PREFIX_DELEGATION on the aws-node DaemonSet)
kubectl set env daemonset aws-node -n kube-system ENABLE_PREFIX_DELEGATION=true

# With prefix delegation, an m5.large can host ~110 Pods instead of ~29
# /28 prefix = 16 IPs per slot; 2 ENIs × 9 slots × 16 = 288 IPs (minus overhead)
```

Key considerations:
- Subnets need sufficient IP space for prefixes (plan for `/28` blocks).
- Each prefix counts as one slot in the ENI limit table.
- Recommended for clusters with many small Pods (microservices, batch).

### Security groups for Pods

By default, all Pods on a node share the node's security group. Security groups for Pods assigns an **individual security group directly to a Pod**, enabling fine-grained network access control.

```
Without SG for Pods:
  All Pods on node → node SG → same firewall rules

With SG for Pods:
  payment-service Pod → payment-sg → allow RDS port 5432 from payment-sg only
  frontend Pod        → frontend-sg → allow 443 from ALB SG only
```

Requirements:
- Trunk ENI support (not all instance types; check compatibility matrix).
- `SecurityGroupPolicy` CRD installed (part of VPC CNI).
- Node group must use nitro-based instances.

```yaml
apiVersion: vpcresources.k8s.aws/v1beta1
kind: SecurityGroupPolicy
metadata:
  name: payment-sg-policy
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: payment-service
  securityGroups:
    groupIds:
      - sg-0a1b2c3d4e5f67890   # your payment security group
```

### Kubernetes network policy

Kubernetes `NetworkPolicy` resources are enforced by the CNI. The VPC CNI supports network policies natively (since VPC CNI v1.14+); earlier clusters may need Calico or Cilium.

```yaml
# Allow the payment service to call the orders DB on port 5432 only
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-payment-to-db
  namespace: payments
spec:
  podSelector:
    matchLabels:
      app: orders-db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: payment-service
      ports:
        - protocol: TCP
          port: 5432
```

### AWS Load Balancer Controller vs in-tree controller

| | AWS Load Balancer Controller (LBC) | In-tree controller (legacy) |
|---|---|---|
| **Manages** | ALB (Ingress), NLB (Service type LoadBalancer) | Classic ELB (CLB) and some NLB via annotations |
| **Deployment** | Add-on / Helm chart you install | Built into Kubernetes controller-manager |
| **ALB support** | Full (path-based routing, WAF, Cognito auth) | No ALB support |
| **NLB target modes** | Instance or IP target mode | Instance mode only |
| **Health checks** | Fine-grained annotation control | Basic |
| **Auto Mode** | Built in — no separate install | N/A |
| **Recommendation** | Use LBC for all new clusters | Only on very old clusters |

Installing AWS LBC (non-Auto Mode clusters):
```bash
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=prod-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

### Ingress with ALB — minimal example

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-service
                port:
                  number: 80
```

### Private cluster connectivity requirements

A private cluster has no public endpoint for the Kubernetes API server. Nodes and workloads cannot reach the internet by default.

Required VPC endpoints (at minimum):
```
aws eks                  — EKS cluster API
com.amazonaws.*.ec2      — VPC CNI node management
com.amazonaws.*.ecr.api  — ECR image manifest
com.amazonaws.*.ecr.dkr  — ECR image layers
com.amazonaws.*.s3       — ECR layer storage (Gateway endpoint)
com.amazonaws.*.sts      — IRSA / Pod Identity credential exchange
com.amazonaws.*.logs     — CloudWatch logging
com.amazonaws.*.ssm      — (optional) SSM Session Manager for node access
```

Kubectl access to a private endpoint requires being in the VPC or connected via VPN/Direct Connect. CI/CD systems must also be VPC-resident or use a bastion/jump host.

---

## 5. Scheduling and scaling

### Resource requests vs limits — the real difference

```
Requests = what the scheduler uses to place the Pod
Limits   = what the container runtime enforces at runtime

CPU:
  Requests → guaranteed CPU time; scheduler places Pod on a node with available CPU
  Limits   → enforced via Linux CFS quotas; CPU is THROTTLED when limit is exceeded
             (the container is NOT killed; it just runs slower)

Memory:
  Requests → minimum memory guaranteed
  Limits   → if the container exceeds its memory limit, the kernel sends SIGKILL
             (OOMKilled) — the container RESTARTS, potentially causing disruption

Missing requests:
  → Scheduler has no data; may pack too many Pods on one node
  → Node can become overloaded and start OOM-killing Pods

Oversized requests:
  → Scheduler "reserves" capacity that is never used
  → Nodes appear full while actual CPU is idle (wasted money)
```

Best practice table:

| | CPU | Memory |
|---|---|---|
| **Always set** | requests | requests + limits |
| **Optional but recommended** | limits (watch for throttle) | — |
| **Effect of exceeding** | Throttle (CFS quota) | OOMKill (container restart) |
| **VPA helps with** | Right-sizing requests | Right-sizing requests |

### Horizontal Pod Autoscaler (HPA) example

```yaml
# Requires Metrics Server installed in the cluster
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # target 60% of CPU request across all Pods
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
```

HPA requires:
- Metrics Server (or KEDA for custom/external metrics)
- Pod resource requests set (HPA uses % of request, not absolute)
- Deployment (or other scalable target) — not StatefulSet without careful thought

### Karpenter — NodePool and NodeClaim concepts

Karpenter replaces the Cluster Autoscaler. Instead of scaling pre-defined node groups, it watches for unschedulable Pods and provisions exactly the right instance type in seconds.

```
Cluster Autoscaler:
  Unschedulable Pod → scale up existing node group → wait 2–5 min

Karpenter:
  Unschedulable Pod → evaluate constraints → pick optimal instance → launch in ~30s
```

Key resources:
- **NodePool**: defines a set of constraints (instance families, availability zones, OS, taints, labels) and expiry/disruption policies.
- **NodeClaim**: the representation of a single provisioned node; Karpenter creates and manages these automatically.
- **EC2NodeClass**: AWS-specific configuration (AMI selector, subnet selector, security group selector, IAM role).

```yaml
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: general-purpose
spec:
  template:
    metadata:
      labels:
        workload-type: general
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand", "spot"]
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64", "arm64"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["5"]
      taints: []
  limits:
    cpu: "1000"
    memory: 2000Gi
  disruption:
    consolidationPolicy: WhenUnderutilized
    consolidateAfter: 1m
---
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2023
  role: KarpenterNodeRole-prod-cluster
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: prod-cluster
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: prod-cluster
```

### Pod Disruption Budget (PDB) — example and limits

```yaml
# Ensure at least 2 replicas of the payment service are always available
# during voluntary disruptions (node drain, rolling upgrade, etc.)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: payment-pdb
  namespace: payments
spec:
  minAvailable: 2       # alternatively use maxUnavailable: 1
  selector:
    matchLabels:
      app: payment-service
```

What PDB protects against (voluntary disruptions):
- Node drains during upgrades
- Karpenter consolidation
- kubectl drain

What PDB does NOT protect against (involuntary disruptions):
- Node hardware failure
- AZ outage
- OOMKilled containers
- Kubernetes bug crashing the Pod

```
Mental model:
  PDB = "politely refuse voluntary evictions that would break availability"
  PDB ≠ "protect against the world ending"

For AZ resilience: use replicas ≥ 3 with topology spread constraints across AZs.
```

### Topology spread — distribute across AZs

```yaml
# In the Deployment spec.template.spec:
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: webapp
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: webapp
```

### Scaling mechanisms comparison

| Mechanism | Scales | Trigger | Common trap |
|---|---|---|---|
| HPA | Pod replica count | CPU / memory / custom metrics | Requires resource requests; scale-down delay needed |
| VPA | Pod resource requests/limits | Historical usage | Pod restart required to apply new requests |
| Cluster Autoscaler | Node group size | Unschedulable Pods | Cannot satisfy impossible constraints; slow (2–5 min) |
| Karpenter | Individual nodes (any type) | Unschedulable Pods or consolidation | NodePool constraints must be broad enough to match Pod needs |
| Auto Mode compute | Nodes (AWS-managed) | Unschedulable Pods | Disruption budgets still govern when nodes are replaced |

---

## 6. Health, deployments, and state

- **Readiness probe:** can this Pod accept traffic? Failing readiness removes the Pod from Service endpoints — traffic stops flowing to it.
- **Liveness probe:** should the container be restarted? A failed liveness probe triggers a container restart, not a new Pod.
- **Startup probe:** allow slow initialization before liveness/readiness probes start checking. Prevents restart loops for slow-starting JVMs or migration containers.

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 2

startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30    # allow up to 30 × 10s = 5 min to start
  periodSeconds: 10
```

- Coordinate `terminationGracePeriodSeconds`, `preStop` hooks, and load-balancer deregistration delay to achieve zero-downtime rollouts.
- Use rolling-update `maxSurge` and `maxUnavailable` with enough spare node capacity.
- Validate schema migrations are backward-compatible before rolling an application update.
- EBS volumes are AZ-scoped. A Pod bound to an EBS PVC cannot schedule in a different AZ. Use EFS for cross-AZ shared storage.

A StatefulSet manages stable network identities and persistent volume claims per Pod ordinal. It does not make your database HA, replicated, or safe to fail over — that is the application's responsibility.

---

## 7. Versions and upgrades

EKS supports **14 months of standard support plus 12 months of extended support** (additional charges apply). Check the [live lifecycle table](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html) — do not hard-code version numbers in long-lived notes.

### Upgrade sequence

```
Step 1: Upgrade the control plane
  aws eks update-cluster-version --name prod-cluster --kubernetes-version 1.32

Step 2: Update add-ons (must be compatible with new control plane version)
  aws eks update-addon --cluster-name prod-cluster --addon-name vpc-cni --addon-version v1.18.x
  aws eks update-addon --cluster-name prod-cluster --addon-name coredns --addon-version v1.11.x
  aws eks update-addon --cluster-name prod-cluster --addon-name kube-proxy --addon-version v1.32.x

Step 3: Upgrade managed node groups
  aws eks update-nodegroup-version --cluster-name prod-cluster --nodegroup-name general

Step 4: Validate workloads
  kubectl get pods -A    # check all namespaces
  Run application smoke tests

Step 5: Update Karpenter / Cluster Autoscaler (if used)
Step 6: Rotate self-managed or Fargate nodes
```

Before upgrading:
- Inspect deprecated Kubernetes API versions (`kubectl convert`, `pluto` tool).
- Check add-on compatibility matrix in the EKS console.
- Review admission webhooks (they may reject requests to new API versions).
- Test in a non-production cluster first.
- Ensure PDBs allow node drains during the upgrade.

---

## 8. Common patterns

### Mental Model: IAM → EKS Access → RBAC Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    HUMAN/AUTOMATION → KUBERNETES API                     │
└──────────────────────────────────────────────────────────────────────────┘

Step 1: AWS Authentication
┌─────────────────────────┐
│ IAM Identity            │
│  • IAM User             │──── aws sts get-caller-identity
│  • IAM Role (assumed)   │     returns: arn:aws:iam::123456789012:role/DevRole
│  • Federated (SSO)      │
└─────────────────────────┘
           │
           ▼
Step 2: EKS Access Entry Mapping
┌─────────────────────────────────────────────────────────────────┐
│ EKS Access Entry                                                │
│   Principal ARN: arn:aws:iam::123456789012:role/DevRole         │
│   Type: STANDARD                                                │
│   Kubernetes Groups: ["developers", "viewers"]                  │
│   └─── Maps IAM identity → Kubernetes username + groups         │
└─────────────────────────────────────────────────────────────────┘
           │
           ▼
Step 3: Kubernetes RBAC Evaluation
┌────────────────────────────────────────────────────────────────┐
│ ClusterRole / Role                                             │
│   name: developer-access                                       │
│   rules:                                                       │
│     - apiGroups: ["apps"]                                      │
│       resources: ["deployments"]                               │
│       verbs: ["get", "list", "create", "update"]               │
└────────────────────────────────────────────────────────────────┘
           │
           ▼
┌────────────────────────────────────────────────────────────────┐
│ ClusterRoleBinding / RoleBinding                               │
│   subjects:                                                    │
│     - kind: Group                                              │
│       name: developers  ←── matches access entry group         │
│   roleRef:                                                     │
│     kind: ClusterRole                                          │
│     name: developer-access                                     │
└────────────────────────────────────────────────────────────────┘
           │
           ▼
Step 4: API Action Authorized
┌────────────────────────────────────────────────────────────────┐
│ kubectl apply -f deployment.yaml                               │
│ ✓ ALLOWED                                                      │
└────────────────────────────────────────────────────────────────┘
```

### Mental Model: Pod Identity Credential Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                      POD → AWS SERVICE ACCESS                            │
└──────────────────────────────────────────────────────────────────────────┘

Setup Phase (one-time configuration):
┌──────────────────────────┐
│ 1. Create IAM Role       │
│    Trust Policy:         │
│    Principal:            │
│      pods.eks.amazonaws.com
│    Permission Policy:    │
│      S3, DynamoDB, etc.  │
└──────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│ 2. Create ServiceAccount in Kubernetes                   │
│    apiVersion: v1                                        │
│    kind: ServiceAccount                                  │
│    metadata:                                             │
│      name: app-sa                                        │
│      namespace: prod                                     │
└──────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│ 3. Create Pod Identity Association                       │
│    aws eks create-pod-identity-association \             │
│      --cluster prod-cluster \                            │
│      --namespace prod \                                  │
│      --service-account app-sa \                          │
│      --role-arn arn:aws:iam::123:role/AppRole            │
└──────────────────────────────────────────────────────────┘

Runtime Flow (automatic, per request):
┌──────────────────────────────────────────────────────────┐
│ Pod with serviceAccountName: app-sa                      │
│   Environment injected by agent:                         │
│   AWS_ROLE_ARN=arn:aws:iam::123:role/AppRole             │
│   AWS_CONTAINER_CREDENTIALS_FULL_URI=http://...          │
│   AWS_CONTAINER_AUTHORIZATION_TOKEN=<token>              │
└──────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│ Application Code (AWS SDK)                               │
│   s3_client = boto3.client('s3')                         │
│   # SDK automatically discovers credentials from env     │
└──────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│ Pod Identity Agent (DaemonSet on node)                   │
│   • Intercepts credential request                        │
│   • Validates Pod's ServiceAccount                       │
│   • Matches association: namespace + SA → IAM Role       │
│   • Calls STS AssumeRole on behalf of Pod                │
│   • Returns temporary credentials (15 min – 12 hr)       │
└──────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────────┐
│ AWS Service (S3, DynamoDB, SQS, etc.)                    │
│   • Receives request with temporary credentials          │
│   • Validates IAM role permissions                       │
│   • Executes action if allowed                           │
└──────────────────────────────────────────────────────────┘

Comparison: Pod Identity vs IRSA
┌────────────────────────┬────────────────────┬──────────────────────┐
│                        │   Pod Identity     │       IRSA           │
├────────────────────────┼────────────────────┼──────────────────────┤
│ Trust Policy Principal │ pods.eks.aws.com   │ OIDC provider URL    │
│ Association Location   │ EKS API            │ IAM trust condition  │
│ ServiceAccount Annot.  │ Not needed         │ Required             │
│ Fargate Support        │ No                 │ Yes                  │
│ Setup Complexity       │ Low                │ Medium-High          │
│ Preferred for New      │ Yes                │ Legacy/Fargate only  │
└────────────────────────┴────────────────────┴──────────────────────┘
```

### Pattern A: Deploying Stateless App with HPA + Cluster Autoscaler

**Goal:** Elastically scale a web application from 2 to 50 replicas based on CPU load, with nodes provisioning automatically.

**Architecture:**
```
Internet → Route 53 → ALB (managed by AWS LBC)
                       ↓
               Kubernetes Service (ClusterIP)
                       ↓
           Deployment (2-50 replicas with HPA)
                       ↓
                    Pods (on auto-scaled nodes)
```

**Complete Implementation:**

```yaml
# 1. Deployment with resource requests (required for HPA)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: prod
  labels:
    app: webapp
spec:
  replicas: 2   # HPA will override this
  selector:
    matchLabels:
      app: webapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 0   # zero-downtime rollouts
  template:
    metadata:
      labels:
        app: webapp
        version: v1.2.0
    spec:
      serviceAccountName: webapp-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: webapp
      containers:
        - name: webapp
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/webapp:v1.2.0
          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
          env:
            - name: PORT
              value: "8080"
            - name: LOG_LEVEL
              value: "info"
          resources:
            requests:
              cpu: "250m"      # HPA uses this as baseline
              memory: "256Mi"
            limits:
              memory: "512Mi"  # prevent OOM on node
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 10
            periodSeconds: 10
            timeoutSeconds: 2
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
            timeoutSeconds: 2
            failureThreshold: 2
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]  # allow load balancer deregistration
      terminationGracePeriodSeconds: 30
---
# 2. Service (ClusterIP for ALB target-type: ip)
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: prod
spec:
  type: ClusterIP
  selector:
    app: webapp
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: http
---
# 3. Ingress with ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: prod
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc123
    alb.ingress.kubernetes.io/healthcheck-path: /ready
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "15"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "5"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
    alb.ingress.kubernetes.io/success-codes: "200"
    alb.ingress.kubernetes.io/load-balancer-attributes: idle_timeout.timeout_seconds=60
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-service
                port:
                  number: 80
---
# 4. HorizontalPodAutoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60   # scale up when average CPU > 60% of request
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15   # double pods every 15 seconds
        - type: Pods
          value: 4
          periodSeconds: 15   # or add 4 pods, whichever is higher
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300   # wait 5 min before scaling down
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60   # remove 50% of pods per minute max
        - type: Pods
          value: 2
          periodSeconds: 60
      selectPolicy: Min
---
# 5. PodDisruptionBudget (protect during node drains)
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: webapp-pdb
  namespace: prod
spec:
  minAvailable: 1   # always keep at least 1 replica during voluntary disruptions
  selector:
    matchLabels:
      app: webapp
```

**Cluster Autoscaler Configuration:**
```bash
# Install via Helm
helm repo add autoscaler https://kubernetes.github.io/autoscaler
helm install cluster-autoscaler autoscaler/cluster-autoscaler \
  --namespace kube-system \
  --set autoDiscovery.clusterName=prod-cluster \
  --set awsRegion=us-east-1 \
  --set rbac.create=true
```

**Traffic Flow:**
1. User hits `https://app.example.com`
2. Route 53 resolves to ALB
3. ALB sends traffic to healthy Pod IPs (readiness probe passing)
4. CPU rises → HPA adds Pods
5. Pods pending (no capacity) → Cluster Autoscaler adds nodes
6. Pods schedule on new nodes
7. Load drops → HPA removes Pods (after stabilization window)
8. Nodes underutilized → Cluster Autoscaler drains and terminates (respects PDB)

---

### Pattern B: StatefulSet with EBS for Database Workload

**Goal:** Run a 3-node PostgreSQL cluster with persistent storage, stable network identities, and ordered deployment.

**Key Differences from Deployment:**
- Each Pod gets a stable hostname: `postgres-0.postgres-service.prod.svc.cluster.local`
- Each Pod gets its own PersistentVolumeClaim (not shared)
- EBS volumes are AZ-scoped — Pods cannot move to different AZs
- Startup/shutdown is ordered: 0 → 1 → 2 (scale up), 2 → 1 → 0 (scale down)

```yaml
# 1. Headless Service (for stable DNS)
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: prod
  labels:
    app: postgres
spec:
  clusterIP: None   # headless service
  selector:
    app: postgres
  ports:
    - name: postgres
      port: 5432
      targetPort: 5432
---
# 2. StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: prod
spec:
  serviceName: postgres-service   # must match headless service
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  podManagementPolicy: OrderedReady   # wait for each Pod to be Ready before starting next
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: postgres
    spec:
      serviceAccountName: postgres-sa
      securityContext:
        fsGroup: 999   # postgres user
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: postgres
              topologyKey: kubernetes.io/hostname   # never co-locate on same node
      containers:
        - name: postgres
          image: postgres:15-alpine
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              memory: "2Gi"
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
          livenessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U postgres
            initialDelaySeconds: 30
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3
          readinessProbe:
            exec:
              command:
                - /bin/sh
                - -c
                - pg_isready -U postgres
            initialDelaySeconds: 10
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 2
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes: ["ReadWriteOnce"]   # EBS supports RWO only
        storageClassName: gp3-encrypted   # use encrypted gp3 EBS
        resources:
          requests:
            storage: 100Gi
---
# 3. StorageClass (encrypted gp3 EBS)
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012
volumeBindingMode: WaitForFirstConsumer   # delay provisioning until Pod is scheduled
allowVolumeExpansion: true
reclaimPolicy: Retain   # keep data if StatefulSet is deleted
---
# 4. PodDisruptionBudget
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: postgres-pdb
  namespace: prod
spec:
  minAvailable: 2   # always keep at least 2/3 replicas during drains
  selector:
    matchLabels:
      app: postgres
```

**Critical StatefulSet Considerations:**
- **AZ Failure:** If `postgres-1` is in `us-east-1a` and that AZ fails, the Pod will NOT reschedule to `us-east-1b` because the EBS volume is AZ-locked. You must use EFS (cross-AZ) or application-level replication across AZs.
- **Ordered Startup:** Ensure Pod 0 initializes the cluster before Pods 1 and 2 join.
- **Volume Deletion:** `reclaimPolicy: Retain` keeps PVCs after StatefulSet deletion. Delete manually or use `Delete` policy with caution.
- **Not HA by Default:** StatefulSet provides stable identity, not automatic failover. Application-level replication (Patroni, Stolon) is required for HA.

---

### Pattern C: Cross-Account EKS Access

**Goal:** Developers in AWS Account 111111111111 need `kubectl` access to an EKS cluster in Account 222222222222.

**Architecture:**
```
Account 111111111111 (Dev Account)          Account 222222222222 (Prod Account)
┌─────────────────────────┐                 ┌─────────────────────────┐
│ IAM User: alice         │                 │ EKS Cluster: prod       │
│ Assumes Role:           │                 │ Access Entry:           │
│   EKS-Admin-Role        │────────────────▶│   Principal:            │
│   (in Account 222...)   │   AssumeRole    │     EKS-Admin-Role      │
└─────────────────────────┘                 │   Groups: ["admins"]    │
                                            │                         │
                                            │ RBAC: ClusterRoleBinding│
                                            │   admins → cluster-admin│
                                            └─────────────────────────┘
```

**Step 1: Create IAM Role in Account 222222222222 (Prod)**

Trust policy (allows Account 111111111111 to assume):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-secret-12345"
        }
      }
    }
  ]
}
```

```bash
# In Account 222222222222
aws iam create-role \
  --role-name EKS-Admin-Role \
  --assume-role-policy-document file://trust-policy.json

# Attach EKS describe permissions
aws iam attach-role-policy \
  --role-name EKS-Admin-Role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

**Step 2: Create EKS Access Entry in Account 222222222222**

```bash
aws eks create-access-entry \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::222222222222:role/EKS-Admin-Role \
  --kubernetes-groups "admins" \
  --type STANDARD

# Or grant managed policy (alternative to custom RBAC)
aws eks associate-access-policy \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::222222222222:role/EKS-Admin-Role \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy \
  --access-scope type=cluster
```

**Step 3: Create RBAC in Cluster (Account 222222222222)**

```yaml
# ClusterRoleBinding (admins group → cluster-admin)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admins-cluster-admin
subjects:
  - kind: Group
    name: admins   # matches access entry kubernetes-groups
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-admin   # built-in Kubernetes role
  apiGroup: rbac.authorization.k8s.io
```

**Step 4: Grant AssumeRole Permission in Account 111111111111 (Dev)**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::222222222222:role/EKS-Admin-Role",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-secret-12345"
        }
      }
    }
  ]
}
```

Attach this policy to IAM users/groups in Account 111111111111.

**Step 5: Configure kubectl in Dev Account**

```bash
# Assume cross-account role
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/EKS-Admin-Role \
  --role-session-name alice-session \
  --external-id unique-secret-12345

# Export credentials (or use --profile)
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_SESSION_TOKEN=...

# Update kubeconfig
aws eks update-kubeconfig \
  --name prod-cluster \
  --region us-east-1 \
  --role-arn arn:aws:iam::222222222222:role/EKS-Admin-Role

# Verify
kubectl get nodes
```

**Alternative: aws-vault or AWS SSO**
```bash
# Using aws-vault
aws-vault exec prod-admin -- kubectl get pods -A

# Using AWS SSO profile
aws eks update-kubeconfig --name prod-cluster --profile prod-admin
kubectl get nodes
```

---

### Pattern D: GitHub Actions OIDC to EKS (No Long-Term Credentials)

**Goal:** GitHub Actions workflow deploys to EKS without storing AWS access keys in GitHub Secrets.

**Architecture:**
```
GitHub Actions Workflow
        │
        │ OIDC Token (claims: repo, branch, actor)
        ▼
AWS IAM OIDC Provider (token.actions.githubusercontent.com)
        │
        │ AssumeRoleWithWebIdentity
        ▼
IAM Role (trusts GitHub OIDC + repo condition)
        │
        │ Temporary Credentials
        ▼
kubectl apply (via EKS access entry + RBAC)
```

**Step 1: Create IAM OIDC Provider for GitHub Actions**

```bash
# In AWS Account
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**Step 2: Create IAM Role with GitHub OIDC Trust Policy**

Trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:myorg/myapp:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Explanation:
- `sub` claim restricts to specific repo and branch
- Change to `"repo:myorg/myapp:*"` to allow all branches
- Change to `"repo:myorg/*"` to allow all repos in org (risky)

```bash
aws iam create-role \
  --role-name GitHubActionsEKSDeployRole \
  --assume-role-policy-document file://github-trust-policy.json

# Attach EKS permissions
aws iam attach-role-policy \
  --role-name GitHubActionsEKSDeployRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

**Step 3: Create EKS Access Entry**

```bash
aws eks create-access-entry \
  --cluster-name prod-cluster \
  --principal-arn arn:aws:iam::123456789012:role/GitHubActionsEKSDeployRole \
  --kubernetes-groups "ci-deployers" \
  --type STANDARD
```

**Step 4: Create Kubernetes RBAC**

```yaml
# Role: deploy permissions in prod namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: prod
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["pods/log"]
    verbs: ["get", "list"]
---
# RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: deployer-binding
  namespace: prod
subjects:
  - kind: Group
    name: ci-deployers   # matches access entry group
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployer
  apiGroup: rbac.authorization.k8s.io
```

**Step 5: GitHub Actions Workflow**

```yaml
# .github/workflows/deploy.yml
name: Deploy to EKS
on:
  push:
    branches: [main]

permissions:
  id-token: write   # required for OIDC token
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsEKSDeployRole
          role-session-name: github-actions-deploy
          aws-region: us-east-1

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --name prod-cluster \
            --region us-east-1

      - name: Verify access
        run: kubectl get nodes

      - name: Deploy application
        run: |
          kubectl set image deployment/webapp \
            webapp=123456789012.dkr.ecr.us-east-1.amazonaws.com/webapp:${{ github.sha }} \
            -n prod
          kubectl rollout status deployment/webapp -n prod

      - name: Verify deployment
        run: kubectl get pods -n prod -l app=webapp
```

**Security Best Practices:**
- Restrict `sub` claim to specific repo + branch
- Use separate roles for staging vs production
- Grant minimal RBAC permissions (namespace-scoped Role, not ClusterRole)
- Use ECR image scanning and IAM policy to prevent unapproved images
- Audit CloudTrail for `AssumeRoleWithWebIdentity` events

---

## 9. Troubleshooting

### Symptom → root cause map

| Symptom | Most common root causes | First commands |
|---|---|---|
| Pod `Pending` — insufficient CPU/memory | Node group at max size; requests are too large; VPA recommendations not applied | `kubectl describe pod <name>` → Events section; `kubectl top nodes` |
| Pod `Pending` — unschedulable (taint/affinity) | Node taints not tolerated; affinity/anti-affinity conflict; nodeSelector references label that doesn't exist | `kubectl describe pod` → Events; `kubectl get nodes --show-labels` |
| Pod `Pending` — no IPs available | Subnet exhausted; prefix delegation not enabled; ENI limit reached for instance type | `kubectl describe node` → Capacity/Allocatable; check VPC subnet free IPs |
| `CrashLoopBackOff` — application exit | Bug in app; missing env var / secret; wrong command/entrypoint; readiness probe kills before app ready | `kubectl logs <pod> --previous`; check exit code in `kubectl describe pod` |
| `CrashLoopBackOff` — probe failure | Liveness probe too aggressive (wrong path/port/timing); startup probe missing for slow app | `kubectl describe pod` → Last State exit code; review probe config |
| `ImagePullBackOff` — wrong name/tag | Image tag does not exist in registry | `kubectl describe pod` → Events; try `docker pull` from same network |
| `ImagePullBackOff` — ECR auth | Node role missing `ecr:GetAuthorizationToken` / `ecr:BatchGetImage`; image in different account | Check node IAM role policies; confirm image ARN account matches |
| `ImagePullBackOff` — arch mismatch | Image is amd64-only but node is arm64 (Graviton) | Check image manifest: `docker manifest inspect <image>`; build multi-arch image |
| `OOMKilled` | Memory limit too low; memory leak; JVM heap not bounded; container flags (`-Xmx`) exceed limit | `kubectl describe pod` → OOMKilled exit code (137); check container memory limit vs JVM args |
| Service unreachable — selector mismatch | Service `spec.selector` labels don't match Pod labels | `kubectl get endpoints <service>` — empty means no matching Pods; `kubectl describe svc` |
| Service unreachable — readiness failing | Pods in service but not passing readiness; probe path/port wrong | `kubectl describe pod` → readiness probe events; check `/ready` endpoint manually |
| Service unreachable — network policy | NetworkPolicy blocks traffic; default-deny installed but allowlist incomplete | `kubectl get networkpolicy -A`; test with `kubectl exec` + `curl` from source Pod |
| `AccessDenied` calling AWS API from Pod | Pod Identity association missing; IRSA annotation wrong; wrong namespace/SA name in association | `kubectl exec -it <pod> -- env | grep AWS`; `aws eks list-pod-identity-associations`; check role trust policy |
| `kubectl` unauthorized | IAM caller not in access entry; wrong kubeconfig context; role ARN has path mismatch | `aws sts get-caller-identity`; `aws eks list-access-entries`; check kubeconfig with `kubectl config view` |
| ALB returns 503 | All target Pods failing health check; target group deregistration in progress; readiness probe failing | Check ALB target group health in AWS console; `kubectl get endpoints` in Ingress namespace |
| Nodes NotReady | Node out of disk/memory (node pressure); kubelet crash; network connectivity issue | `kubectl describe node <name>` → Conditions; `kubectl get events`; check node system logs via SSM/CloudWatch |

### Diagnostic command cheat sheet

```bash
# Pod-level investigation
kubectl describe pod <pod-name> -n <namespace>        # events, conditions, probe config
kubectl logs <pod-name> -n <namespace>                 # current logs
kubectl logs <pod-name> -n <namespace> --previous      # logs from last crashed container
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh  # interactive shell

# Workload state
kubectl get pods -A -o wide                            # all Pods with node placement
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl top pods -n <namespace>                        # CPU/memory usage (requires Metrics Server)
kubectl top nodes

# Service and networking
kubectl get endpoints -n <namespace>                   # check Pod IPs behind a Service
kubectl describe svc <service-name> -n <namespace>

# EKS/IAM
aws sts get-caller-identity                            # confirm who kubectl is authenticated as
aws eks list-access-entries --cluster-name <cluster>
aws eks list-pod-identity-associations --cluster-name <cluster>
```

---

## 10. Quick reference card

### Key Kubernetes resource cheat table

| Resource | API group | Key fields | kubectl shortname |
|---|---|---|---|
| Pod | core (v1) | containers, volumes, serviceAccountName | po |
| Deployment | apps/v1 | replicas, selector, template, strategy | deploy |
| StatefulSet | apps/v1 | replicas, volumeClaimTemplates, serviceName | sts |
| DaemonSet | apps/v1 | selector, template, tolerations | ds |
| Service | core (v1) | selector, ports, type | svc |
| Ingress | networking.k8s.io/v1 | rules, tls, ingressClassName | ing |
| ConfigMap | core (v1) | data, binaryData | cm |
| Secret | core (v1) | data (base64), type | (none) |
| ServiceAccount | core (v1) | automountServiceAccountToken | sa |
| HorizontalPodAutoscaler | autoscaling/v2 | scaleTargetRef, minReplicas, maxReplicas, metrics | hpa |
| PodDisruptionBudget | policy/v1 | minAvailable / maxUnavailable, selector | pdb |
| NetworkPolicy | networking.k8s.io/v1 | podSelector, ingress, egress | netpol |
| PersistentVolumeClaim | core (v1) | storageClassName, accessModes, resources | pvc |
| NodePool (Karpenter) | karpenter.sh/v1 | template, limits, disruption | (none) |

### kubectl one-liners

```bash
# Get all Pods with their node and IP
kubectl get pods -A -o wide

# Watch Pod status live
kubectl get pods -n <ns> -w

# Get all Pods not in Running/Completed state
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# Describe all events in a namespace sorted by time
kubectl get events -n <ns> --sort-by=.lastTimestamp

# Scale a deployment
kubectl scale deployment <name> -n <ns> --replicas=5

# Rollout restart (triggers rolling redeploy without image change)
kubectl rollout restart deployment/<name> -n <ns>

# Check rollout status
kubectl rollout status deployment/<name> -n <ns>

# Roll back to previous revision
kubectl rollout undo deployment/<name> -n <ns>

# Print environment variables of a running Pod
kubectl exec <pod> -n <ns> -- env

# Port-forward a service to localhost for testing
kubectl port-forward svc/<service> 8080:80 -n <ns>

# Get resource usage across all nodes
kubectl top nodes

# Get resource usage across all Pods
kubectl top pods -A --sort-by=memory

# Decode a Secret value
kubectl get secret <name> -n <ns> -o jsonpath='{.data.<key>}' | base64 -d

# List all images running in the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{.spec.containers[*].image}{"\n"}{end}' | sort -u

# Check which Pods have no resource requests
kubectl get pods -A -o json | jq '.items[] | select(.spec.containers[].resources.requests == null) | .metadata.name'

# Drain a node safely (respects PDBs)
kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data

# Force delete a stuck terminating Pod (last resort)
kubectl delete pod <name> -n <ns> --grace-period=0 --force
```

### EKS version support timeline reminder

```
Version lifecycle (check live table — numbers change):
  Standard support:  14 months from GA release
  Extended support:  +12 months (additional charges)
  Total max:         ~26 months before forced upgrade

Upgrade policy options:
  STANDARD  → cluster auto-upgrades at standard support end
  EXTENDED  → cluster stays on extended support (charged per hour per cluster)

Practical rule: stay within 2 minor versions of the latest release.
Run at most N-1 (ideally current) in production.
```

### Key EKS limits (verify current values in AWS docs)

| Limit | Value | Notes |
|---|---|---|
| Nodes per cluster | 450 (default soft limit) | Increasable via support ticket |
| Pods per node (default, without prefix delegation) | Depends on instance ENI/IP limits | m5.large: ~29; c5.xlarge: ~58 |
| Pods per node (with prefix delegation) | Up to 110 (kubelet default max) | Requires nitro instances |
| Kubernetes namespaces per cluster | No hard limit (practical: hundreds) | Organize by team/environment |
| Max cluster versions behind | N-1 supported upgrade path | Must upgrade sequentially (1.29 → 1.30 → 1.31) |
| Fargate Pods per region | 500 (default, soft limit) | Request increase |
| EKS access entries per cluster | 1,000 | — |
| Pod identity associations per cluster | 1,000 | — |
| Add-ons per cluster | Up to 30 | — |

---

## 11. Practical inspection

Read-only examples, assuming an existing authorized kubeconfig:

```bash
# Cluster info
aws eks describe-cluster --name app-cluster
aws eks list-access-entries --cluster-name app-cluster
aws eks list-pod-identity-associations --cluster-name app-cluster
aws eks list-addons --cluster-name app-cluster

# Node and Pod state
kubectl get nodes -o wide
kubectl get pods -n orders -o wide
kubectl describe pod POD_NAME -n orders
kubectl logs POD_NAME -n orders --previous
kubectl get events -n orders --sort-by=.lastTimestamp

# Networking
kubectl get svc -A
kubectl get ingress -A
kubectl get endpoints -n orders

# RBAC inspection
kubectl get clusterrolebindings -o wide | grep <user-or-group>
kubectl auth can-i list pods --namespace orders --as system:serviceaccount:orders:order-processor-sa
```

---

## 12. Exam tips summary

### Must-know concepts

1. **EKS manages the control plane only** (by default). You own the nodes, workloads, RBAC, and application availability — unless using Auto Mode, which delegates more infrastructure management.
2. **Two separate authorization systems**: access entries / RBAC for Kubernetes API access; Pod Identity / IRSA for AWS API access. A Pod can call S3 but have no Kubernetes RBAC permissions, or vice versa.
3. **Pod Identity is the current preferred mechanism** for pods accessing AWS. IRSA is still valid (and required for Fargate). Never use the node role for application permissions.
4. **CPU limits cause throttling, not OOMKill**. Memory limits exceeding causes OOMKill and container restart. Always set memory requests + limits; CPU limits are optional but watch for CFS throttle.
5. **HPA scales Pods; Karpenter / Cluster Autoscaler scales nodes**. Both are needed for elastic workloads. HPA requires resource requests to calculate utilization percentage.
6. **PDB governs voluntary disruptions only** (node drain, rolling upgrade). It does not protect against node failure, AZ outage, or OOMKill. Real resilience requires replicas across AZs and sufficient capacity.
7. **Fargate requires**: a matching Fargate profile, CPU/memory in every Pod spec, no DaemonSets, no hostNetwork. Good for serverless/bursty/security-sensitive workloads.
8. **Private clusters need VPC endpoints** for ECR, EKS, STS, EC2, S3 (gateway), and CloudWatch Logs — or a controlled NAT path.
9. **EKS upgrades must be sequential** (1.29 → 1.30, not 1.29 → 1.31). Upgrade control plane first, then add-ons, then node groups. Always inspect deprecated APIs before upgrading.
10. **Karpenter is faster and more flexible than Cluster Autoscaler** — it provisions any instance type in ~30 seconds vs. scaling a pre-defined group in 2–5 minutes.
11. **VPC CNI prefix delegation** multiplies Pod density per node from ~30 to ~110 by assigning /28 prefixes instead of individual IPs.
12. **AWS Load Balancer Controller is required** for ALB Ingress. Auto Mode includes it. Without it, Ingress resources will have no ADDRESS and traffic will not flow.

### Common exam scenarios

| Scenario | Answer |
|---|---|
| Pods can't reach S3 | Check Pod Identity association or IRSA annotation; verify role trust policy; check node role does NOT have S3 access (to avoid false positives) |
| Pod stays Pending after adding nodes | Check taints/tolerations, affinity rules, volume topology (EBS AZ mismatch), or IP exhaustion |
| Rolling upgrade stuck | A PDB is blocking eviction; reduce minAvailable temporarily, or fix the unhealthy Pods before draining |
| All Pods OOMKilled after traffic spike | Memory limits too low; set appropriate limits and consider VPA for recommendations |
| Developer can deploy but can't read secrets | RBAC Role/binding missing `get` on `secrets` in that namespace |
| New team needs access — least privilege | Create access entry for their IAM role with kubernetes-groups; write a Role/ClusterRole with only needed verbs; create RoleBinding in their namespace |
| Cost spike after Karpenter migration | NodePool allows too many large instances; add consolidation policy and instance size upper bounds |

### Red flags in exam questions

- ❌ **Node role used for application AWS access** → Use Pod Identity or IRSA
- ❌ **Missing resource requests** → HPA won't work; scheduling is random; costs unpredictable
- ❌ **No PDB before node group upgrade** → Potential zero-replica window during drain
- ❌ **Skipping a minor version during upgrade** → Not supported; must upgrade sequentially
- ❌ **Secret stored as plaintext env var** → Use Kubernetes Secret or AWS Secrets Manager CSI driver
- ❌ **Single replica with no PDB** → Any node drain = downtime
- ❌ **Public cluster endpoint with no IP allowlist** → Kubernetes API exposed to internet
- ❌ **Auto Mode with hardcoded node SSH scripts** → Nodes are immutable; SSH workflow breaks
- ❌ **IRSA trust policy missing namespace/SA condition** → Any service account in the cluster can assume the role

---

## 13. Interview questions and model answers

**Q1: When would you choose EKS over ECS?**

Choose EKS when you have a specific Kubernetes requirement: the team already knows Kubernetes, you need to run Helm charts or Kubernetes-native ecosystem tools (Argo CD, Istio, Kyverno), you want portability across clouds/on-premises (via Kubernetes APIs), or your organization mandates Kubernetes as the platform standard. ECS is simpler to operate for teams that don't need Kubernetes APIs — fewer components, tighter AWS integration, and a shallower learning curve. The honest answer is that EKS has a higher operational cost; justify it with a real requirement, not "everyone uses Kubernetes."

---

**Q2: How does Pod Identity differ from IRSA?**

Both let a Pod assume an IAM role without long-term credentials.

IRSA works by projecting a service account JWT into the Pod, which the SDK exchanges for temporary credentials via STS using the cluster's OIDC provider. Each IAM role trust policy must be scoped to a specific OIDC URL, namespace, and service account name.

Pod Identity works via an agent (DaemonSet or built-in in Auto Mode) that listens on a credential endpoint injected into matching Pods. The IAM role trust policy simply trusts `pods.eks.amazonaws.com`, and the association (cluster, namespace, service account → role) is managed in EKS, not in the IAM trust policy. This is simpler to set up and audit, but requires the agent and is not supported on Fargate.

---

**Q3: How does Karpenter differ from Cluster Autoscaler?**

Cluster Autoscaler works with pre-defined node groups (Auto Scaling Groups). It scales up an existing group when Pods are unschedulable and scales down when nodes are underutilized. It can only provision instance types that match a node group definition, and scaling takes 2–5 minutes.

Karpenter watches for unschedulable Pods directly and launches any EC2 instance type that satisfies the Pod's requirements (CPU, memory, GPU, architecture, capacity type). It provisions a node in ~30 seconds and can consolidate underutilized nodes by moving Pods and terminating the emptied node. This gives better bin-packing, faster scale-out, and lower cost through spot/Graviton diversity. The trade-off is more configuration complexity (NodePool, EC2NodeClass).

---

**Q4: What does EKS Auto Mode change, and what doesn't it change?**

Auto Mode extends AWS management to include: compute provisioning and scaling, node OS updates (immutable nodes), the VPC CNI, Pod Identity agent, EBS/EFS CSI drivers, load balancer controller, and CoreDNS/kube-proxy equivalents.

It does not change: your workload manifests, resource requests/limits, RBAC configuration, Pod Disruption Budgets, IAM policies, application logic, or any Kubernetes API object you write. Operationally, you gain less infrastructure toil but still own application correctness and availability.

---

**Q5: Walk me through setting up RBAC for a new developer team.**

1. Create an IAM role for the team (e.g., `DevTeamRole`) and grant the team's users permission to assume it.
2. Create an EKS access entry mapping the IAM role ARN to a Kubernetes group (e.g., `dev-team`).
3. Write a Kubernetes `Role` in the team's namespace with only the verbs they need (get, list, watch, create, update on deployments; get, list on pods and logs).
4. Create a `RoleBinding` binding `dev-team` group to that `Role` in the namespace.
5. Do not grant ClusterAdmin or cluster-wide roles unless explicitly required.
6. Validate with `kubectl auth can-i --list --as=oidc:<their-role-arn>` or have a team member test.

---

**Q6: How does a PDB protect a workload, and what doesn't it protect against?**

A PDB tells Kubernetes: "during a voluntary eviction, do not bring the number of available Pods below this threshold." Node drains, rolling upgrades, and Karpenter consolidation all respect PDBs.

PDBs do not protect against: node hardware failure, AZ outage, OOMKilled containers, Pods crashing due to application bugs, or any involuntary disruption. For real resilience you need multiple replicas spread across AZs, sufficient capacity headroom, and meaningful readiness/liveness probes.

---

**Q7: Describe your EKS upgrade strategy for a production cluster.**

Pre-upgrade:
- Scan manifests for deprecated API versions (pluto or kubectl convert).
- Review EKS add-on compatibility matrix for the target version.
- Confirm all critical Deployments have PDBs and at least 2 replicas.
- Rehearse the upgrade sequence in staging with representative traffic.

Upgrade order:
1. Control plane: `aws eks update-cluster-version`. Wait for ACTIVE status.
2. Add-ons: update vpc-cni, coredns, kube-proxy to compatible versions.
3. Managed node groups: `aws eks update-nodegroup-version`. AWS drains old nodes in rolling fashion, respecting PDBs.
4. Karpenter: update to a certified version for the new Kubernetes version.
5. Validate: all nodes show new version, no Pods stuck, run smoke tests.

Post-upgrade: update infrastructure-as-code to reflect new version; schedule next upgrade before standard support window closes.

---

**Q8: A Pod gets `AccessDenied` calling DynamoDB. Walk through your debug process.**

1. `kubectl exec -it <pod> -- env | grep AWS` — confirm `AWS_ROLE_ARN` is set (Pod Identity) or `AWS_WEB_IDENTITY_TOKEN_FILE` (IRSA).
2. If neither is set, the Pod is falling back to the node role (or no role). Create a Pod Identity association or IRSA annotation.
3. If `AWS_ROLE_ARN` is set, check `aws eks list-pod-identity-associations` — confirm the association exists and matches the correct namespace and service account name.
4. Check the IAM role's permission policy — confirm it allows `dynamodb:GetItem` (or whatever action) on the specific table ARN.
5. Check for a resource-based policy on the DynamoDB table (rare but possible).
6. Check no SCP is blocking DynamoDB in the account.
7. Use `aws sts decode-authorization-message --encoded-message <message>` if you have the encoded error.

---

**Q9: What is the difference between a Kubernetes Service of type ClusterIP, NodePort, and LoadBalancer?**

- **ClusterIP** (default): stable internal virtual IP reachable only within the cluster. Used for service discovery between Pods.
- **NodePort**: exposes the Service on a static port on every node's IP. Accessible from outside the cluster via `<NodeIP>:<NodePort>`. Rarely used in production directly; usually used as a stepping stone.
- **LoadBalancer**: creates an external load balancer (NLB with AWS LBC; legacy CLB with in-tree controller). Appropriate for exposing a service externally but creates one load balancer per Service — use Ingress with ALB for HTTP/S to share one LB across many services.

---

**Q10: Why can't you just increase pod limits by adding more nodes when pods are pending due to affinity rules?**

Adding nodes expands capacity for scheduling, but it does not make affinity/toleration constraints satisfiable. If a Pod requires `nodeSelector: zone=us-east-1a` and all nodes in `us-east-1a` are full but you add nodes in `us-east-1b`, the Pod remains unschedulable because the constraint is not met. Similarly, if a Pod has a required anti-affinity rule (do not co-locate with another app), and both the new and old nodes already host that app, more nodes do not help. You must fix the constraint — change the affinity rule, add the required node label, or adjust the topology.

---

**Q11: How would you debug a CrashLoopBackOff?**

```
1. kubectl describe pod <name> -n <ns>
   → Look at: Last State exit code, restart count, probe config, Events

Exit code guide:
  0   → Process exited successfully but unexpectedly (check entrypoint/command)
  1   → Application error (check app logs)
  137 → SIGKILL — OOMKilled (memory limit too low) or force-deleted
  139 → Segfault (application crash)
  143 → SIGTERM graceful shutdown (check liveness probe)

2. kubectl logs <name> -n <ns> --previous
   → Read last crash logs before the container restarted

3. Common causes:
   → Missing required environment variable or secret
   → Wrong command/entrypoint in the Deployment
   → Liveness probe failing before app is ready (add startupProbe)
   → OOMKill: JVM heap settings (-Xmx) larger than memory limit
   → Application bug at startup
```

---

**Q12: What is the difference between EKS and EKS Anywhere?**

EKS is a managed Kubernetes control plane running on AWS infrastructure. You manage workloads; AWS manages the control plane VMs, etcd, and API server availability.

EKS Anywhere lets you run an EKS-compatible Kubernetes cluster on your own on-premises hardware (or VMware/bare-metal/Snow). You own the full cluster, including the control plane. AWS provides the EKS Distro (the same Kubernetes distribution EKS uses), curated add-ons, and optional connected mode for console visibility. EKS Anywhere is chosen when data residency, latency, or air-gapped requirements prevent using AWS regions.

---

## 14. AWS sources

- [EKS Auto Mode](https://docs.aws.amazon.com/eks/latest/userguide/automode.html)
- [Cluster access control (access entries)](https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html)
- [EKS Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html)
- [IAM roles for service accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)
- [Kubernetes version lifecycle](https://docs.aws.amazon.com/eks/latest/userguide/kubernetes-versions.html)
- [VPC CNI prefix delegation](https://docs.aws.amazon.com/eks/latest/userguide/cni-increase-ip-addresses.html)
- [Security groups for Pods](https://docs.aws.amazon.com/eks/latest/userguide/security-groups-for-pods.html)
- [AWS Load Balancer Controller](https://kubernetes-sigs.github.io/aws-load-balancer-controller/)
- [Karpenter documentation](https://karpenter.sh/docs/)
- [EKS managed node groups](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html)
- [EKS Fargate](https://docs.aws.amazon.com/eks/latest/userguide/fargate.html)

**Related:** [ECS](aws-ecs-cheatsheet.md), [EC2](aws-ec2-cheatsheet.md), [ELB](aws-elb-cheatsheet.md), [IAM](aws-iam-cheatsheet.md), [VPC](aws-vpc-networking-cheatsheet.md).
