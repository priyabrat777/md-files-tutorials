# AWS Encryption & Decryption Cheatsheet - SAA-C03

## Table of Contents
- [Encryption Fundamentals](#encryption-fundamentals)
- [AWS Key Management Service (KMS)](#aws-key-management-service-kms)
- [AWS CloudHSM](#aws-cloudhsm)
- [AWS Certificate Manager (ACM)](#aws-certificate-manager-acm)
- [Service-Specific Encryption](#service-specific-encryption)
- [Encryption Best Practices](#encryption-best-practices)
- [Exam Tips](#exam-tips)

---

## Encryption Fundamentals

### Encryption Types
- **Encryption at Rest**: Data encrypted when stored on disk
- **Encryption in Transit**: Data encrypted while moving between systems (TLS/SSL)
- **Client-Side Encryption**: Data encrypted before sending to AWS
- **Server-Side Encryption**: AWS encrypts data after receiving it

### Key Concepts
- **Symmetric Encryption**: Same key for encryption and decryption (faster, AES-256)
- **Asymmetric Encryption**: Public key encrypts, private key decrypts (RSA)
- **Envelope Encryption**: Data key encrypts data, master key encrypts data key
- **Key Rotation**: Automatic or manual rotation of encryption keys

---

## AWS Key Management Service (KMS)

### KMS Key Types

#### AWS Managed Keys (aws/service-name)
- ✅ Automatically created by AWS services
- ✅ Free to use
- ✅ Automatic rotation every 1 year (cannot disable)
- ❌ Cannot view, manage, or use directly
- ❌ Cannot change key policies

#### Customer Managed Keys (CMK)
- ✅ Full control over key policies and usage
- ✅ Can enable/disable keys
- ✅ Automatic rotation optional (every 1 year)
- ✅ Can import your own key material
- ✅ Can create multi-region keys
- 💰 $1/month per key + API call charges

#### AWS Owned Keys
- ✅ Used by AWS services internally
- ✅ Free
- ❌ Not visible in your account
- ❌ No control over these keys

### KMS Key Features

**Key Policies**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "Enable IAM User Permissions",
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::123456789012:root"
    },
    "Action": "kms:*",
    "Resource": "*"
  }]
}
```

**Multi-Region Keys**
- Primary key in one region
- Replica keys in other regions
- Same key ID and key material
- Useful for disaster recovery and global applications
- Each replica can be managed independently

**Key Rotation**
- Automatic: AWS rotates key material every year (CMK only)
- Manual: Create new key and update applications
- Old key material retained for decryption
- Rotation doesn't change Key ID

**Key States**
- `Enabled`: Available for use
- `Disabled`: Not available, but can be re-enabled
- `Pending Deletion`: 7-30 day waiting period
- `Pending Import`: Awaiting key material import

### KMS Encryption Operations

**API Calls**
- `Encrypt`: Encrypt up to 4 KB of data
- `Decrypt`: Decrypt ciphertext
- `GenerateDataKey`: Create data key for envelope encryption
- `GenerateDataKeyWithoutPlaintext`: Returns only encrypted data key
- `ReEncrypt`: Decrypt and re-encrypt with different key (never exposed)

**Request Quotas**
- Shared quota: 5,500-30,000 requests/second (varies by region)
- Can request quota increase
- Use data key caching to reduce API calls

### KMS Grants
- Alternative to key policies
- Temporary, granular permissions
- Programmatically grant access
- Can be revoked
- Use cases: Cross-account access, temporary access

---

## AWS CloudHSM

### Overview
- Hardware Security Module (HSM) in AWS cloud
- Dedicated hardware for cryptographic operations
- FIPS 140-2 Level 3 compliance (KMS is Level 2)
- Single-tenant, runs in your VPC
- You manage keys, AWS manages hardware

### CloudHSM vs KMS

| Feature | KMS | CloudHSM |
|---------|-----|----------|
| **Tenancy** | Multi-tenant | Single-tenant |
| **FIPS Level** | Level 2/3 | Level 3 |
| **Key Control** | Shared | Customer exclusive |
| **Management** | AWS managed | Customer managed |
| **Pricing** | Pay per key/API call | Pay per HSM hour |
| **Use Cases** | Most applications | Regulatory compliance |
| **Integration** | Native AWS integration | Custom integration |

### CloudHSM Cluster
- Spread across multiple AZs
- Minimum 2 HSMs recommended for HA
- Automatic synchronization
- Access via industry-standard APIs (PKCS#11, JCE, CNG)

### Use Cases
- Regulatory requirements (FIPS 140-2 Level 3)
- Contractual obligations for key control
- Custom cryptographic operations
- Offload SSL/TLS processing
- Oracle TDE, SQL Server encryption

---

## AWS Certificate Manager (ACM)

### Features
- Provision, manage, and deploy SSL/TLS certificates
- **Free public certificates** for AWS services
- Automatic renewal for public certificates
- Private certificates via Private CA (paid)

### Certificate Types

**Public Certificates**
- Domain validation (DV) only
- Free of charge
- Auto-renewal
- Valid for 13 months
- Covers domain and subdomains (wildcard)

**Private Certificates**
- For internal applications
- Requires ACM Private CA
- Custom validity period
- Pay per certificate

### Integration
- ✅ Elastic Load Balancer (ALB, NLB, CLB)
- ✅ CloudFront
- ✅ API Gateway
- ✅ Elastic Beanstalk
- ❌ EC2 instances (cannot export public certs)
- ❌ Route 53 (DNS validation only)

### Certificate Renewal
- **Public**: Automatic renewal before expiration
- **Private**: Manual or automatic via SDK/CLI
- Uses DNS or email validation
- Must maintain CNAME records for DNS validation

### Importing Certificates
- Import third-party certificates
- Must manually renew before expiration
- Can export imported certificates
- Use CloudWatch to monitor expiration

---

## Service-Specific Encryption

### Amazon S3

**Server-Side Encryption (SSE)**

1. **SSE-S3** (AES-256)
   - AWS manages keys
   - Free
   - Default encryption option
   - Header: `x-amz-server-side-encryption: AES256`

2. **SSE-KMS**
   - KMS manages keys
   - Audit trail via CloudTrail
   - KMS API call costs
   - Request quota limits apply
   - Header: `x-amz-server-side-encryption: aws:kms`
   - Can specify Key ID

3. **SSE-C** (Customer-Provided Keys)
   - Customer manages keys outside AWS
   - AWS performs encryption/decryption
   - Must provide key with every request
   - HTTPS required
   - Key not stored by AWS

**Client-Side Encryption**
- Encrypt before upload
- Decrypt after download
- Use AWS Encryption SDK or KMS
- Full control over encryption process

**S3 Bucket Keys**
- Reduce KMS API calls by up to 99%
- S3 uses bucket-level key from KMS
- Generates data keys in S3
- Lower encryption cost

**Encryption in Transit**
- HTTPS endpoints available (recommended)
- HTTP available but not encrypted
- Enforce via bucket policy: `aws:SecureTransport`

### Amazon EBS

**EBS Encryption**
- Encrypt EBS volumes at rest
- Uses KMS keys (AWS or customer managed)
- Data, snapshots, and volumes from snapshots encrypted
- Minimal performance impact
- Cannot encrypt root volume during launch (use AMI or copy)

**Encryption Process**
- Enabled at volume creation
- Cannot enable/disable after creation
- Can copy unencrypted to encrypted snapshot
- Can enable encryption by default (regional setting)

**Snapshots**
- Encrypted volume → encrypted snapshot
- Unencrypted volume → can copy to encrypted snapshot
- Cannot share encrypted snapshots publicly
- Can share encrypted snapshots with other accounts (must share KMS key)

### Amazon RDS

**RDS Encryption at Rest**
- Uses AWS KMS
- Encrypt DB instance and storage
- Automated backups, snapshots, read replicas encrypted
- Cannot enable on existing unencrypted DB
- Must encrypt at creation or restore from encrypted snapshot

**Transparent Data Encryption (TDE)**
- Oracle and SQL Server native encryption
- Can use CloudHSM for key storage
- Different from RDS encryption

**Encryption in Transit**
- Use SSL/TLS connections
- Force SSL via parameter groups:
  - PostgreSQL: `rds.force_ssl=1`
  - MySQL: `REQUIRE SSL` in grants
  - SQL Server: Force encryption

### Amazon DynamoDB

**Encryption at Rest**
- Always encrypted (mandatory)
- Options:
  - AWS owned key (default, free)
  - AWS managed key (`aws/dynamodb`)
  - Customer managed key (CMK)
- Includes table, indexes, streams, backups

**Encryption in Transit**
- HTTPS endpoints
- TLS for all API calls

### Amazon EFS

**Encryption at Rest**
- Uses KMS keys
- Enabled at file system creation
- Cannot enable after creation
- No performance impact

**Encryption in Transit**
- Use EFS mount helper with TLS option
- Enables TLS 1.2 encryption

### Amazon Redshift

**Encryption at Rest**
- KMS or CloudHSM
- 4-tier key hierarchy
- Enabled at cluster launch
- Cannot enable after creation

**Encryption in Transit**
- SSL connections
- Force SSL via parameter group

### AWS Lambda

**Environment Variables**
- Encrypted at rest using KMS
- Can use customer managed keys
- Decrypted at function invocation

### Amazon SQS

**Server-Side Encryption (SSE)**
- Uses KMS
- Encrypts message body (not metadata)
- AWS managed key or customer managed key
- No additional charges for SSE

### Amazon SNS

**Server-Side Encryption**
- Uses KMS
- Encrypts message body and attributes
- AWS managed or customer managed keys

### AWS Systems Manager Parameter Store

**Encryption**
- Standard parameters: optional encryption
- Advanced parameters: optional encryption
- Uses KMS for encryption
- SecureString parameter type for sensitive data

### AWS Secrets Manager

**Encryption**
- Always encrypted at rest using KMS
- Automatic rotation support
- Can use AWS managed or customer managed keys

---

## Cross-Region & Cross-Account Encryption Scenarios

### Cross-Region Encryption

#### S3 Cross-Region Replication (CRR) with Encryption

**Scenario 1: Unencrypted → Encrypted**
- Source: Unencrypted bucket
- Destination: Encrypted bucket
- ✅ Supported - objects encrypted at destination
- Configure encryption in replication rule

**Scenario 2: SSE-S3 → SSE-S3**
- ✅ Automatic encryption replication
- No additional configuration needed

**Scenario 3: SSE-KMS → SSE-KMS**
- ✅ Supported with additional configuration
- Must grant replication role permission to:
  - Decrypt from source KMS key
  - Encrypt with destination KMS key
- Can use different KMS keys in each region
- Can use multi-region KMS keys (same Key ID)

**Scenario 4: SSE-KMS → SSE-S3**
- ✅ Supported - change encryption type during replication

**Replication Configuration for Encrypted Objects**
```json
{
  "Role": "arn:aws:iam::account:role/replication-role",
  "Rules": [{
    "Status": "Enabled",
    "Priority": 1,
    "Filter": {},
    "Destination": {
      "Bucket": "arn:aws:s3:::destination-bucket",
      "ReplicationTime": {
        "Status": "Enabled",
        "Time": { "Minutes": 15 }
      },
      "EncryptionConfiguration": {
        "ReplicaKmsKeyID": "arn:aws:kms:dest-region:account:key/key-id"
      }
    },
    "SourceSelectionCriteria": {
      "SseKmsEncryptedObjects": {
        "Status": "Enabled"
      }
    }
  }]
}
```

**IAM Role Permissions for KMS Replication**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:source-region:account:key/source-key-id"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt"
      ],
      "Resource": "arn:aws:kms:dest-region:account:key/dest-key-id"
    }
  ]
}
```

#### RDS Cross-Region Read Replicas

**Encrypted Source Database**
- Read replica automatically encrypted
- Can use different KMS key in destination region
- Cannot create unencrypted replica from encrypted source
- Encryption must be same or compatible

**Creating Encrypted Cross-Region Replica**
```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier myreadreplicadb \
  --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:mydb \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/replica-key-id \
  --region us-west-2
```

#### EBS Snapshot Copy Cross-Region

**Scenario 1: Unencrypted → Encrypted**
```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-066877671789bd71b \
  --destination-region us-west-2 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/dest-key-id
```

**Scenario 2: Encrypted → Encrypted (Different Key)**
```bash
aws ec2 copy-snapshot \
  --source-region us-east-1 \
  --source-snapshot-id snap-066877671789bd71b \
  --destination-region us-west-2 \
  --encrypted \
  --kms-key-id arn:aws:kms:us-west-2:123456789012:key/dest-key-id
```

**Key Points**
- Can change KMS key during copy
- Can encrypt unencrypted snapshot during copy
- Original snapshot unchanged
- Must have permission for both KMS keys

#### Multi-Region KMS Keys

**Benefits**
- Same Key ID across regions
- Interoperable encrypted data
- Simplified cross-region operations
- Independent key policies per region
- Useful for disaster recovery

**Primary and Replica Keys**
- Create primary key in one region
- Replicate to other regions
- Each replica can be promoted to primary
- Same key material, different ARNs

**Use Cases**
- Global applications
- Disaster recovery
- Client-side encryption with cross-region access
- DynamoDB Global Tables encryption
- Aurora Global Database encryption

**Creating Multi-Region Key**
```bash
# Create primary key
aws kms create-key --multi-region true --description "Multi-region key"

# Replicate to another region
aws kms replicate-key \
  --key-id mrk-xxxxx \
  --replica-region us-west-2
```

**Important Considerations**
- Slightly higher cost than single-region keys
- Each replica managed independently
- Key policies set per region
- Cannot convert existing keys to multi-region
- Limited to specific regions

### Cross-Account Encryption

#### Sharing KMS Encrypted Resources

**Required Steps**
1. Grant cross-account access in KMS key policy (source account)
2. Grant IAM permissions to use KMS key (destination account)
3. Share the encrypted resource
4. Destination account must explicitly accept permissions

#### S3 Cross-Account Access with KMS

**Source Account - KMS Key Policy**
```json
{
  "Sid": "Allow use of the key for cross-account",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::DESTINATION-ACCOUNT-ID:root"
  },
  "Action": [
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

**Source Account - S3 Bucket Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::DESTINATION-ACCOUNT-ID:role/CrossAccountRole"
    },
    "Action": [
      "s3:GetObject",
      "s3:ListBucket"
    ],
    "Resource": [
      "arn:aws:s3:::source-bucket",
      "arn:aws:s3:::source-bucket/*"
    ]
  }]
}
```

**Destination Account - IAM Role Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::source-bucket",
        "arn:aws:s3:::source-bucket/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": "arn:aws:kms:region:SOURCE-ACCOUNT-ID:key/key-id"
    }
  ]
}
```

#### EBS Snapshot Cross-Account Sharing

**Scenario: Share Encrypted Snapshot**

1. **Modify KMS key policy** (source account)
```json
{
  "Sid": "Allow attachment of persistent resources",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::DESTINATION-ACCOUNT-ID:root"
  },
  "Action": [
    "kms:CreateGrant",
    "kms:Decrypt",
    "kms:DescribeKey"
  ],
  "Resource": "*"
}
```

2. **Share the snapshot**
```bash
aws ec2 modify-snapshot-attribute \
  --snapshot-id snap-1234567890abcdef0 \
  --attribute createVolumePermission \
  --operation-add \
  --user-ids DESTINATION-ACCOUNT-ID
