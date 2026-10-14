# AWS IAM Cheat Sheet

**Complete Guide to Identity and Access Management for AWS SAA-C03 Certification**

---

## Table of Contents

1. [IAM Fundamentals](#1-iam-fundamentals)
2. [Identity Types](#2-identity-types)
3. [Authentication](#3-authentication)
4. [Authorization & Policies](#4-authorization--policies)
5. [Policy Evaluation Logic](#5-policy-evaluation-logic)
6. [IAM Roles Deep Dive](#6-iam-roles-deep-dive)
7. [Cross-Account Access](#7-cross-account-access)
8. [Federation & SSO](#8-federation--sso)
9. [Service Control Policies (SCPs)](#9-service-control-policies-scps)
10. [Permission Boundaries](#10-permission-boundaries)
11. [Security Best Practices](#11-security-best-practices)
12. [Common Patterns](#12-common-patterns)
13. [Troubleshooting](#13-troubleshooting)
14. [Policy Examples](#14-policy-examples)

---

## 1. IAM Fundamentals

### Overview
AWS Identity and Access Management (IAM) is a global service that controls authentication (who you are) and authorization (what you can do) for AWS resources.

### Core Concepts

| Component | Description | Scope |
|-----------|-------------|-------|
| **Principal** | Entity making a request | User, Role, Service |
| **Authentication** | Proving identity | Credentials |
| **Authorization** | Permission to act | Policies |
| **Policy** | JSON document defining permissions | Attached to identities |
| **Resource** | AWS object being accessed | S3 bucket, EC2, etc. |

### Mental Model: Authentication vs Authorization

```
Authentication = "Who are you?"
├── IAM Users (username + password)
├── IAM Roles (temporary credentials)
├── Federated Users (external identity)
└── Service Principals (AWS services)

Authorization = "What can you do?"
├── Identity-based Policies (attached to users/roles)
├── Resource-based Policies (attached to resources)
├── Permission Boundaries (set max permissions)
└── SCPs (organizational guardrails)
```

### IAM is Global
- Not region-specific
- Users, groups, roles, and policies exist across all regions
- Resources are regional, but IAM controls are global
- Changes take effect immediately (eventual consistency)

### Key Limits

| Resource | Default Limit | Notes |
|----------|---------------|-------|
| Users per account | 5,000 | Use federation for more |
| Groups per account | 300 | Groups cannot be nested |
| Roles per account | 1,000 | Soft limit, increasable |
| Managed policies per user/role/group | 10 | Plus 1 inline policy |
| Policy size | 2,048 chars (inline)<br/>6,144 chars (managed) | JSON format |
| MFA devices per user | 8 | Virtual + hardware |

### Exam Tip
**IAM is free.** No charge for users, groups, roles, or policies. You only pay for AWS resources accessed. IAM changes are eventually consistent but typically take effect within seconds.

---

## 2. Identity Types

### IAM Users

#### Overview
IAM Users represent permanent, long-term identities for people or applications that need consistent access to AWS.

#### Characteristics
- **Permanent credentials:** Password and/or access keys
- **Individual identity:** Each person gets their own user
- **Direct authentication:** Authenticates directly to AWS
- **Maximum:** 5,000 users per account

#### Use Cases
- Individual employees needing AWS Console access
- Service accounts for applications (legacy, not recommended)
- Break-glass admin access
- CI/CD pipelines (not recommended, use roles instead)

#### Mental Model
```
IAM User = Permanent Identity
├── Console Access: Username + Password + MFA
├── Programmatic Access: Access Key + Secret Key
├── Policies: Attached directly or via groups
└── Credentials: Long-lived (rotate regularly)
```

#### Best Practices
- Enable MFA for all users
- Use groups to assign permissions, not individual policies
- Rotate access keys regularly (max 90 days)
- Delete unused users
- Enforce strong password policy
- Never share user credentials
- Avoid using root user (create IAM admin instead)

#### Exam Tip
**One person = One IAM user.** Never share credentials. For applications, use IAM roles, not IAM users with access keys.

---

### IAM Groups

#### Overview
Groups are collections of IAM users. Permissions attached to groups apply to all members.

#### Characteristics
- **Simplify management:** Assign permissions once to group
- **No nesting:** Groups cannot contain other groups
- **No identity:** Cannot be referenced in resource policies
- **Limit:** 300 groups per account

#### Mental Model
```
Group Membership
├── Users can belong to multiple groups (max 10)
├── Permissions are union of all group policies
├── Groups have no credentials
└── Groups exist only for permission management

Example Hierarchy:
├── Developers Group → Read/Write Dev resources
├── Admins Group → Full access
├── Auditors Group → Read-only all resources
└── Alice (user) → Member of Developers + Admins
```

#### Common Patterns
```
Organization Structure:
├── Admin Group (PowerUserAccess or AdministratorAccess)
├── Developer Group (EC2, Lambda, RDS in dev/test)
├── Data Engineer Group (S3, Glue, Athena, Redshift)
├── Security Group (CloudTrail, GuardDuty, SecurityHub)
└── ReadOnly Group (ViewOnlyAccess)
```

#### Best Practices
- Create groups based on job function, not projects
- Attach policies to groups, not individual users
- Use descriptive names: `Developers`, `DataEngineers`, `SecurityTeam`
- Document group purposes
- Regular access reviews

#### Exam Tip
**Groups simplify permission management but are not identities.** Cannot use groups as principals in S3 bucket policies or assume roles. Groups are just a way to attach policies to multiple users.

---

### IAM Roles

#### Overview
IAM roles are identities with defined permissions that can be assumed by authorized principals—including IAM users, 
other IAM roles, AWS services, and federated identities—to obtain temporary security credentials.

#### Characteristics
- **Temporary credentials:** Auto-rotating, time-limited
- **No permanent credentials:** No password or access keys
- **Assumable:** Can be assumed by users, services, or accounts
- **Trust policy:** Defines who can assume the role

#### Mental Model
```
IAM Role = Temporary Identity
├── Trust Policy: WHO can assume this role
├── Permission Policy: WHAT the role can do
├── Temporary Credentials: Valid 15 min to 12 hours
└── Use Cases: EC2, Lambda, Cross-account, Federation

Flow:
1. Principal requests to assume role (AssumeRole API)
2. STS validates trust policy
3. STS issues temporary credentials
4. Principal uses credentials (auto-refresh)
5. Credentials expire (automatic cleanup)
```

#### Role Types

| Type | Purpose | Example |
|------|---------|---------|
| **Service Role** | AWS service assumes | EC2, Lambda |
| **Cross-Account Role** | Another AWS account assumes | Account 111 → Account 222 |
| **Federated Role** | External identity assumes | SAML, OIDC, Web Identity |
| **Instance Profile** | EC2 metadata service | App on EC2 |

#### Use Cases
- EC2 instances accessing S3, DynamoDB
- Lambda functions accessing AWS services
- Cross-account access without sharing credentials
- Federated users from corporate directory
- Applications running in ECS, EKS, Fargate
- CI/CD pipelines (GitHub Actions, Jenkins)

#### Best Practices
- Use roles instead of access keys for applications
- Set minimum session duration needed
- Use conditions in trust policy for security
- Regularly review role usage with Access Analyzer
- Use session tags for attribute-based access control
- Name roles descriptively: `EC2-S3-ReadOnly`, `Lambda-DDB-Writer`

#### Exam Tip
**Roles are the preferred way to grant permissions to AWS services and applications.** More secure than access keys because credentials are temporary and automatically rotated.

---

### Service-Linked Roles

#### Overview
Special IAM roles created and managed by AWS services. Pre-defined trust and permission policies.

#### Characteristics
- **Auto-created:** Created by the service when needed
- **Predefined permissions:** AWS manages the policy
- **Cannot modify:** Trust policy is locked
- **Service-specific:** Each service has its own

#### Examples
```
Common Service-Linked Roles:
├── AWSServiceRoleForAutoScaling
├── AWSServiceRoleForElasticLoadBalancing
├── AWSServiceRoleForRDS
├── AWSServiceRoleForLambda
└── AWSServiceRoleForAmazonGuardDuty
```

#### Exam Tip
**Service-linked roles are automatically created by AWS services.** You typically don't create them manually. Cannot delete until the service no longer needs them.

---

## 3. Authentication

### Root User

#### Overview
The root user is created when you first create an AWS account. Has complete, unrestricted access to all AWS services and resources.

#### Characteristics
- **Email address:** Associated with account email
- **Unrestricted access:** Bypasses all IAM policies
- **Account-level actions:** Only root can perform certain tasks
- **Dangerous:** Cannot be restricted by IAM policies

#### Root-Only Actions
```
Tasks ONLY root user can perform:
├── Change account settings (name, email, password, root access keys)
├── Close AWS account
├── Change AWS Support plan
├── Register as seller in Reserved Instance Marketplace
├── Enable MFA Delete on S3 bucket
├── Create CloudFront key pairs
├── Configure S3 bucket for Requester Pays
└── Sign up for GovCloud
```

#### Security Best Practices
1. **Enable MFA** on root account (hardware token recommended)
2. **Delete root access keys** (never create them)
3. **Create IAM admin user** for daily tasks
4. **Lock root credentials** in secure location
5. **Monitor root usage** with CloudTrail
6. **Use root only for root-only tasks**

#### Exam Tip
**Never use root user for everyday tasks.** Create IAM admin, enable MFA on root, and lock away credentials. Root user bypasses all IAM restrictions.

---

### IAM User Authentication

#### Console Access
```
Username + Password + MFA (optional but recommended)
├── Password Policy: Complexity requirements
├── MFA Device: Virtual (app) or Hardware (token)
└── Login URL: https://[account-id-or-alias].signin.aws.amazon.com/console
```

#### Programmatic Access
```
Access Key ID + Secret Access Key
├── Used with: AWS CLI, SDKs, API calls
├── Format: AKIA... (long-term), ASIA... (temporary)
├── Limit: 2 active access keys per user
├── Rotation: Recommended every 90 days
└── Scope: Same permissions as the user
```

#### MFA Types

| Type | Description | Use Case |
|------|-------------|----------|
| **Virtual MFA** | Authenticator app (Google, Authy, Microsoft) | Most common, free |
| **Hardware MFA** | Physical token (Gemalto, YubiKey) | High security environments |
| **U2F Security Key** | USB device (YubiKey) | Phishing-resistant |
| **SMS MFA** | Text message (deprecated) | Not recommended |

#### Password Policy Configuration
```json
{
  "MinimumPasswordLength": 14,
  "RequireSymbols": true,
  "RequireNumbers": true,
  "RequireUppercaseCharacters": true,
  "RequireLowercaseCharacters": true,
  "AllowUsersToChangePassword": true,
  "ExpirePasswords": true,
  "MaxPasswordAge": 90,
  "PasswordReusePrevention": 24,
  "HardExpiry": false
}
```

#### Access Key Best Practices
- **Rotate regularly:** Every 90 days maximum
- **Audit usage:** Check CloudTrail for last used
- **Delete unused keys:** Use Access Advisor
- **Never commit to code:** Use AWS Secrets Manager
- **Prefer roles:** Use roles over access keys when possible

#### Exam Tip
**Access keys starting with AKIA are long-term credentials (IAM users). ASIA are temporary (STS/roles).** MFA adds a second factor, making authentication much more secure.

---

### STS (Security Token Service)

#### Overview
STS generates temporary, limited-privilege credentials for trusted entities to access AWS resources.

#### Characteristics
- **Temporary:** Expiring credentials (15 min to 12 hours)
- **Auto-rotating:** Refreshed automatically before expiry
- **Limited scope:** Can restrict permissions via session policy
- **Global service:** With regional endpoints available

#### Key STS API Actions

| Action | Purpose | Who Uses |
|--------|---------|----------|
| **AssumeRole** | Assume an IAM role | Users, services, accounts |
| **AssumeRoleWithSAML** | Federated access via SAML | Corporate SSO (AD FS, Okta) |
| **AssumeRoleWithWebIdentity** | Web identity federation | Mobile apps (deprecated for AWS resources) |
| **GetSessionToken** | MFA-authenticated session | IAM users with MFA |
| **GetFederationToken** | Custom federation | Legacy applications |

#### Mental Model
```
STS Credential Flow:
1. Principal calls STS API (e.g., AssumeRole)
2. STS validates trust policy and permissions
3. STS generates temporary credentials:
   ├── Access Key ID (starts with ASIA)
   ├── Secret Access Key
   ├── Session Token
   └── Expiration timestamp
4. Principal uses credentials to call AWS APIs
5. Credentials auto-refresh (SDK handles this)
6. Credentials expire (automatic cleanup)
```

#### Session Duration
```
Default and Maximum Session Durations:
├── AssumeRole: 1 hour (default), up to 12 hours
├── Console Federation: 1 hour (default), up to 12 hours
├── AssumeRoleWithSAML: 1 hour (default), up to 12 hours
└── GetSessionToken: 12 hours (default), up to 36 hours (IAM user)
```

#### Session Policies
Optional JSON policies passed during AssumeRole to further restrict permissions.

```
Effective Permissions = Role Permissions ∩ Session Policy
(Both must allow the action)

Example: Role has S3 full access, session policy allows only read
→ Result: Read-only access to S3
```

#### Exam Tip
**STS credentials are temporary and start with ASIA.** Session policies can only restrict permissions, never expand them. Used for federation, cross-account access, and service roles.

---

## 4. Authorization & Policies

### Policy Types Overview

| Policy Type | Attached To | Purpose |
|-------------|-------------|---------|
| **Identity-based** | Users, Groups, Roles | What an identity can do |
| **Resource-based** | Resources (S3, SQS, SNS) | Who can access this resource |
| **Permission Boundaries** | Users, Roles | Maximum permissions allowed |
| **Service Control Policies (SCP)** | Organization/OU/Account | AWS Organizations guardrails |
| **Access Control Lists (ACL)** | S3 objects, buckets | Legacy access control |
| **Session Policies** | STS AssumeRole | Temporary restriction |

### Policy Structure

#### Basic JSON Format
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DescriptiveIdentifier",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:user/Alice"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mybucket/*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

#### Statement Elements

| Element | Required | Description | Example |
|---------|----------|-------------|---------|
| **Version** | Yes | Policy language version | "2012-10-17" |
| **Statement** | Yes | List of permissions | Array of statements |
| **Sid** | No | Statement ID | "AllowS3Read" |
| **Effect** | Yes | Allow or Deny | "Allow" or "Deny" |
| **Principal** | Conditional | Who (resource-based only) | "AWS": "arn..." |
| **Action** | Yes | API actions | "s3:GetObject" |
| **Resource** | Yes | AWS resources | "arn:aws:s3:::bucket/*" |
| **Condition** | No | Contextual constraints | IP, time, tags, etc. |

#### Mental Model
```
Policy Statement = WHO can do WHAT on WHICH resource WHEN

WHO (Principal)     → Only in resource-based policies
WHAT (Action)       → API operations
WHICH (Resource)    → ARN of AWS resource
WHEN (Condition)    → Optional constraints
EFFECT              → Allow or Deny
```

### Identity-Based Policies

#### Overview
Attached to IAM users, groups, or roles. Define what actions the identity can perform.

#### Types

**1. Managed Policies**
```
AWS Managed Policies:
├── Created and managed by AWS
├── Examples: AdministratorAccess, ReadOnlyAccess, PowerUserAccess
├── Cannot be modified
├── Updated by AWS when new services launch
└── Shared across multiple users/roles

Customer Managed Policies:
├── Created by you
├── Reusable across users/roles/groups
├── Version controlled (up to 5 versions)
├── Can be modified anytime
└── Recommended for common permission sets
```

**2. Inline Policies**
```
Inline Policy:
├── Embedded directly into a user/role/group
├── 1:1 relationship with identity
├── Deleted when identity is deleted
├── Use for strict 1:1 relationship only
└── Not reusable
```

#### Example: S3 Read-Only Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

#### Best Practices
- Prefer managed policies over inline
- Use customer managed policies for common patterns
- Start with AWS managed policies and customize
- Follow principle of least privilege
- Use wildcards carefully
- Regular access reviews with Access Analyzer

#### Exam Tip
**Managed policies are reusable and version-controlled. Inline policies have 1:1 relationship with identity.** AWS managed policies are updated automatically when AWS launches new services.

---

### Resource-Based Policies

#### Overview
Attached directly to AWS resources. Define who can access the resource and what actions they can perform.

#### Supported Resources
```
Resources with Resource-Based Policies:
├── S3 Buckets (Bucket Policy)
├── S3 Glacier Vaults (Vault Access Policy)
├── SNS Topics (Topic Policy)
├── SQS Queues (Queue Policy)
├── Lambda Functions (Function Policy)
├── API Gateway (Resource Policy)
├── Secrets Manager (Resource Policy)
├── KMS Keys (Key Policy)
├── ECR Repositories (Repository Policy)
└── CloudWatch Logs (Resource Policy)
```

#### Key Difference: Principal Element
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mybucket/*"
    }
  ]
}
```

#### Principal Types

| Principal Type | Syntax | Example |
|----------------|--------|---------|
| **AWS Account** | "AWS": "account-id" | "AWS": "123456789012" |
| **IAM User** | "AWS": "arn:aws:iam::..." | "AWS": "arn:aws:iam::123456789012:user/Alice" |
| **IAM Role** | "AWS": "arn:aws:iam::..." | "AWS": "arn:aws:iam::123456789012:role/MyRole" |
| **Federated User** | "Federated": "..." | "Federated": "cognito-identity.amazonaws.com" |
| **AWS Service** | "Service": "service.amazonaws.com" | "Service": "lambda.amazonaws.com" |
| **Everyone** | "*" | "*" (public access) |

#### S3 Bucket Policy Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCrossAccountRead",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::mybucket/*"
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::mybucket",
        "arn:aws:s3:::mybucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

#### Identity vs Resource Policy

| Aspect | Identity-Based | Resource-Based |
|--------|----------------|----------------|
| **Attached to** | User/Role/Group | Resource |
| **Principal** | Not specified (implicit) | Explicitly specified |
| **Cross-account** | Must assume role | Can grant directly |
| **Use case** | Grant permissions to identity | Control resource access |
| **Evaluation** | OR logic with resource policy | OR logic with identity policy |

#### Exam Tip
**Resource-based policies have a Principal element. Identity-based policies don't.** For cross-account access, resource policies allow direct access without role assumption. Both policies use OR logic (if either allows, access is granted).

---

### Trust Policies (Roles)

#### Overview
Special type of resource-based policy attached to IAM roles. Defines who or what can assume the role.

#### Structure
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Common Trust Policy Patterns

**1. Service Role (EC2)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**2. Cross-Account Role**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id-12345"
        }
      }
    }
  ]
}
```

**3. SAML Federation**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/MyProvider"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

**4. OIDC/Web Identity (GitHub Actions)**
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
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

#### External ID Pattern
Used in cross-account scenarios to prevent the "confused deputy" problem.

```
Confused Deputy Problem:
├── Your AWS Account (123) creates role for Partner (456)
├── Malicious actor tells partner "access MY account 123"
├── Partner accidentally accesses your data
└── Solution: External ID (secret shared between you and partner)

Trust Policy with External ID:
"Condition": {
  "StringEquals": {
    "sts:ExternalId": "unique-secret-id"
  }
}
```

#### Exam Tip
**Trust policy defines WHO can assume the role. Permission policy defines WHAT the role can do.** External ID prevents confused deputy attacks in cross-account scenarios.

---

## 5. Policy Evaluation Logic

### The Policy Evaluation Flow

#### Mental Model
```
Policy Evaluation = Implicit Deny → Explicit Allow → Explicit Deny

Step 1: Start with Implicit DENY (default)
Step 2: Evaluate all applicable policies for ALLOW
Step 3: Check for any Explicit DENY (overrides all allows)
Step 4: Final decision: ALLOW or DENY
```

#### Evaluation Order

```mermaid
flowchart TD
    Start([Request Made]) --> Authenticate{Authenticated?}
    Authenticate -->|No| Deny1[DENY]
    Authenticate -->|Yes| OrgSCP{Organization SCP?}
    
    OrgSCP -->|Denies| Deny2[DENY]
    OrgSCP -->|Allows or N/A| PermBoundary{Permission Boundary?}
    
    PermBoundary -->|Denies| Deny3[DENY]
    PermBoundary -->|Allows or N/A| IdentityPolicy{Identity-Based Policy?}
    
    IdentityPolicy -->|No Allow| ResourcePolicy{Resource-Based Policy?}
    IdentityPolicy -->|Allows| ExplicitDeny1{Explicit Deny?}
    
    ResourcePolicy -->|No Allow| Deny4[DENY]
    ResourcePolicy -->|Allows| ExplicitDeny2{Explicit Deny?}
    
    ExplicitDeny1 -->|Yes| Deny5[DENY]
    ExplicitDeny1 -->|No| Allow1[ALLOW]
    
    ExplicitDeny2 -->|Yes| Deny6[DENY]
    ExplicitDeny2 -->|No| Allow2[ALLOW]
```

#### The Rules

1. **By default, everything is denied** (Implicit Deny)
2. **An explicit ALLOW overrides the implicit deny**
3. **An explicit DENY overrides any allows** (Deny always wins)
4. **Multiple policies are evaluated with OR logic** (any allow = allowed)
5. **Within same account: Identity OR Resource policy can allow**
6. **Cross-account: BOTH identity AND resource policy must allow**

#### Same-Account Access
```
Scenario: User Alice in Account 123 accessing S3 bucket in Account 123

Evaluation:
├── Identity-Based Policy (on Alice) → Allow S3:GetObject?
├── OR
├── Resource-Based Policy (on bucket) → Allow Alice?
└── Result: If EITHER allows → Access granted (unless explicit deny)
```

#### Cross-Account Access
```
Scenario: User Bob in Account 456 accessing S3 bucket in Account 123

Evaluation:
├── Identity-Based Policy (on Bob in 456) → Allow S3:GetObject?
├── AND
├── Resource-Based Policy (on bucket in 123) → Allow Bob/Account 456?
└── Result: BOTH must allow → Access granted (unless explicit deny)
```

### Policy Combination Logic

#### Multiple Policy Types
```
Final Permission = 
  (SCP ∩ Permission Boundary ∩ Identity Policy) 
  OR 
  Resource Policy
  
  MINUS
  
  Any Explicit Deny

Where:
∩ = Intersection (all must allow)
OR = Union (any can allow)
MINUS = Deny always wins
```

#### Example Scenario
```
Given:
├── Identity Policy: Allow s3:*, ec2:*, dynamodb:*
├── Permission Boundary: Allow s3:*, ec2:*
├── SCP: Allow s3:*, dynamodb:*
└── Explicit Deny: Deny s3:DeleteBucket

Effective Permissions:
├── s3:* ∩ s3:* ∩ s3:* = s3:* (except DeleteBucket - explicit deny)
├── ec2:* ∩ ec2:* ∩ (not in SCP) = DENIED
├── dynamodb:* ∩ (not in boundary) ∩ dynamodb:* = DENIED
└── Final: S3 operations except DeleteBucket
```

### Conditions in Policies

#### Common Condition Keys

| Condition Key | Description | Example |
|---------------|-------------|---------|
| **aws:SourceIp** | Source IP address | "aws:SourceIp": "203.0.113.0/24" |
| **aws:SourceVpc** | Source VPC | "aws:SourceVpc": "vpc-12345" |
| **aws:SourceVpce** | VPC endpoint | "aws:SourceVpce": "vpce-12345" |
| **aws:RequestedRegion** | Target region | "aws:RequestedRegion": "us-east-1" |
| **aws:PrincipalOrgID** | AWS Organization ID | "aws:PrincipalOrgID": "o-12345" |
| **aws:PrincipalTag** | Principal's tag | "aws:PrincipalTag/Department": "Engineering" |
| **aws:SecureTransport** | HTTPS/TLS | "aws:SecureTransport": "true" |
| **aws:CurrentTime** | Request time | "aws:CurrentTime": "2024-01-01T00:00:00Z" |
| **aws:MultiFactorAuthPresent** | MFA used | "aws:MultiFactorAuthPresent": "true" |

#### Condition Operators

| Operator | Description | Example |
|----------|-------------|---------|
| **StringEquals** | Exact string match | "StringEquals": {"aws:PrincipalTag/Role": "Admin"} |
| **StringLike** | Wildcard match | "StringLike": {"s3:prefix": ["home/${aws:username}/*"]} |
| **IpAddress** | IP CIDR match | "IpAddress": {"aws:SourceIp": "10.0.0.0/8"} |
| **DateGreaterThan** | After date | "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T00:00:00Z"} |
| **Bool** | Boolean | "Bool": {"aws:SecureTransport": "true"} |
| **NumericLessThan** | Numeric comparison | "NumericLessThan": {"s3:max-keys": "10"} |

#### Example: IP and MFA Restriction
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
        },
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

#### Example: Time-Based Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T09:00:00Z"},
        "DateLessThan": {"aws:CurrentTime": "2024-01-01T17:00:00Z"}
      }
    }
  ]
}
```

#### Example: Tag-Based Access (ABAC)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Owner": "${aws:username}",
          "ec2:ResourceTag/Environment": "Development"
        }
      }
    }
  ]
}
```

#### Exam Tip
**Explicit Deny always wins.** Multiple policies use OR logic (any allow = allowed), but a single deny overrides everything. Cross-account requires both sides to allow.

---

## 6. IAM Roles Deep Dive

### Role Anatomy

```
IAM Role Components:
├── Trust Policy (Who can assume)
├── Permission Policy (What role can do)
├── Permission Boundary (Max permissions)
├── Tags (Metadata)
├── Max Session Duration (15 min - 12 hours)
└── Role ARN (Unique identifier)
```

### Service Roles for EC2

#### Overview
EC2 instances assume roles via instance profiles to access AWS services without embedding credentials.

#### Architecture
```
EC2 Instance
├── Instance Profile (container for role)
│   └── IAM Role (single role per profile)
│       ├── Trust Policy: ec2.amazonaws.com
│       └── Permission Policy: What EC2 can access
├── Instance Metadata Service (IMDS)
│   └── Endpoint: http://169.254.169.254/latest/meta-data/
└── Temporary Credentials (Auto-rotated)
```

#### How It Works
```
1. EC2 instance launches with instance profile attached
2. Application needs to access S3
3. AWS SDK queries IMDS for credentials
4. IMDS returns temporary credentials (auto-rotated)
5. SDK uses credentials to call S3 API
6. Credentials expire and auto-refresh
```

#### Instance Metadata Service (IMDS)

**IMDSv1 (Original)**
```bash
# Get credentials (IMDSv1 - not recommended)
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/RoleName
```

**IMDSv2 (Recommended - Session-based)**
```bash
# Get token first
TOKEN=$(curl -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")

# Use token to get credentials
curl -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/RoleName
```

#### Best Practices
- Always use roles for EC2, never embed access keys
- Enforce IMDSv2 (more secure)
- Use least privilege permission policies
- Separate roles by function (web server, worker, etc.)
- Regularly review role usage with Access Analyzer

#### Exam Tip
**Instance profiles are containers for roles. One role per instance profile.** AWS SDKs automatically fetch credentials from IMDS. IMDSv2 protects against SSRF attacks.

---

### Service Roles for Lambda

#### Overview
Lambda functions execute with an execution role that grants permissions to AWS services.

#### Architecture
```
Lambda Function
├── Execution Role (IAM Role)
│   ├── Trust Policy: lambda.amazonaws.com
│   ├── Permission Policy: What Lambda can access
│   └── AWS Managed Policy: AWSLambdaBasicExecutionRole
├── Temporary Credentials (Managed by Lambda)
└── Environment Variables (Can use for config)

CloudWatch Logs Permission (Required):
├── logs:CreateLogGroup
├── logs:CreateLogStream
└── logs:PutLogEvents
```

#### Execution Role Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    }
  ]
}
```

#### Lambda Resource-Based Policy
Separate from execution role—controls who can invoke the Lambda.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "lambda:InvokeFunction",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:MyFunction",
      "Condition": {
        "StringEquals": {
          "AWS:SourceAccount": "123456789012"
        },
        "ArnLike": {
          "AWS:SourceArn": "arn:aws:s3:::my-bucket"
        }
      }
    }
  ]
}
```

#### Exam Tip
**Lambda execution role = what Lambda can access. Lambda resource-based policy = who can invoke Lambda.** Always include CloudWatch Logs permissions for logging.

---

### Service Roles for ECS/Fargate

#### Overview
ECS tasks use task execution roles (for ECS agent) and task roles (for application).

#### Architecture
```
ECS Task
├── Task Execution Role (ECS Agent)
│   ├── Pull images from ECR
│   ├── Write logs to CloudWatch
│   └── Retrieve secrets from Secrets Manager/SSM
│
└── Task Role (Application Container)
    ├── Application's AWS permissions
    └── S3, DynamoDB, SQS access, etc.

Mental Model:
Task Execution Role = Infrastructure needs (ECS)
Task Role = Application needs (Your code)
```

#### Task Execution Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "secretsmanager:GetSecretValue"
      ],
      "Resource": "*"
    }
  ]
}
```

#### Task Role
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    }
  ]
}
```

#### Exam Tip
**Task Execution Role pulls images and writes logs. Task Role gives application AWS permissions.** Both are required for most ECS deployments.

---

### Assuming Roles

#### AssumeRole Flow
```
Step-by-Step:
1. Principal has permission to call sts:AssumeRole
2. Principal calls AssumeRole API with role ARN
3. STS validates trust policy on target role
4. STS validates principal has sts:AssumeRole permission
5. STS generates temporary credentials (ASIA...)
6. Principal receives credentials (15 min to 12 hours)
7. Principal uses credentials to call AWS APIs
8. Credentials auto-refresh before expiry
```

#### Example: User Assuming Role
```bash
# AWS CLI
aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/MyRole \
  --role-session-name my-session \
  --duration-seconds 3600

# Response (temporary credentials)
{
  "Credentials": {
    "AccessKeyId": "ASIAIOSFODNN7EXAMPLE",
    "SecretAccessKey": "wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY",
    "SessionToken": "FwoGZXIvYXdz...",
    "Expiration": "2024-01-01T12:00:00Z"
  },
  "AssumedRoleUser": {
    "AssumedRoleId": "AROA...:my-session",
    "Arn": "arn:aws:sts::123456789012:assumed-role/MyRole/my-session"
  }
}
```

#### Identity Requiring sts:AssumeRole Permission
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::123456789012:role/MyRole"
    }
  ]
}
```

#### Role Requiring Trust Policy
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111122223333:user/Alice"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

#### Role Chaining
Assuming a role from another assumed role.

```
User → Role A → Role B → Role C

Limitations:
├── Maximum duration: 1 hour (role chaining limits)
├── Cannot exceed 12 hours total
└── Each hop reduces available session time
```

#### Exam Tip
**To assume a role, you need BOTH: 1) sts:AssumeRole permission on your identity, 2) Trust policy on target role allowing you.** Role chaining is limited to 1 hour sessions.

---

## 7. Cross-Account Access

### Pattern 1: Cross-Account Role

#### Overview
Most common pattern for cross-account access. Account A users/services assume role in Account B.

#### Architecture
```
Account A (111111111111)              Account B (222222222222)
├── IAM User: Alice                   ├── IAM Role: CrossAccountRole
│   └── Policy: Allow                 │   ├── Trust Policy: Allow Account A
│       sts:AssumeRole                │   └── Permission Policy: S3 access
│                                     │
└── AssumeRole ───────────────────────> └── Temporary Credentials
                                            └── Access S3 in Account B
```

#### Setup Steps

**1. Account B: Create Role with Trust Policy**
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

**2. Account B: Attach Permission Policy to Role**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::account-b-bucket",
        "arn:aws:s3:::account-b-bucket/*"
      ]
    }
  ]
}
```

**3. Account A: Grant User Permission to Assume Role**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::222222222222:role/CrossAccountRole"
    }
  ]
}
```

**4. Account A: User Assumes Role**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::222222222222:role/CrossAccountRole \
  --role-session-name alice-session \
  --external-id unique-secret-12345
```

#### External ID
Prevents "confused deputy" problem in third-party scenarios.

```
Scenario without External ID:
├── You (Account A) create role for Partner (Account P)
├── Attacker tells Partner "access Account A" (with A's account ID)
├── Partner accidentally accesses your resources
└── Problem: Partner can't distinguish legitimate vs malicious requests

Solution: External ID (shared secret)
├── You give Partner unique external ID
├── Trust policy requires matching external ID
├── Attacker doesn't know external ID
└── Partner only accesses when customer provides correct external ID
```

#### Exam Tip
**Cross-account role is the preferred method.** Requires trust policy in target account and sts:AssumeRole permission in source account. External ID prevents confused deputy attacks.

---

### Pattern 2: Resource-Based Policy

#### Overview
Grant cross-account access directly through resource policy (S3, SNS, SQS, Lambda, etc.).

#### Architecture
```
Account A (111111111111)              Account B (222222222222)
├── IAM User: Alice                   ├── S3 Bucket: account-b-bucket
│   └── Policy: Allow s3:GetObject    │   └── Bucket Policy:
│                                     │       Allow Account A users
└── Direct Access ─────────────────────> S3 GetObject
    (No role assumption needed)
```

#### S3 Bucket Policy (Account B)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:root"
      },
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::account-b-bucket/*"
    }
  ]
}
```

#### Identity Policy (Account A)
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::account-b-bucket/*"
    }
  ]
}
```

#### Cross-Account Policy Evaluation
```
For cross-account access to succeed:
├── Identity policy in Account A → ALLOW
├── AND
├── Resource policy in Account B → ALLOW
└── AND
└── No explicit DENY anywhere

Key Difference from Same-Account:
Same Account: Identity OR Resource policy can allow
Cross-Account: Identity AND Resource policy must BOTH allow
```

#### When to Use Resource Policy vs Role

| Use Case | Recommended Approach |
|----------|---------------------|
| Cross-account S3 access | Resource-based policy (simpler) |
| Cross-account EC2 management | Role (EC2 doesn't support resource policies) |
| Lambda invoked by another account | Resource-based policy |
| Many users need access | Role (easier management) |
| Service-to-service (S3, SNS, SQS) | Resource-based policy |
| Need to track who accessed | Role (role session name in CloudTrail) |

#### Exam Tip
**Resource-based policies allow direct access without role assumption.** Cross-account requires BOTH identity and resource policies to allow. Simpler than roles for supported services.

---

## 8. Federation & SSO

### Federation Overview

#### Mental Model
```
Federation = External Identity → Temporary AWS Credentials

Types:
├── SAML 2.0 (Enterprise SSO - AD FS, Okta, Azure AD)
├── OIDC/Web Identity (Google, Facebook, Amazon - for apps)
├── Custom Identity Broker (Legacy, custom auth)
└── AWS IAM Identity Center (Recommended for enterprise)
```

### SAML 2.0 Federation

#### Overview
Enterprise SSO using SAML identity providers (IdP) like Okta, Azure AD, AD FS.

#### Architecture
```
1. User → Corporate IdP (Okta/Azure AD)
2. IdP authenticates user (username/password/MFA)
3. IdP generates SAML assertion
4. User → AWS Sign-In (SAML assertion)
5. AWS validates assertion
6. AWS calls STS:AssumeRoleWithSAML
7. STS returns temporary credentials
8. User accesses AWS Console/CLI with temp credentials
```

#### Setup Components

**1. SAML Identity Provider in AWS**
```bash
aws iam create-saml-provider \
  --name MyCorpIdP \
  --saml-metadata-document file://metadata.xml
```

**2. IAM Role for SAML Users**
Trust Policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::123456789012:saml-provider/MyCorpIdP"
      },
      "Action": "sts:AssumeRoleWithSAML",
      "Condition": {
        "StringEquals": {
          "SAML:aud": "https://signin.aws.amazon.com/saml"
        }
      }
    }
  ]
}
```

**3. IdP Configuration**
Configure corporate IdP to:
- Generate SAML assertion with Role ARN
- Include SAML attributes: `RoleSessionName`, `Role`
- Redirect to AWS SAML endpoint

#### SAML Assertion Attributes
```xml
<Attribute Name="https://aws.amazon.com/SAML/Attributes/Role">
  <AttributeValue>
    arn:aws:iam::123456789012:role/DevRole,arn:aws:iam::123456789012:saml-provider/MyCorpIdP
  </AttributeValue>
</Attribute>
<Attribute Name="https://aws.amazon.com/SAML/Attributes/RoleSessionName">
  <AttributeValue>alice@example.com</AttributeValue>
</Attribute>
```

#### Use Cases
- Employees accessing AWS Console with corporate credentials
- No need to create IAM users for each employee
- Centralized user management in corporate IdP
- MFA enforcement at IdP level

#### Exam Tip
**SAML 2.0 is for enterprise SSO.** Users authenticate with corporate IdP, receive SAML assertion, exchange for AWS credentials. No IAM users created.

---

### Web Identity Federation (OIDC)

#### Overview
Mobile/web apps authenticate users with public IdPs (Google, Facebook, Amazon) or custom OIDC providers (Auth0).

#### Architecture for Apps (Cognito)
```
1. User → Mobile App
2. App → Google/Facebook (authenticate)
3. Google returns JWT token
4. App → Cognito (exchange JWT for Cognito token)
5. Cognito → STS (AssumeRoleWithWebIdentity)
6. STS returns AWS credentials
7. App accesses AWS services (S3, DynamoDB)
```

#### Architecture for CI/CD (GitHub Actions)
```
1. GitHub Action starts
2. GitHub generates OIDC token
3. Action calls STS:AssumeRoleWithWebIdentity
4. STS validates token against GitHub OIDC provider
5. STS returns temporary credentials
6. Action deploys to AWS with credentials
```

#### GitHub OIDC Setup

**1. Create OIDC Provider in AWS**
```bash
aws iam create-open-id-connect-provider \
  --url https://token.actions.githubusercontent.com \
  --client-id-list sts.amazonaws.com \
  --thumbprint-list 6938fd4d98bab03faadb97b34396831e3780aea1
```

**2. Create Role with OIDC Trust Policy**
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
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
        },
        "StringLike": {
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:*"
        }
      }
    }
  ]
}
```

**3. GitHub Actions Workflow**
```yaml
name: Deploy to AWS
on: [push]
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
          aws-region: us-east-1
      - run: aws s3 ls
```

#### Exam Tip
**OIDC for modern CI/CD and mobile apps.** No long-term credentials stored. GitHub Actions OIDC is preferred over access keys for CI/CD.

---

### AWS IAM Identity Center (AWS SSO)

#### Overview
Successor to AWS SSO. Centralized access management for multiple AWS accounts and cloud applications.

#### Architecture
```
Identity Center
├── Identity Source
│   ├── Identity Center directory (built-in)
│   ├── Active Directory (via AWS Managed AD or AD Connector)
│   └── External IdP (Azure AD, Okta) via SAML
│
├── Permission Sets (templates for IAM roles)
│   ├── Predefined (AdministratorAccess, ViewOnlyAccess)
│   └── Custom (JSON policies)
│
├── Account Assignments
│   ├── User/Group → Permission Set → AWS Account
│   └── Automatically creates roles in target accounts
│
└── Application Assignments
    └── Access to SaaS apps (Salesforce, Office 365)
```

#### Mental Model
```
Permission Set = Template for IAM Role
├── Define once in Identity Center
├── Assign to users/groups for specific accounts
└── Identity Center creates corresponding IAM role

Example:
├── Permission Set: "Developer"
├── Assigned to: DevTeam group
├── For accounts: Dev account, Test account
└── Result: IAM role created in Dev and Test accounts
```

#### Permission Set Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:*",
        "s3:*",
        "dynamodb:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": ["us-east-1", "us-west-2"]
        }
      }
    }
  ]
}
```

#### Benefits
- **Centralized management:** Single place for all AWS accounts
- **No IAM users:** Federation with existing directory
- **Automatic role creation:** Permission sets become roles
- **MFA enforcement:** Configured once, applies everywhere
- **Audit trail:** CloudTrail logs all SSO activities

#### Use Cases
- Organizations with many AWS accounts
- Federated access from Active Directory
- Centralized permission management
- Compliance requirements (audit, MFA)

#### Exam Tip
**IAM Identity Center is the recommended solution for multi-account access management.** Replaces individual IAM users in each account. Uses permission sets to template IAM roles.

---

## 9. Service Control Policies (SCPs)

### Overview
SCPs are organization-level policies that set maximum permissions for accounts. Applied through AWS Organizations.

#### Mental Model
```
SCP = Permission Boundary for Entire AWS Account

SCP Does:
├── Set maximum permissions (guardrails)
├── Apply to all users/roles in account (except root tasks)
├── Restrict at organization, OU, or account level
└── Cannot grant permissions, only limit

SCP Does NOT:
├── Grant permissions (only restricts)
├── Affect service-linked roles
├── Affect root user for account management tasks
└── Apply to management account (always exempt)
```

### SCP Architecture

```
Organization
├── Management Account (SCP doesn't apply to it)
│
├── OU: Production
│   ├── SCP: Restrict to us-east-1, us-west-2
│   ├── Account: Prod-Web (SCP applies)
│   └── Account: Prod-API (SCP applies)
│
└── OU: Development
    ├── SCP: Allow all
    ├── Account: Dev-1 (SCP applies)
    └── Account: Dev-2 (SCP applies)
```

### SCP Strategies

#### 1. Deny List Strategy (Default)
```
Default SCP: FullAWSAccess (allows everything)
├── Add explicit Deny SCPs for restrictions
└── Everything else is allowed

Example: Deny all except safe services
```

**Default FullAWSAccess SCP:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    }
  ]
}
```

**Additional Deny SCP:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance"
      ],
      "Resource": "*"
    }
  ]
}
```

#### 2. Allow List Strategy
```
Remove FullAWSAccess SCP
├── Explicitly allow only approved services
└── Everything else implicitly denied (more secure)

Example: Only allow S3, EC2, Lambda
```

**Allow List SCP:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "ec2:*",
        "lambda:*",
        "iam:*",
        "organizations:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Common SCP Patterns

#### 1. Restrict Regions
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        },
        "NotAction": [
          "iam:*",
          "organizations:*",
          "route53:*",
          "cloudfront:*",
          "support:*",
          "budgets:*"
        ]
      }
    }
  ]
}
```

#### 2. Require MFA
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

#### 3. Prevent Root User Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringLike": {
          "aws:PrincipalArn": "arn:aws:iam::*:root"
        }
      }
    }
  ]
}
```

#### 4. Deny Leaving Organization
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "organizations:LeaveOrganization",
      "Resource": "*"
    }
  ]
}
```

#### 5. Enforce Encryption
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "s3:PutObject",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    }
  ]
}
```

### SCP Inheritance

```
Organization Root (SCP A)
│
├── OU: Production (SCP B)
│   ├── Account: Prod-1 (Effective: A ∩ B)
│   └── OU: Web (SCP C)
│       └── Account: Prod-Web (Effective: A ∩ B ∩ C)
│
└── OU: Development (SCP D)
    └── Account: Dev-1 (Effective: A ∩ D)

Effective Permission = Intersection of all SCPs in hierarchy
```

### SCP + Identity Policy

```
Effective Permission = SCP ∩ Identity Policy

Example:
├── SCP: Allow S3, EC2, Lambda only
├── Identity Policy: Allow S3:*, RDS:*, Lambda:*
└── Effective: S3:*, Lambda:* (RDS blocked by SCP)

Key Point: SCP cannot grant permissions, only restrict
```

### Exam Tip
**SCPs set maximum permissions for accounts.** Management account is exempt. SCPs don't grant permissions, only limit. Use deny list (default) or allow list strategy.

---

## 10. Permission Boundaries

### Overview
Permission boundaries set the maximum permissions a user or role can have, regardless of identity policies.

#### Mental Model
```
Permission Boundary = Maximum Permission Ceiling

Effective Permission = Identity Policy ∩ Permission Boundary
(Both must allow)

Use Case: Delegated administration with safety limits
├── Developer can create roles
├── But roles cannot exceed boundary
└── Prevents privilege escalation
```

### Architecture

```
Scenario: Junior admin creates IAM roles
├── Admin Identity Policy: Allow iam:CreateRole, iam:AttachRolePolicy
├── Admin Permission Boundary: Allow S3, DynamoDB only
├── Admin creates role with EC2 permissions → Role cannot use EC2
└── Boundary ensures admin cannot create roles exceeding their boundary
```

### Permission Boundary Policy Example

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "logs:*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Delegated Admin with Boundaries

**1. Permission Boundary Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowedServices",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**2. Developer Policy (can create roles)**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRoleWithBoundary",
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    },
    {
      "Sid": "AllowAllWithinBoundary",
      "Effect": "Allow",
      "Action": [
        "s3:*",
        "dynamodb:*",
        "lambda:*",
        "ec2:*"
      ],
      "Resource": "*"
    }
  ]
}
```

**3. Developer applies boundary when creating role**
```bash
aws iam create-role \
  --role-name MyLambdaRole \
  --assume-role-policy-document file://trust-policy.json \
  --permissions-boundary arn:aws:iam::123456789012:policy/DeveloperBoundary

aws iam attach-role-policy \
  --role-name MyLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

**Result:**
```
Developer has: S3, DynamoDB, Lambda, EC2 permissions
Boundary allows: S3, DynamoDB, Lambda only
Created role can use: S3, DynamoDB, Lambda only (EC2 blocked by boundary)
```

### Use Cases
- **Delegated administration:** Developers create roles within limits
- **Prevent privilege escalation:** Users can't create more powerful roles
- **Compliance:** Enforce organizational boundaries
- **Multi-tenant:** Different teams with different boundaries

### Boundary vs SCP

| Aspect | Permission Boundary | SCP |
|--------|-------------------|-----|
| **Scope** | Individual user/role | Entire account |
| **Applied to** | IAM users and roles | All principals in account |
| **Managed in** | IAM | AWS Organizations |
| **Hierarchy** | No inheritance | Inherits through OU tree |
| **Use case** | Delegated admin | Organization-wide guardrails |

### Exam Tip
**Permission boundaries prevent privilege escalation.** Use for delegated administration. Effective permission = Identity ∩ Boundary (both must allow).

---

## 11. Security Best Practices

### Principle of Least Privilege

```
Start with minimum permissions → Add as needed

Bad Practice:
└── Grant AdministratorAccess to everyone

Good Practice:
├── Grant specific permissions for job function
├── Use managed policies as starting point
├── Review and refine regularly
└── Remove unused permissions
```

### IAM Access Analyzer

#### Overview
Identifies resources shared with external entities and helps refine policies.

#### Features
```
Access Analyzer:
├── External Access Findings
│   ├── S3 buckets accessible outside account
│   ├── IAM roles assumable by external accounts
│   ├── KMS keys shared externally
│   └── Lambda functions with external permissions
│
├── Policy Validation
│   ├── Syntax errors
│   ├── Security warnings
│   └── Best practice suggestions
│
└── Policy Generation (from CloudTrail logs)
    └── Creates least-privilege policy based on actual usage
```

#### Policy Generation Workflow
```
1. User/Role performs actions (logged in CloudTrail)
2. Access Analyzer analyzes CloudTrail logs (90 days)
3. Generates policy with only used permissions
4. Review and refine generated policy
5. Replace overly permissive policy with least-privilege policy
```

### Credential Management

#### Password Policy Best Practices
```
Strong Password Policy:
├── Minimum length: 14+ characters
├── Require: uppercase, lowercase, numbers, symbols
├── Password expiration: 90 days
├── Password reuse: Prevent last 24 passwords
├── Allow users to change own password
└── No hard expiry (allow grace period)
```

#### Access Key Rotation
```
Access Key Lifecycle:
1. Create new access key (Key 2)
2. Update applications to use Key 2
3. Disable Key 1 (not delete yet)
4. Test applications with Key 2
5. Delete Key 1 after confirmed working
6. Repeat every 90 days

Audit:
├── Check last used date (IAM console, Access Advisor)
├── Delete unused keys
└── CloudWatch alarm for old keys (90+ days)
```

### MFA Everywhere

```
MFA Priority:
1. Root user (hardware token recommended)
2. IAM users with console access (required)
3. IAM users with CLI/API access (for sensitive operations)
4. Federated users (enforce at IdP level)

Conditional MFA:
├── Require MFA for sensitive actions
└── Example: Delete production resources
```

**Policy Requiring MFA:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:TerminateInstances",
        "rds:DeleteDBInstance",
        "s3:DeleteBucket"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### Regular Audits

```
Audit Checklist (Quarterly):
├── Review IAM users (delete unused)
├── Review access keys (rotate old keys)
├── Review roles (delete unused, check trust policies)
├── Review policies (remove excessive permissions)
├── Review CloudTrail logs (unusual access patterns)
├── Run Access Analyzer (external access findings)
└── Credential Report (password age, MFA status)
```

#### Credential Report
```bash
# Generate credential report
aws iam generate-credential-report

# Download credential report
aws iam get-credential-report --output text --query Content | base64 -d > report.csv
```

### CloudTrail Integration

```
CloudTrail for IAM:
├── Log all IAM API calls
├── Who assumed which role
├── Policy changes (create, update, delete)
├── Failed authentication attempts
├── Access key usage
└── Federated user activity

Key Events to Monitor:
├── ConsoleLogin (especially root)
├── AssumeRole
├── CreateAccessKey
├── DeleteRole
├── PutRolePolicy
└── UpdateAssumeRolePolicy
```

### Deny Public S3 Buckets (Account-Level)

```bash
# Block public access at account level
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,\
    IgnorePublicAcls=true,\
    BlockPublicPolicy=true,\
    RestrictPublicBuckets=true
```

### Security Checklist

```
Essential IAM Security:
☑ Root user: MFA enabled, no access keys, not used
☑ IAM users: MFA enabled, strong password policy
☑ Access keys: Rotated every 90 days, unused deleted
☑ Roles: Used instead of long-term credentials
☑ Policies: Least privilege, no wildcards (*)
☑ CloudTrail: Enabled in all regions, logs to S3
☑ Access Analyzer: Enabled, findings reviewed
☑ Password policy: 14+ chars, complexity, rotation
☑ SCPs: Organization guardrails in place
☑ Credential report: Generated and reviewed monthly
```

### Exam Tip
**Always use least privilege. Enable MFA on root. Use roles instead of access keys. Enable CloudTrail and Access Analyzer.** These are the most frequently tested security best practices.

---

## 12. Common Patterns

### Pattern: EC2 Instance Accessing S3

```
Components:
├── IAM Role: EC2-S3-Access-Role
├── Trust Policy: Allow ec2.amazonaws.com
├── Permission Policy: S3 read/write
├── Instance Profile: Contains the role
└── EC2 Instance: Attached to instance profile

Trust Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}

Permission Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

### Pattern: Lambda Function with DynamoDB Access

```
Components:
├── IAM Role: Lambda-DDB-Role
├── Trust Policy: Allow lambda.amazonaws.com
├── AWS Managed Policy: AWSLambdaBasicExecutionRole (CloudWatch Logs)
└── Custom Policy: DynamoDB permissions

Trust Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}

Permission Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:Query",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable"
    }
  ]
}
```

### Pattern: Cross-Account S3 Access

```
Account A (Source): 111111111111
Account B (Bucket Owner): 222222222222

Option 1: Resource-Based Policy (Simpler)

Account B - S3 Bucket Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::account-b-bucket/*"
    }
  ]
}

Account A - User/Role Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::account-b-bucket/*"
    }
  ]
}

Option 2: Cross-Account Role

Account B - IAM Role with Trust Policy:
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
      "Action": "sts:AssumeRole"
    }
  ]
}

Account A - User assumes role:
aws sts assume-role --role-arn arn:aws:iam::222222222222:role/S3AccessRole
```

### Pattern: GitHub Actions Deployment

```
1. Create OIDC Provider in AWS
2. Create role with GitHub trust policy
3. Configure GitHub workflow

OIDC Provider:
URL: https://token.actions.githubusercontent.com
Audience: sts.amazonaws.com

Role Trust Policy:
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
          "token.actions.githubusercontent.com:sub": "repo:myorg/myrepo:ref:refs/heads/main"
        }
      }
    }
  ]
}

GitHub Workflow:
- uses: aws-actions/configure-aws-credentials@v2
  with:
    role-to-assume: arn:aws:iam::123456789012:role/GitHubActionsRole
    aws-region: us-east-1
```

### Pattern: Multi-Tenant Application (Tag-Based ABAC)

```
Scenario: Users can only access resources they own

User Policy (Tag-Based):
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject"
      ],
      "Resource": "arn:aws:s3:::shared-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:ExistingObjectTag/Owner": "${aws:username}"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObjectTagging",
      "Resource": "arn:aws:s3:::shared-bucket/*",
      "Condition": {
        "StringEquals": {
          "s3:RequestObjectTag/Owner": "${aws:username}"
        }
      }
    }
  ]
}

Usage:
User alice can only access S3 objects tagged Owner=alice
User bob can only access S3 objects tagged Owner=bob
```

### Exam Tip
**Know these patterns cold: EC2/Lambda service roles, cross-account access (both methods), tag-based ABAC, and GitHub OIDC.** These appear frequently in scenarios.

---

## 13. Troubleshooting

### Common Error Messages

#### 1. Access Denied (403)
```
Error: User: arn:aws:iam::123456789012:user/Alice is not authorized 
to perform: s3:GetObject on resource: arn:aws:s3:::mybucket/file.txt

Troubleshooting Checklist:
☑ Identity policy allows the action?
☑ Resource policy allows the principal?
☑ No explicit Deny in any policy?
☑ SCP allows the action (if in Organization)?
☑ Permission boundary allows (if set)?
☑ Cross-account: Both sides allow?
☑ Correct resource ARN in policy?
☑ Condition keys satisfied (IP, MFA, time)?
```

#### 2. Invalid Principal in Policy
```
Error: Invalid principal in policy

Common Causes:
├── Principal ARN doesn't exist
├── Principal in different partition (aws vs aws-cn)
├── Service principal misspelled
└── Cross-account principal not yet created

Solution:
├── Verify ARN exists
├── Use account root ARN for cross-account: "AWS": "123456789012"
└── Check service principal naming: "Service": "lambda.amazonaws.com"
```

#### 3. Cannot Assume Role
```
Error: User is not authorized to perform: sts:AssumeRole

Troubleshooting:
☑ User has sts:AssumeRole permission?
☑ Role trust policy allows the user?
☑ External ID matches (if required)?
☑ Role exists in target account?
☑ Conditions in trust policy satisfied?
☑ Session duration within limits?
```

#### 4. Policy Too Large
```
Error: Policy size exceeds maximum

Limits:
├── Inline policy: 2,048 characters
├── Managed policy: 6,144 characters
└── Role trust policy: 2,048 characters

Solutions:
├── Use managed policies instead of inline
├── Consolidate similar statements
├── Use wildcards (carefully)
├── Reference resources by prefix
└── Split into multiple policies
```

### Debugging Tools

#### IAM Policy Simulator
```
Use Cases:
├── Test policies before applying
├── Debug why access is denied
├── Validate policy changes
└── Understand effective permissions

Access:
https://policysim.aws.amazon.com/

Test:
1. Select user/role/group
2. Choose service and actions
3. Specify resources
4. Run simulation
5. View allowed/denied results
```

#### Access Advisor
```
Shows:
├── Services accessed by user/role
├── Last accessed timestamp
├── Unused permissions (candidates for removal)
└── Helps achieve least privilege

Location: IAM Console → Users/Roles → Access Advisor tab
```

#### CloudTrail Event History
```
View Recent API Calls:
├── Who made the call (principal)
├── What action was attempted
├── When it occurred
├── Result (success or failure)
├── Source IP
└── User agent

Location: CloudTrail Console → Event history
Filter by: User name, Event name, Resource name
```

#### AWS CLI Debugging
```bash
# Enable debug output
aws s3 ls --debug

# Use --dry-run for some services
aws ec2 run-instances --dry-run ...

# Check caller identity
aws sts get-caller-identity

# Decode authorization failure message
aws sts decode-authorization-message --encoded-message <message>
```

### Policy Evaluation Logic Debugging

```
Step-by-Step Evaluation:
1. Authenticate → Is request from valid principal?
2. Check context → Region, IP, time, MFA, tags
3. Retrieve policies → All applicable policies
4. SCP evaluation → Does SCP allow? (if applicable)
5. Permission boundary → Does boundary allow? (if set)
6. Identity policy → Does identity policy allow?
7. Resource policy → Does resource policy allow?
8. Explicit Deny check → Any explicit deny?
9. Final decision → Allow or Deny

Debug Questions:
├── Which policies apply to this principal?
├── Is there an explicit deny anywhere?
├── Are conditions satisfied?
├── Is this cross-account (need both sides)?
└── Is this in an Organization (check SCP)?
```

### Common Mistakes

```
Mistake 1: Forgetting Resource ARN format
Wrong: "Resource": "mybucket"
Right: "Resource": "arn:aws:s3:::mybucket/*"

Mistake 2: Not distinguishing bucket vs objects
Bucket operations: "arn:aws:s3:::mybucket"
Object operations: "arn:aws:s3:::mybucket/*"

Mistake 3: Assuming roles without trust policy
Need BOTH:
├── User policy: Allow sts:AssumeRole
└── Role trust policy: Allow the user

Mistake 4: Wildcard in Principal
Wrong: "Principal": {"AWS": "*"} (public access)
Right: "Principal": {"AWS": "arn:aws:iam::123456789012:root"}

Mistake 5: Using groups in resource policies
Wrong: "Principal": {"AWS": "arn:aws:iam::123:group/Developers"}
Right: Use user ARNs or account ARN, not group ARNs

Mistake 6: Forgetting cross-account needs both sides
Cross-account requires:
├── Identity policy in source account → Allow
└── Resource policy in target account → Allow
```

### Exam Tip
**For access denied errors, check in order: authentication, identity policy, resource policy, explicit denies, SCPs, permission boundaries.** Cross-account requires both sides to allow.

---

## 14. Policy Examples

### Example 1: S3 Full Access to Specific Bucket

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ]
    }
  ]
}
```

### Example 2: Read-Only Access to EC2

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:Describe*",
        "ec2:Get*",
        "ec2:List*"
      ],
      "Resource": "*"
    }
  ]
}
```

### Example 3: DynamoDB Access with Conditions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:BatchGetItem",
        "dynamodb:Query",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable",
      "Condition": {
        "ForAllValues:StringEquals": {
          "dynamodb:LeadingKeys": ["${aws:username}"]
        }
      }
    }
  ]
}
```

### Example 4: Enforce MFA for Sensitive Operations

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*"
    },
    {
      "Effect": "Deny",
      "Action": [
        "ec2:StopInstances",
        "ec2:TerminateInstances"
      ],
      "Resource": "*",
      "Condition": {
        "BoolIfExists": {
          "aws:MultiFactorAuthPresent": "false"
        }
      }
    }
  ]
}
```

### Example 5: IP Restriction

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.0/24"
          ]
        }
      }
    },
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "NotIpAddress": {
          "aws:SourceIp": [
            "203.0.113.0/24",
            "198.51.100.0/24"
          ]
        }
      }
    }
  ]
}
```

### Example 6: Time-Based Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ec2:*",
      "Resource": "*",
      "Condition": {
        "DateGreaterThan": {
          "aws:CurrentTime": "2024-01-01T09:00:00Z"
        },
        "DateLessThan": {
          "aws:CurrentTime": "2024-01-01T17:00:00Z"
        }
      }
    }
  ]
}
```

### Example 7: Tag-Based Access Control (ABAC)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Owner": "${aws:username}",
          "ec2:ResourceTag/Environment": "Development"
        }
      }
    }
  ]
}
```

### Example 8: Deny All Except Specific Regions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-east-1",
            "us-west-2"
          ]
        },
        "ArnNotLike": {
          "aws:PrincipalArn": [
            "arn:aws:iam::*:role/AdminRole"
          ]
        }
      }
    }
  ]
}
```

### Example 9: Require HTTPS/TLS

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Example 10: Prevent Privilege Escalation

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:CreateRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::123456789012:policy/DeveloperBoundary"
        }
      }
    }
  ]
}
```

---

## Quick Reference Card

### IAM Entity Cheat Sheet

| Entity | Auth Method | Use Case | Max Count |
|--------|-------------|----------|-----------|
| **Root User** | Email + Password + MFA | Account creation, billing | 1 per account |
| **IAM User** | Username + Password/Key | Individual person/app | 5,000 per account |
| **IAM Group** | N/A (not an identity) | Permission management | 300 per account |
| **IAM Role** | Temporary credentials | Services, cross-account, federation | 1,000 per account |
| **Federated User** | External IdP | SSO from corporate directory | Unlimited |

### Policy Type Quick Reference

| Policy Type | Where | Who Defines | Effect |
|-------------|-------|-------------|--------|
| **Identity-Based** | User/Role/Group | Admin | What identity can do |
| **Resource-Based** | S3/SQS/Lambda | Resource owner | Who can access resource |
| **Trust Policy** | IAM Role | Admin | Who can assume role |
| **SCP** | Organization/OU | Org admin | Max permissions for accounts |
| **Permission Boundary** | User/Role | Admin | Max permissions for identity |
| **Session Policy** | AssumeRole call | Caller | Temporary restriction |

### Permission Evaluation

```
Decision Flow:
1. Implicit Deny (default)
2. Evaluate for Allow:
   - SCP allows? (if applicable)
   - Permission Boundary allows? (if set)
   - Identity policy allows? OR Resource policy allows?
3. Check Explicit Deny (overrides all allows)
4. Final: Allow or Deny

Remember: Deny always wins!
```

### Common Condition Keys

| Key | Purpose | Example |
|-----|---------|---------|
| `aws:SourceIp` | IP address | "aws:SourceIp": "203.0.113.0/24" |
| `aws:SourceVpc` | VPC | "aws:SourceVpc": "vpc-12345" |
| `aws:RequestedRegion` | Region | "aws:RequestedRegion": "us-east-1" |
| `aws:MultiFactorAuthPresent` | MFA used | "aws:MultiFactorAuthPresent": "true" |
| `aws:SecureTransport` | HTTPS | "aws:SecureTransport": "true" |
| `aws:PrincipalTag/Key` | Principal tag | "aws:PrincipalTag/Dept": "Eng" |
| `aws:CurrentTime` | Request time | Date comparison |
| `aws:PrincipalOrgID` | Organization | "aws:PrincipalOrgID": "o-123" |

### ARN Format

```
arn:partition:service:region:account-id:resource-type/resource-id

Examples:
├── User: arn:aws:iam::123456789012:user/Alice
├── Role: arn:aws:iam::123456789012:role/MyRole
├── S3 Bucket: arn:aws:s3:::mybucket
├── S3 Object: arn:aws:s3:::mybucket/file.txt
├── EC2: arn:aws:ec2:us-east-1:123456789012:instance/i-1234567890abcdef0
└── Lambda: arn:aws:lambda:us-east-1:123456789012:function:MyFunction
```

---

## Exam Tips Summary

### Must-Know Concepts
1. **Policy evaluation logic:** Explicit deny always wins, cross-account needs both sides
2. **Role assumption:** Requires trust policy + sts:AssumeRole permission
3. **SCPs:** Set maximum permissions, management account exempt
4. **Permission boundaries:** Prevent privilege escalation
5. **Federation:** SAML for enterprise, OIDC for apps/CI-CD
6. **Cross-account:** Role assumption vs resource-based policy
7. **Service roles:** EC2 (instance profile), Lambda (execution role)
8. **External ID:** Prevents confused deputy attacks
9. **IAM is global:** Not region-specific
10. **Temporary credentials:** Start with ASIA, long-term with AKIA

### Common Exam Scenarios
- **EC2 needs S3 access:** Use IAM role with instance profile
- **Cross-account access:** Resource policy (S3) or role assumption
- **Restrict by IP/region:** Use condition keys in policies
- **Federated access:** SAML for employees, OIDC for apps
- **Prevent privilege escalation:** Permission boundaries
- **Multi-account guardrails:** Service Control Policies (SCPs)
- **External partner access:** Cross-account role with external ID
- **Temporary elevated access:** sts:AssumeRole with MFA condition
- **Tag-based access:** ABAC with condition keys on resource tags
- **GitHub Actions deployment:** OIDC provider with trust policy

### Red Flags in Exam Questions
- ❌ **Access keys in code** → Use roles instead
- ❌ **Root user for daily tasks** → Create IAM admin user
- ❌ **Wildcard permissions (*)** → Use least privilege
- ❌ **No MFA on root** → Always enable MFA
- ❌ **Public S3 bucket** → Block public access unless required
- ❌ **Long-term credentials** → Use temporary credentials (roles)
- ❌ **Groups in resource policies** → Use users, roles, or accounts
- ❌ **Hardcoded credentials** → Use instance profiles or Secrets Manager

---

**End of AWS IAM Cheat Sheet**

*For AWS SAA-C03 Certification Preparation*