```

3. **Destination account creates volume**
```bash
aws ec2 create-volume \
  --snapshot-id snap-1234567890abcdef0 \
  --availability-zone us-east-1a
```

**Key Points**
- Cannot share encrypted snapshots publicly
- Must share KMS key with destination account
- Destination account needs KMS permissions
- Can copy to own account with different KMS key

#### RDS Snapshot Cross-Account Sharing

**Share Encrypted RDS Snapshot**

1. **Share KMS key** (source account)
   - Add destination account to KMS key policy
   - Grant `kms:CreateGrant` and `kms:DescribeKey`

2. **Share snapshot**
```bash
aws rds modify-db-snapshot-attribute \
  --db-snapshot-identifier mydbsnapshot \
  --attribute-name restore \
  --values-to-add DESTINATION-ACCOUNT-ID
```

3. **Destination account restores**
```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier mynewdb \
  --db-snapshot-identifier arn:aws:rds:region:SOURCE-ACCOUNT:snapshot:mydbsnapshot \
  --kms-key-id arn:aws:kms:region:DESTINATION-ACCOUNT:key/dest-key-id
```

**Notes**
- Can re-encrypt with destination account's KMS key
- Manual snapshot sharing only (not automated)
- Both accounts need proper KMS permissions

#### AMI Cross-Account Sharing

**Share Encrypted AMI**

1. **Source account shares KMS key** for EBS snapshots
2. **Share AMI**
```bash
aws ec2 modify-image-attribute \
  --image-id ami-1234567890abcdef0 \
  --launch-permission "Add=[{UserId=DESTINATION-ACCOUNT-ID}]"
```

3. **Destination account launches instance**
   - Can launch directly with source KMS key
   - Or copy AMI with different KMS key

**Copy AMI to Own Account**
```bash
aws ec2 copy-image \
  --source-image-id ami-1234567890abcdef0 \
  --source-region us-east-1 \
  --name "My-copied-AMI" \
  --encrypted \
  --kms-key-id arn:aws:kms:us-east-1:DESTINATION-ACCOUNT:key/key-id
```

#### DynamoDB Cross-Region Replication (Global Tables)

**Encryption Requirements**
- All replicas must be encrypted
- Can use different KMS keys per region
- Or use multi-region KMS keys (recommended)
- Encryption settings inherited from first table

**With Multi-Region Keys**
```bash
# Create table with multi-region key
aws dynamodb create-table \
  --table-name MyGlobalTable \
  --sse-specification Enabled=true,SSEType=KMS,KMSMasterKeyId=mrk-xxxxx \
  --region us-east-1

# Add replica region
aws dynamodb update-table \
  --table-name MyGlobalTable \
  --replica-updates '[{"Create": {"RegionName": "us-west-2"}}]' \
  --region us-east-1
```

**With Different Keys Per Region**
- More complex key management
- Need to specify key for each replica
- Independent key policies per region

### Cross-Account + Cross-Region Scenarios

#### S3 CRR to Different Account

**Requirements**
1. Replication role in source account
2. Destination bucket policy allowing source account
3. KMS key permissions in both accounts
4. Replication configuration with encryption settings

**Destination Account - Bucket Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::SOURCE-ACCOUNT-ID:role/replication-role"
    },
    "Action": [
      "s3:ReplicateObject",
      "s3:ReplicateDelete",
      "s3:ReplicateTags",
      "s3:GetObjectVersionTagging"
    ],
    "Resource": "arn:aws:s3:::destination-bucket/*"
  }]
}
```

**Destination Account - KMS Key Policy**
```json
{
  "Sid": "Allow source account to encrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::SOURCE-ACCOUNT-ID:role/replication-role"
  },
  "Action": [
    "kms:Encrypt",
    "kms:GenerateDataKey"
  ],
  "Resource": "*"
}
```

**Key Points**
- Change ownership of replicated objects
- Destination bucket owner pays storage
- Can change storage class during replication
- Supports encryption type changes

#### Disaster Recovery Patterns

**Pattern 1: Pilot Light with Encrypted Backups**
- Encrypted snapshots in primary region
- Copy encrypted snapshots to DR region
- Use same or different KMS keys
- Multi-region keys simplify management

**Pattern 2: Warm Standby with RDS**
- Encrypted RDS primary database
- Encrypted cross-region read replica
- Promote replica during failover
- Each region uses own KMS key or multi-region key

**Pattern 3: Multi-Region Active-Active**
- DynamoDB Global Tables with multi-region KMS keys
- S3 CRR with encryption
- Aurora Global Database with encryption
- CloudFront with ACM certificates (automatic multi-region)

### Common Cross-Region/Account Exam Scenarios

**Scenario 1**: Copy encrypted EBS snapshot to another region
- **Answer**: Use `copy-snapshot` with `--encrypted` and specify destination KMS key

**Scenario 2**: Share encrypted EBS snapshot with another account
- **Answer**: Share KMS key policy + share snapshot + destination needs KMS permissions

**Scenario 3**: Replicate encrypted S3 objects to another region
- **Answer**: Enable CRR + configure encryption + grant KMS permissions to replication role

**Scenario 4**: RDS read replica in another region with different KMS key
- **Answer**: Create read replica and specify destination region KMS key

**Scenario 5**: Global application with encrypted DynamoDB
- **Answer**: Use DynamoDB Global Tables with multi-region KMS keys

**Scenario 6**: Share encrypted RDS snapshot cross-account
- **Answer**: Share KMS key + share snapshot + destination restores with own KMS key

**Scenario 7**: Copy encrypted AMI to another account and region
- **Answer**: Destination account copies AMI with `--encrypted` flag and own KMS key

**Scenario 8**: Disaster recovery for encrypted resources
- **Answer**: Multi-region KMS keys + automated snapshot copying + replication

**Scenario 9**: Another account needs to read encrypted S3 objects
- **Answer**: KMS key policy + S3 bucket policy + destination IAM policy (all three required)

**Scenario 10**: Minimize cross-region encryption latency
- **Answer**: Use multi-region KMS keys (same Key ID, local decryption)

### Encryption Replication Matrix

| Service | Cross-Region | Cross-Account | Encryption Support |
|---------|--------------|---------------|-------------------|
| **S3 CRR** | ✅ Yes | ✅ Yes | SSE-S3, SSE-KMS, can change type |
| **EBS Snapshot** | ✅ Copy | ✅ Share then copy | Can re-encrypt with different key |
| **RDS Snapshot** | ✅ Copy | ✅ Share then restore | Can re-encrypt with different key |
| **RDS Read Replica** | ✅ Yes | ❌ Same account only | Automatically encrypted |
| **Aurora Global** | ✅ Yes | ❌ Same account only | All regions encrypted |
| **DynamoDB Global** | ✅ Yes | ❌ Same account only | Multi-region or per-region keys |
| **AMI** | ✅ Copy | ✅ Share then copy | Can re-encrypt with different key |
| **Secrets Manager** | ✅ Replicate | ❌ Same account only | Encrypted in all regions |

## Encryption Best Practices

### General Best Practices
1. **Enable encryption by default** for all services
2. **Use customer managed keys** for sensitive data and audit requirements
3. **Enable automatic key rotation** for customer managed keys
4. **Use envelope encryption** for large data
5. **Implement least privilege** for key access
6. **Monitor key usage** via CloudTrail and CloudWatch
7. **Enable encryption in transit** (TLS/SSL)
8. **Use separate keys** for different applications/environments
9. **Document key policies** and access patterns
10. **Test disaster recovery** procedures

### Key Management
- Use descriptive key aliases
- Tag keys for cost allocation and organization
- Set up CloudWatch alarms for unusual activity
- Regular audit key policies and grants
- Use key policy conditions for additional security
- Implement key deletion waiting period (7-30 days)

### Cross-Region Considerations
- Use multi-region KMS keys for disaster recovery
- Understand data residency requirements
- Plan for regional service failures
- Consider latency for cross-region encryption/decryption

### Compliance
- Understand your compliance requirements (HIPAA, PCI-DSS, GDPR)
- Use CloudHSM for FIPS 140-2 Level 3 requirements
- Document encryption architecture
- Regular compliance audits
- Data classification and encryption mapping

---

## Exam Tips

### Common Scenarios

**Scenario 1**: Need encryption with full control over keys
- **Answer**: Customer Managed Keys (CMK) in KMS

**Scenario 2**: Regulatory requirement for FIPS 140-2 Level 3
- **Answer**: AWS CloudHSM

**Scenario 3**: Free SSL/TLS certificates for load balancer
- **Answer**: AWS Certificate Manager (ACM)

**Scenario 4**: Encrypt existing unencrypted EBS volume
- **Answer**: Create snapshot → copy to encrypted snapshot → create volume

**Scenario 5**: Encrypt existing RDS database
- **Answer**: Create encrypted snapshot → restore to new encrypted DB instance

**Scenario 6**: Share encrypted EBS snapshot cross-account
- **Answer**: Share snapshot + share KMS key with target account (KMS key policy + snapshot permissions)

**Scenario 7**: Reduce KMS API costs for S3
- **Answer**: Enable S3 Bucket Keys

**Scenario 8**: Audit who used KMS keys and when
- **Answer**: AWS CloudTrail (KMS API calls logged)

**Scenario 9**: Encrypt data before sending to AWS
- **Answer**: Client-side encryption using AWS Encryption SDK

**Scenario 10**: Automatic certificate renewal
- **Answer**: ACM with DNS validation (CNAME records)

**Scenario 11**: Copy encrypted EBS snapshot to another region
- **Answer**: Use `copy-snapshot` with `--encrypted` flag and specify destination region KMS key

**Scenario 12**: Replicate encrypted S3 objects to another region
- **Answer**: Configure S3 CRR + add encryption configuration + grant KMS decrypt/encrypt permissions to replication role

**Scenario 13**: Global application with low-latency encryption
- **Answer**: Multi-region KMS keys (same Key ID, local decryption in each region)

**Scenario 14**: Share encrypted RDS snapshot with another AWS account
- **Answer**: Add destination account to KMS key policy + share snapshot + destination restores with own KMS key

**Scenario 15**: Cross-account access to encrypted S3 objects
- **Answer**: Three-layer permissions - KMS key policy (source) + S3 bucket policy (source) + IAM policy (destination)

**Scenario 16**: Disaster recovery for encrypted RDS database
- **Answer**: Automated encrypted snapshots + copy snapshots to DR region + use multi-region KMS keys or region-specific keys

**Scenario 17**: DynamoDB table replicated globally with encryption
- **Answer**: DynamoDB Global Tables with multi-region KMS keys for simplified management

**Scenario 18**: Another account needs to launch EC2 from your encrypted AMI
- **Answer**: Share KMS key policy + share AMI + destination account can launch or copy with own KMS key

**Scenario 19**: S3 replication to different account and region with encryption
- **Answer**: Destination bucket policy + destination KMS key policy + source replication role with decrypt/encrypt permissions

**Scenario 20**: Encrypted EBS volume needs to be shared (not snapshot)
- **Answer**: Cannot share volumes directly - must create snapshot, share snapshot with KMS key, destination creates volume

### Key Exam Points

#### General Encryption
✅ **KMS is region-specific** (except multi-region keys)
✅ **Default encryption uses AWS managed keys** (free)
✅ **Customer managed keys cost $1/month** + API calls
✅ **Automatic rotation is optional for CMKs** (yearly)
✅ **Cannot enable encryption after resource creation** (EBS, RDS, EFS)
✅ **S3 Bucket Keys reduce KMS costs** by up to 99%
✅ **CloudHSM for single-tenant** and FIPS Level 3
✅ **ACM public certificates are free** and auto-renew
✅ **Envelope encryption for large data** (> 4 KB)
✅ **KMS encrypt API limited to 4 KB** of data
✅ **Always encrypt in transit** (HTTPS/TLS/SSL)
✅ **Key policies + IAM policies** control KMS access
✅ **CloudTrail logs all KMS API calls** for audit
✅ **Multi-region keys** have same Key ID across regions
✅ **Cannot export public ACM certificates**

#### Cross-Region Encryption
✅ **Can copy encrypted snapshots** to different regions with new KMS key
✅ **S3 CRR supports encryption** type changes during replication
✅ **Multi-region KMS keys** simplify cross-region encryption
✅ **RDS read replicas** can use different KMS key per region
✅ **DynamoDB Global Tables** work best with multi-region keys
✅ **Each region's KMS key** is independent (separate policies)
✅ **Can encrypt during copy** (unencrypted → encrypted snapshot)
✅ **Aurora Global Database** requires encryption in all regions
✅ **Secrets Manager** can replicate secrets across regions (encrypted)

#### Cross-Account Encryption
✅ **Three permission layers needed**: KMS key policy + resource policy + IAM policy
✅ **Cannot share encrypted snapshots publicly** (EBS, RDS, AMI)
✅ **KMS key must grant** destination account permissions
✅ **Destination account** must explicitly accept with IAM policy
✅ **Can re-encrypt with own KMS key** when copying resources
✅ **S3 cross-account** requires bucket policy + KMS key policy + IAM
✅ **kms:CreateGrant** often needed for snapshots/volumes
✅ **Automated backups cannot be shared** (manual snapshots only)
✅ **Ownership change** available in S3 cross-account replication

#### Replication & DR
✅ **S3 CRR/SRR** replicates encrypted objects (need KMS permissions)
✅ **Replication role** needs decrypt source + encrypt destination
✅ **Can replicate to different account** and region simultaneously
✅ **DynamoDB Global Tables** automatically replicate with encryption
✅ **RDS automated backups** stay in same region (manual for cross-region)
✅ **Multi-region keys** reduce complexity for DR scenarios
✅ **Pilot light DR**: copy encrypted snapshots to DR region
✅ **EBS snapshots** are incremental but independently encrypted

### Quick Decision Tree

**Need encryption?**
- At rest? → Enable service-specific encryption (KMS)
- In transit? → Use HTTPS/TLS/SSL
- Both? → Enable both

**Which KMS key?**
- Don't care about control? → AWS managed key (free)
- Need control/audit? → Customer managed key ($1/month)
- Regulatory compliance Level 3? → CloudHSM

**SSL/TLS certificates?**
- For AWS services? → ACM (free, auto-renew)
- For EC2/on-premises? → Import to ACM or use third-party

**Cross-region encryption?**
- Active-active app? → Multi-region KMS keys
- Disaster recovery? → Multi-region keys or replicate
- Single region? → Standard KMS keys
- Simple replication? → Copy with new KMS key

**Cross-account sharing?**
- Read-only access? → KMS key policy + resource policy + IAM
- Full copy? → Share + destination copies with own KMS key
- S3 objects? → Bucket policy + KMS key policy + destination IAM
- Snapshots? → KMS key policy (CreateGrant) + share snapshot + destination IAM

**Replication scenarios?**
- S3 → S3? → Enable CRR/SRR + encryption config + KMS permissions
- RDS → RDS? → Cross-region read replica or snapshot copy
- DynamoDB → DynamoDB? → Global Tables with multi-region keys
- EBS → EBS? → Snapshot copy with encryption
- DR strategy? → Automated snapshot copying + multi-region keys

---

## Quick Reference Commands

### KMS AWS CLI Examples

```bash
# Create a KMS key
aws kms create-key --description "My CMK"

# Create an alias
aws kms create-alias --alias-name alias/my-key --target-key-id <key-id>

# Enable key rotation
aws kms enable-key-rotation --key-id <key-id>

# Encrypt data
aws kms encrypt --key-id alias/my-key --plaintext "Hello" --output text --query CiphertextBlob

# Decrypt data
aws kms decrypt --ciphertext-blob fileb://encrypted.txt --output text --query Plaintext

# Generate data key
aws kms generate-data-key --key-id alias/my-key --key-spec AES_256

# List keys
aws kms list-keys

# Describe key
aws kms describe-key --key-id <key-id>

# Schedule key deletion
aws kms schedule-key-deletion --key-id <key-id> --pending-window-in-days 30
```

### S3 Encryption Examples

```bash
# Upload with SSE-S3
aws s3 cp file.txt s3://bucket/ --sse AES256

# Upload with SSE-KMS
aws s3 cp file.txt s3://bucket/ --sse aws:kms --sse-kms-key-id <key-id>

# Set default encryption on bucket
aws s3api put-bucket-encryption --bucket my-bucket --server-side-encryption-configuration '{
  "Rules": [{
    "ApplyServerSideEncryptionByDefault": {
      "SSEAlgorithm": "aws:kms",
      "KMSMasterKeyID": "arn:aws:kms:region:account:key/key-id"
    }
  }]
}'
```

---

## Summary

### Remember for the Exam
1. **Encryption is cheaper than non-compliance** - enable everywhere
2. **KMS for key management** - most common encryption service
3. **CloudHSM for regulatory requirements** - single-tenant HSM
4. **ACM for certificates** - free public certs, auto-renewal
5. **Envelope encryption pattern** - common for large data
6. **Can't encrypt after creation** - plan encryption from start
7. **S3 Bucket Keys** - massive cost savings for S3+KMS
8. **CloudTrail for audit** - all KMS operations logged
9. **In transit + at rest** - comprehensive security
10. **Multi-region keys** - disaster recovery and global apps

### Cost Optimization
- Use AWS managed keys when possible (free)
- Enable S3 Bucket Keys to reduce KMS API calls
- Use data key caching in applications
- Monitor and optimize KMS API usage
- Consider encryption requirements per environment

---

*Last Updated: 2026*
*For AWS Solutions Architect Associate (SAA-C03) Certification*
