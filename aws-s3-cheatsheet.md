# Amazon S3 Cheat Sheet

**Interview preparation: object storage, consistency, security, lifecycle, replication, and recovery.**

**Verified:** 2026-10-13. Unless stated otherwise, examples concern general-purpose buckets; directory/table/vector bucket capabilities are not interchangeable.

[All AWS cheatsheets](aws-core-services-interview-index.md)

## Table of Contents

- [1. Fundamentals and mental model](#1-fundamentals-and-mental-model)
- [2. Current limits and consistency](#2-current-limits-and-consistency)
- [3. Storage-class selection](#3-storage-class-selection)
- [4. Versioning, lifecycle, and Object Lock](#4-versioning-lifecycle-and-object-lock)
- [5. Security and encryption](#5-security-and-encryption)
- [6. Replication and disaster recovery](#6-replication-and-disaster-recovery)
- [7. Performance and event-driven processing](#7-performance-and-event-driven-processing)
- [8. Cost and observability](#8-cost-and-observability)
- [9. Common Patterns](#9-common-patterns)
- [10. Troubleshooting and practical inspection](#10-troubleshooting-and-practical-inspection)
- [11. Interview questions and answers](#11-interview-questions-and-answers)
- [Quick Reference Card](#quick-reference-card)
- [Exam Tips Summary](#exam-tips-summary)
- [12. Quick revision](#12-quick-revision)
- [13. AWS sources](#13-aws-sources)

---

## 1. Fundamentals and mental model

S3 is object storage. An object is addressed by bucket and key, optionally with a version ID. Prefixes organize keys; they are not ordinary filesystem directories.

```text
Client → authorized S3 request → bucket → key → object bytes + metadata
                                      ├── versions / retention
                                      ├── lifecycle / replication
                                      └── notifications / access policies
```

Choose S3 for objects, data lakes, backups, and static assets. Use EBS for attached block storage and EFS for shared filesystem semantics. S3 is not a transactional relational database.

## 2. Current limits and consistency

**Updated maximum object size:** AWS announced **50 TB** objects on **December 2, 2025**. The documentation gives the technical maximum as about **48.8 TiB** (53.7 decimal TB). Old 5 TB interview notes are outdated. [Announcement](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-maximum-object-size-50-tb/), [object overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingObjects.html)

Multipart uploads allow up to 10,000 parts, each normally 5 MiB–5 GiB; the final part has no minimum. Choose part size for both throughput and the part-count ceiling. Abort abandoned uploads to avoid paying for unused parts. [Multipart limits](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)

S3 provides strong read-after-write consistency for successful object writes/deletes and subsequent reads/listing. A single-key update is atomic. This does **not** create atomic transactions across many objects or synchronous cross-Region replication. Concurrent writers still need coordination or conditional requests. [Consistency model](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)

## 3. Storage-class selection

| Class | Good fit | Main tradeoff |
|---|---|---|
| Standard | Frequently accessed general objects | Higher storage rate than archive tiers |
| Intelligent-Tiering | Changing/unknown access patterns | Monitoring fee; optional archive tiers require restore |
| Standard-IA | Infrequent access with immediate reads | Retrieval charges and minimum billing duration/size |
| One Zone-IA | Re-creatable, infrequently accessed data | Single-AZ failure exposure |
| Glacier Instant Retrieval | Rarely accessed but immediate retrieval | Retrieval and minimum-duration costs |
| Glacier Flexible Retrieval | Archives with minutes-to-hours tolerance | Restore before ordinary reads |
| Glacier Deep Archive | Long retention with hours-scale recovery | Slow retrieval and duration constraints |
| Express One Zone | Latency-sensitive object access near compute | Single-zone placement; directory-bucket feature differences |

Ask for retrieval time, frequency, object size, retention, and failure tolerance before choosing. Durability and availability are different measures. [Storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)

## 4. Versioning, lifecycle, and Object Lock

- Versioning preserves prior object versions; a normal delete commonly inserts a delete marker.
- Deleting a specific version is different from deleting the current key view.
- Lifecycle rules can transition or expire current/noncurrent versions and clean incomplete multipart uploads.
- Retained old versions consume storage. Versioning without lifecycle management can grow the bill substantially.
- Object Lock provides WORM retention/legal holds for protected versions.

| Object Lock mechanism | Meaning |
|---|---|
| Governance mode | Privileged callers can bypass under explicit conditions |
| Compliance mode | Protected version cannot be deleted/overwritten during retention, including by root |
| Legal hold | Protection without an automatic end date; authorized removal is separate from retention |

Object Lock protects versions, not a guarantee that no newer version or delete marker can exist. Distinguish retention controls from recovery workflows. [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)

## 5. Security and encryption

New objects are encrypted at rest by default. Choose SSE-S3 for S3-managed encryption or customer managed SSE-KMS when key control, auditing, and access separation are required. Encryption alone does not make a public object private. [Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html)

Use Block Public Access, IAM/bucket/access-point policies, and bucket-owner-enforced Object Ownership where suitable. With bucket-owner-enforced ownership, ACLs are disabled and policy-based access controls apply. [Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)

For SSE-KMS, a caller may need both S3 and KMS permissions. In cross-account designs, include the key policy and caller IAM policy. Evaluate endpoint policies, SCPs/RCPs, explicit denies, and object-version permissions when access fails.

Presigned URLs delegate the signer's allowed operation for a limited time. They can expire earlier when underlying temporary credentials expire. Treat them as bearer access and avoid logging them carelessly.

## 6. Replication and disaster recovery

S3 Same-Region and Cross-Region Replication copy eligible objects asynchronously under configured rules. Versioning and required permissions must be in place. Existing objects may need S3 Batch Replication; enabling a rule is not a universal historical backfill.

Decide what to replicate: prefixes/tags, encryption, ownership, delete markers, and retention. Track failed/pending replication and validate destination access. Replication Time Control offers a defined replication-time objective under its conditions; it is not synchronous replication. [Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)

Replication is not automatically a backup: unwanted changes or deletions may propagate depending on configuration. Combine isolation, retention/versioning, and tested restoration according to the threat and RPO/RTO.

## 7. Performance and event-driven processing

Use multipart transfers, parallelism, connection reuse, and retries with backoff. Measure object size and request pattern before changing the naming scheme. ETags are not universally MD5 checksums, especially for multipart/encrypted objects; use supported checksum features.

Treat object notifications as potentially duplicated or out of order. Make processors idempotent using bucket/key/version or another stable event identity. Avoid writing output back into the same triggering prefix without an explicit loop-prevention design.

For large uploads through an API, issue narrowly scoped presigned access and send the bytes directly to S3; this avoids gateway/function payload bottlenecks. Validate ownership, completion, size, and content before processing.

## 8. Cost and observability

Cost includes stored bytes, requests, retrieval, early deletion/minimum duration, transfer, replication, inventory/analytics, and KMS. A cheaper per-GB class can cost more under frequent reads or many tiny objects.

Use Storage Lens/Inventory for storage analysis, request/error metrics for operational health, replication metrics for DR, and CloudTrail data events where object-level audit is required. Management-event logging alone is not a full object-access audit.

## 9. Common Patterns

Real-world S3 architectures and implementation patterns with complete configurations.

### Pattern 1: Static Website with CloudFront CDN

**Use Case:** Host SPA/static site with global edge caching, SSL, and custom domain.

**Architecture:**
```
Route 53 → CloudFront Distribution → S3 Origin (private)
             ├── Edge caching (TTL)
             ├── SSL/TLS certificate
             ├── Custom domain
             └── Origin Access Identity (OAI) or Origin Access Control (OAC)
```

**S3 Bucket Configuration:**
```bash
# Create bucket (private, not website hosting)
aws s3api create-bucket --bucket my-static-site \
  --region us-east-1

# Block public access (CloudFront will access privately)
aws s3api put-public-access-block --bucket my-static-site \
  --public-access-block-configuration \
  "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

**Bucket Policy for CloudFront OAC:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowCloudFrontServicePrincipal",
      "Effect": "Allow",
      "Principal": {
        "Service": "cloudfront.amazonaws.com"
      },
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-static-site/*",
      "Condition": {
        "StringEquals": {
          "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/E1234567890ABC"
        }
      }
    }
  ]
}
```

**Why This Pattern:**
- S3 static website hosting exposes bucket publicly; CloudFront + OAC keeps bucket private
- Edge caching reduces S3 GET requests and improves global latency
- CloudFront handles SSL/TLS termination
- Cost: Pay for S3 storage + CloudFront data transfer (cheaper than S3 → internet)

---

### Pattern 2: S3 Event → Lambda Image Processing Pipeline

**Use Case:** Auto-resize uploaded images, generate thumbnails, extract metadata.

**Architecture:**
```
User uploads → S3 source bucket → S3 Event Notification → Lambda
                                                            ├── Read original from S3
                                                            ├── Process (resize/thumbnail)
                                                            └── Write to S3 dest bucket

Loop Prevention: Different bucket OR different prefix
```

**S3 Event Configuration:**
```json
{
  "LambdaFunctionConfigurations": [
    {
      "Id": "ImageProcessing",
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:123456789012:function:ProcessImage",
      "Events": ["s3:ObjectCreated:*"],
      "Filter": {
        "Key": {
          "FilterRules": [
            {"Name": "prefix", "Value": "uploads/images/"},
            {"Name": "suffix", "Value": ".jpg"}
          ]
        }
      }
    }
  ]
}
```

**Lambda Execution Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion"
      ],
      "Resource": "arn:aws:s3:::source-bucket/uploads/images/*"
    },
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::processed-bucket/thumbnails/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

**Lambda Resource Policy (S3 Invoke Permission):**
```bash
aws lambda add-permission \
  --function-name ProcessImage \
  --statement-id s3-invoke-permission \
  --action "lambda:InvokeFunction" \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::source-bucket" \
  --source-account 123456789012
```

**Pitfalls to Avoid:**
- ❌ **Infinite loop:** Writing output to same bucket/prefix that triggers function
- ❌ **No idempotency:** Duplicate events cause duplicate processing
- ❌ **Synchronous mindset:** Events are asynchronous, may arrive out of order
- ✅ **Solution:** Use separate destination bucket OR different prefix + version ID tracking

**Cost Optimization:**
- Use S3 Select to filter CSV/JSON before downloading entire object
- Lambda ephemeral storage (/tmp) is 512MB–10GB; batch small images
- Monitor Lambda duration; resize libraries vary in performance

---

### Pattern 3: Cross-Account Replication with Encryption

**Use Case:** Replicate encrypted objects to DR account with different KMS key.

**Architecture:**
```
Account A (source)                          Account B (destination)
├── S3 bucket (SSE-KMS with Key A)    →    ├── S3 bucket (SSE-KMS with Key B)
├── Replication Role                        └── Bucket policy allows replication
└── KMS Key A policy allows role            └── KMS Key B policy allows role
```

**Account A: Source Bucket Replication Configuration:**
```json
{
  "Role": "arn:aws:iam::111111111111:role/S3ReplicationRole",
  "Rules": [
    {
      "Status": "Enabled",
      "Priority": 1,
      "Filter": {"Prefix": ""},
      "Destination": {
        "Bucket": "arn:aws:s3:::destination-bucket",
        "Account": "222222222222",
        "ReplicationTime": {
          "Status": "Enabled",
          "Time": {"Minutes": 15}
        },
        "Metrics": {
          "Status": "Enabled",
          "EventThreshold": {"Minutes": 15}
        },
        "EncryptionConfiguration": {
          "ReplicaKmsKeyID": "arn:aws:kms:us-west-2:222222222222:key/destination-key-id"
        },
        "AccessControlTranslation": {
          "Owner": "Destination"
        }
      },
      "SourceSelectionCriteria": {
        "SseKmsEncryptedObjects": {"Status": "Enabled"}
      },
      "DeleteMarkerReplication": {"Status": "Enabled"}
    }
  ]
}
```

**Account A: Replication Role Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"Service": "s3.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Account A: Replication Role Permissions:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::source-bucket"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::source-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::destination-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:us-east-1:111111111111:key/source-key-id",
      "Condition": {
        "StringLike": {
          "kms:ViaService": "s3.us-east-1.amazonaws.com",
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::source-bucket/*"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "kms:Encrypt"
      ],
      "Resource": "arn:aws:kms:us-west-2:222222222222:key/destination-key-id",
      "Condition": {
        "StringLike": {
          "kms:ViaService": "s3.us-west-2.amazonaws.com",
          "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::destination-bucket/*"
        }
      }
    }
  ]
}
```

**Account B: Destination Bucket Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/S3ReplicationRole"
      },
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::destination-bucket/*"
    },
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::111111111111:role/S3ReplicationRole"
      },
      "Action": [
        "s3:List*",
        "s3:GetBucketVersioning",
        "s3:PutBucketVersioning"
      ],
      "Resource": "arn:aws:s3:::destination-bucket"
    }
  ]
}
```

**Account B: KMS Key Policy (add to existing policy):**
```json
{
  "Sid": "AllowReplicationRoleEncrypt",
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:role/S3ReplicationRole"
  },
  "Action": "kms:Encrypt",
  "Resource": "*",
  "Condition": {
    "StringLike": {
      "kms:ViaService": "s3.us-west-2.amazonaws.com",
      "kms:EncryptionContext:aws:s3:arn": "arn:aws:s3:::destination-bucket/*"
    }
  }
}
```

**Common Replication Failures:**
| Symptom | Root Cause |
|---------|------------|
| Replication pending/failed | Missing KMS permissions in source OR destination |
| `AccessDenied` in metrics | Destination bucket policy missing replication actions |
| Existing objects not replicated | Replication only applies to new writes; use S3 Batch Replication |
| Delete markers not replicated | `DeleteMarkerReplication` not enabled |
| Wrong object owner | Missing `AccessControlTranslation` |

**Exam Tips:**
- Both source AND destination buckets must have versioning enabled
- Replication is asynchronous; Replication Time Control (RTC) provides SLA (15 min)
- Existing objects need S3 Batch Replication job
- Cross-account requires bucket policy + KMS key policy in destination

---

### Pattern 4: Presigned URL for Temporary Upload

**Use Case:** Allow untrusted client to upload directly to S3 without exposing credentials.

**Architecture:**
```
Client requests → API Gateway → Lambda (generates presigned URL) → returns URL
Client uploads → Presigned URL → S3 (validates signature + expiration)
```

**Lambda Code (Python):**
```python
import boto3
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    s3_client = boto3.client('s3', region_name='us-east-1')
    
    # Extract metadata from request
    file_name = event['queryStringParameters']['fileName']
    content_type = event['queryStringParameters'].get('contentType', 'application/octet-stream')
    
    # Generate presigned URL for PUT
    try:
        presigned_url = s3_client.generate_presigned_url(
            'put_object',
            Params={
                'Bucket': 'user-uploads-bucket',
                'Key': f'uploads/{file_name}',
                'ContentType': content_type,
                'ServerSideEncryption': 'AES256',
                'Metadata': {
                    'uploaded-by': event['requestContext']['identity']['sourceIp']
                }
            },
            ExpiresIn=300,  # 5 minutes
            HttpMethod='PUT'
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({
                'uploadUrl': presigned_url,
                'expiresIn': 300
            })
        }
    except ClientError as e:
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }
```

**Lambda Execution Role Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::user-uploads-bucket/uploads/*"
    }
  ]
}
```

**Client Upload (JavaScript):**
```javascript
// Get presigned URL
const response = await fetch(
  'https://api.example.com/getUploadUrl?fileName=document.pdf&contentType=application/pdf'
);
const { uploadUrl } = await response.json();

// Upload directly to S3
const file = document.getElementById('fileInput').files[0];
const uploadResponse = await fetch(uploadUrl, {
  method: 'PUT',
  body: file,
  headers: {
    'Content-Type': 'application/pdf'
  }
});

if (uploadResponse.ok) {
  console.log('Upload successful');
}
```

**Security Considerations:**
- Presigned URL inherits signer's permissions at generation time
- If Lambda role loses PutObject permission before URL expires, upload fails
- Temporary credentials (Lambda role) expire → presigned URL expires early
- URL is bearer token; anyone with URL can upload until expiration
- Use short expiration (5–15 minutes) for uploads
- Consider adding conditions: `x-amz-server-side-encryption`, `Content-Length` range

**Advanced: Presigned POST (multipart form upload):**
```python
# Generate presigned POST with policy
presigned_post = s3_client.generate_presigned_post(
    Bucket='user-uploads-bucket',
    Key='uploads/${filename}',
    Fields={
        'acl': 'private',
        'Content-Type': 'image/jpeg'
    },
    Conditions=[
        {'acl': 'private'},
        {'Content-Type': 'image/jpeg'},
        ['content-length-range', 1024, 10485760]  # 1KB - 10MB
    ],
    ExpiresIn=600
)
```

**Benefits:**
- Client uploads directly to S3 (no proxy through API Gateway/Lambda)
- Avoids Lambda 6MB payload limit
- Reduces data transfer costs through gateway
- Scales to any upload size (multipart if needed)

---

## 10. Troubleshooting and practical inspection

### Common Issues and Root Causes

| Symptom | Root Cause | Investigation | Resolution |
|---------|------------|---------------|------------|
| **`AccessDenied` on GetObject** | Missing s3:GetObject permission | Check identity policy, bucket policy, object ownership, KMS decrypt permission | Add required permissions; verify bucket owner enforced settings |
| **`AccessDenied` with SSE-KMS** | Missing kms:Decrypt permission | Check KMS key policy, caller's IAM policy, ViaService condition | Grant kms:Decrypt on key policy for caller |
| **Presigned URL fails** | URL expired OR underlying credentials expired early | Check expiration timestamp, Lambda/role credential lifetime | Use longer-lived credentials for signer; shorten presigned URL expiration |
| **Object appears deleted** | Delete marker present | List object versions: `list-object-versions` | Delete the delete marker to "undelete" |
| **ListBucket returns 403** | Missing s3:ListBucket permission OR bucket policy denies | Check bucket-level permission (not object/*) | Add s3:ListBucket on bucket ARN (not bucket/*) |
| **Replication not working** | Versioning disabled, missing role permissions, KMS policy | Check replication metrics, CloudWatch Logs, role trust/permissions | Enable versioning; grant role decrypt (source) + encrypt (dest) |
| **Existing objects not replicated** | Replication only applies to new writes after rule enabled | Use S3 Batch Replication for historical objects | Create Batch Replication job |
| **Glacier object unreadable** | Object not restored OR restore expired | Check restore status: `head-object --restore` | Initiate restore with tier (Expedited/Standard/Bulk) |
| **Upload fails with `EntityTooLarge`** | Single PUT exceeds 5GB | Use multipart upload for >100MB objects | Switch to multipart upload API |
| **Multipart upload stalled** | Incomplete multipart upload consuming storage | List multipart uploads: `list-multipart-uploads` | Abort incomplete uploads; add lifecycle rule to auto-abort |
| **Unexpected storage costs** | Old versions accumulating, incomplete uploads | Use S3 Storage Lens, check versioning status | Add lifecycle policy to expire noncurrent versions |
| **Cross-account access fails** | Both identity AND resource policies must allow | Check both account policies, explicit denies | Add bucket policy in resource account + identity policy in caller account |
| **Public access blocked** | Block Public Access enabled (default) | Check account/bucket-level BPA settings | Disable BPA if truly needed (not recommended) |
| **Event notification not triggering** | Lambda resource policy missing, wrong prefix/suffix filter | Check Lambda policy for s3.amazonaws.com principal | Add Lambda resource policy allowing S3 invoke |
| **Slow uploads** | Single-threaded, small TCP window, latency | Use multipart + parallel threads, S3 Transfer Acceleration | Enable Transfer Acceleration; use AWS SDK with multipart |
| **ETag mismatch** | Expecting MD5 but got multipart/KMS ETag | ETags for multipart/encrypted objects are not MD5 | Use x-amz-checksum-* headers (SHA256, CRC32) |
| **MFA Delete required** | Bucket has MFA Delete enabled | User must provide MFA serial + token code | Include --mfa "serial token" in delete-object-version |

### Diagnostic Commands

**Check object existence and metadata:**
```bash
# Get object metadata (doesn't download object)
aws s3api head-object \
  --bucket example-bucket \
  --key invoices/2026/report.pdf

# List all versions (shows delete markers)
aws s3api list-object-versions \
  --bucket example-bucket \
  --prefix invoices/2026/report.pdf
```

**Check bucket configurations:**
```bash
# Versioning status
aws s3api get-bucket-versioning --bucket example-bucket

# Public access block
aws s3api get-public-access-block --bucket example-bucket

# Encryption configuration
aws s3api get-bucket-encryption --bucket example-bucket

# Lifecycle rules
aws s3api get-bucket-lifecycle-configuration --bucket example-bucket

# Replication configuration
aws s3api get-bucket-replication --bucket example-bucket

# CORS configuration
aws s3api get-bucket-cors --bucket example-bucket

# Bucket policy
aws s3api get-bucket-policy --bucket example-bucket

# Object ownership
aws s3api get-bucket-ownership-controls --bucket example-bucket
```

**Check incomplete multipart uploads:**
```bash
# List incomplete uploads (often source of unexpected costs)
aws s3api list-multipart-uploads --bucket example-bucket

# Abort specific upload
aws s3api abort-multipart-upload \
  --bucket example-bucket \
  --key large-file.zip \
  --upload-id "upload-id-from-list"
```

**Check Glacier restore status:**
```bash
# Check restore status
aws s3api head-object \
  --bucket archive-bucket \
  --key archive/file.dat \
  | jq '.Restore'

# Initiate restore (Expedited = 1-5 min, Standard = 3-5 hr, Bulk = 5-12 hr)
aws s3api restore-object \
  --bucket archive-bucket \
  --key archive/file.dat \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}'
```

**Check replication status:**
```bash
# Get object replication status
aws s3api head-object \
  --bucket source-bucket \
  --key data/file.csv \
  | jq '.ReplicationStatus'

# Possible values: PENDING, COMPLETED, FAILED, REPLICA
```

**Check access from specific IP/principal:**
```bash
# Simulate IAM policy evaluation
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/Alice \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns arn:aws:s3:::mybucket/data/*
```

**CloudWatch Metrics:**
```bash
# Get request metrics (4xx/5xx errors)
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name 4xxErrors \
  --dimensions Name=BucketName,Value=example-bucket \
  --start-time 2026-10-13T00:00:00Z \
  --end-time 2026-10-14T00:00:00Z \
  --period 3600 \
  --statistics Sum

# Get replication latency
aws cloudwatch get-metric-statistics \
  --namespace AWS/S3 \
  --metric-name ReplicationLatency \
  --dimensions Name=SourceBucket,Value=source-bucket Name=DestinationBucket,Value=dest-bucket \
  --start-time 2026-10-13T00:00:00Z \
  --end-time 2026-10-14T00:00:00Z \
  --period 300 \
  --statistics Average
```

**Access Logs Analysis:**
```bash
# Enable server access logging
aws s3api put-bucket-logging \
  --bucket source-bucket \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "logs-bucket",
      "TargetPrefix": "s3-access-logs/"
    }
  }'

# Query logs with Athena (create table first)
SELECT 
  request_datetime,
  remote_ip,
  operation,
  key,
  http_status,
  error_code
FROM s3_access_logs
WHERE bucket_name = 'example-bucket'
  AND http_status >= 400
  AND request_datetime > '2026-10-13'
ORDER BY request_datetime DESC;
```

## 10. Interview questions and answers

### Q1: Is S3 eventually consistent?

**Answer:** No, S3 provides **strong read-after-write consistency** for all operations (PUTs, DELETEs, and LIST). After a successful write, any subsequent read (including list operations) will return the latest version. This changed from the previous eventually consistent model.

**Important clarifications:**
- Applies to new objects AND overwrites of existing objects
- Applies within a single region for standard operations
- Replication (CRR/SRR) is still asynchronous
- Event notifications may have slight delays
- Does NOT provide multi-object transactions

**Exam scenario:** "Application writes object, immediately reads it back. Will it always see the latest data?" → Yes, strong consistency guarantees this.

---

### Q2: How do you prevent accidental deletion in S3?

**Multi-layered approach:**

1. **Versioning** → Preserve all versions; delete creates delete marker
2. **MFA Delete** → Require MFA token to permanently delete versions or disable versioning
3. **Object Lock** → WORM retention (Compliance mode prevents even root from deleting)
4. **Lifecycle policies** → Transition old versions to cheaper storage (not deletion)
5. **Cross-Region Replication** → Asynchronous copy to another region/account
6. **Bucket policies** → Explicit deny for delete operations from untrusted principals
7. **IAM least privilege** → Don't grant s3:DeleteObject unless needed
8. **Backup** → Use AWS Backup or S3 Batch Copy to isolated account

**Exam tip:** MFA Delete + Object Lock (Compliance) = strongest protection.

---

### Q3: Can we place irreplaceable data only in One Zone-IA?

**Short answer:** Not recommended for the only copy.

**Reasoning:**
- One Zone-IA stores data in a single AZ (11 nines durability IF the AZ is available)
- If the AZ fails, data is lost
- S3 Standard/Standard-IA store across ≥3 AZs (11 nines durability)

**Use One Zone-IA when:**
- Data is reproducible/re-creatable
- Secondary copy exists elsewhere
- Cost savings justify single-AZ risk

**Exam scenario:** "Cost-conscious startup wants to store only copy of customer data in One Zone-IA. Is this appropriate?" → No, use S3 Standard with lifecycle to IA/Glacier if cost is a concern.

---

### Q4: Why can the application list a bucket but not download objects?

**Root cause:** Different permissions required.

| Operation | Required Permission | Resource ARN |
|-----------|---------------------|--------------|
| List bucket | `s3:ListBucket` | `arn:aws:s3:::bucket-name` |
| Get object | `s3:GetObject` | `arn:aws:s3:::bucket-name/*` |
| Put object | `s3:PutObject` | `arn:aws:s3:::bucket-name/*` |

**Common policy mistake:**
```json
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "arn:aws:s3:::mybucket/*"  ❌ Missing bucket-level actions
}
```

**Correct policy:**
```json
{
  "Effect": "Allow",
  "Action": ["s3:ListBucket"],
  "Resource": "arn:aws:s3:::mybucket"
},
{
  "Effect": "Allow",
  "Action": ["s3:GetObject"],
  "Resource": "arn:aws:s3:::mybucket/*"
}
```

**Additional causes:**
- Object uses SSE-KMS → also need `kms:Decrypt` permission on the KMS key
- Bucket owner enforced → object ACLs ignored, bucket policy controls access
- Explicit deny in SCPs, permission boundaries, or session policies

---

### Q5: Does versioning make S3 a database transaction engine?

**Answer:** No. Versioning preserves multiple versions of individual objects, but does NOT provide:

- **ACID transactions** across multiple objects
- **Rollback** of multi-object changes
- **Consistency** across related objects
- **Isolation** for concurrent modifications

**What versioning does:**
- Preserves history of a single object
- Protects against accidental deletion (delete marker)
- Enables point-in-time recovery of individual objects
- Supports replication

**Exam tip:** For transactional workloads across multiple items, use DynamoDB transactions, RDS, or Aurora. S3 is for object storage, not transactional data.

---

### Q6: S3 vs EFS vs EBS – when to use each?

| Feature | S3 | EFS | EBS |
|---------|----|----|-----|
| **Type** | Object storage | Shared file system | Block storage |
| **Access** | HTTP/S API (REST) | NFS protocol (POSIX) | Attached volume (local disk) |
| **Sharing** | Many clients via API | Many EC2 instances (multi-attach) | Single EC2 (Multi-Attach EBS for limited use) |
| **Scale** | Unlimited, automatic | Petabytes, automatic | Fixed size, manual resize |
| **Durability** | 11 nines (Standard) | 11 nines (Standard) | Replicated within AZ |
| **Latency** | Higher (network API) | Low (file system) | Lowest (attached disk) |
| **Use case** | Static assets, backups, data lakes | Shared app data, content management | Boot volumes, databases, apps |
| **Cost** | Cheap storage, pay per request | Higher than S3, pay for provisioned | Pay for provisioned GB + IOPS |

**Decision matrix:**
- **S3:** Data lake, backups, static website, archival, infrequent access
- **EFS:** Shared filesystem for multiple EC2, containerized apps, content management
- **EBS:** Database volumes, application data, high-performance single-instance workloads

---

### Q7: What are the benefits of multipart upload?

**Benefits:**

1. **Large files** → Required for >5GB objects (max single PUT = 5GB)
2. **Parallel uploads** → Upload parts concurrently, faster throughput
3. **Resume** → If part fails, retry only that part (not entire object)
4. **Network optimization** → Smaller parts more resilient to transient failures
5. **Early start** → Begin upload before entire file is ready

**How it works:**
```
1. Initiate multipart upload → Get upload ID
2. Upload parts (5 MiB – 5 GiB each, up to 10,000 parts)
3. Complete upload → S3 assembles parts
OR
   Abort upload → Discard incomplete upload
```

**Best practices:**
- Use for objects >100MB
- Part size 5MB–100MB typical (balance throughput vs part count)
- Use S3 SDK (handles retry, parallelism automatically)
- Add lifecycle rule to auto-abort incomplete uploads after 7 days

**Cost trap:** Incomplete multipart uploads consume storage and cost money until aborted.

---

### Q8: When should I use S3 Select?

**Use S3 Select when:**
- Need to filter/query subset of large CSV/JSON/Parquet file
- Downloading entire object is wasteful
- Want to reduce data transfer and processing time

**Example:**
- 5 GB CSV file, need 50 MB of matching rows
- Without S3 Select: Download 5 GB → process locally → 5 GB transfer cost
- With S3 Select: S3 filters server-side → return 50 MB → 50 MB transfer cost

**SQL query example:**
```sql
SELECT * FROM S3Object s 
WHERE s.status = 'active' AND s.amount > 1000
```

**Limitations:**
- Simple SQL queries only (no JOINs)
- Max 256 KB per record
- Scan-based (not indexed)
- Best for columnar formats (Parquet)

**Alternatives:**
- **Athena:** Complex queries, multiple files, SQL analytics
- **Glue/EMR:** ETL, transformations, large-scale processing
- **Redshift Spectrum:** Combine S3 data with DW queries

---

### Q9: What is S3 Transfer Acceleration and when to use it?

**Overview:** Uses CloudFront edge locations to upload data faster over long distances.

**How it works:**
```
User (Tokyo) → CloudFront Edge (Tokyo) → AWS backbone → S3 (us-east-1)
  ├── Public internet (optimized) → Edge
  └── AWS backbone (faster, reliable) → S3 bucket
```

**When to use:**
- Geographically distributed users uploading to centralized bucket
- Large file uploads over long distances
- Users with unreliable/slow internet
- Backup/sync applications

**Configuration:**
```bash
# Enable Transfer Acceleration
aws s3api put-bucket-accelerate-configuration \
  --bucket my-bucket \
  --accelerate-configuration Status=Enabled

# Use accelerated endpoint
https://my-bucket.s3-accelerate.amazonaws.com/key
```

**Cost:** Additional per-GB fee (varies by region); test speed improvement first.

**Exam tip:** Not available for buckets with periods in name; not compatible with S3 over VPC endpoints.

---

### Q10: What is MFA Delete and how does it work?

**Purpose:** Require MFA token to permanently delete object versions or disable versioning.

**Protection level:**
- Without MFA Delete: User with `s3:DeleteObjectVersion` can permanently delete
- With MFA Delete: User needs `s3:DeleteObjectVersion` AND physical MFA token

**Enable MFA Delete:**
```bash
# Must use root account credentials with MFA
aws s3api put-bucket-versioning \
  --bucket secure-bucket \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"
```

**Delete with MFA:**
```bash
aws s3api delete-object \
  --bucket secure-bucket \
  --key important-file.txt \
  --version-id "version-id" \
  --mfa "arn:aws:iam::123456789012:mfa/root-account-mfa-device 123456"
```

**Limitations:**
- Only root account can enable/disable MFA Delete
- Only works with versioning enabled
- Applies to version deletion and versioning state changes

**Exam tip:** MFA Delete + Object Lock (Compliance mode) = maximum protection against deletion.

---

### Q11: Explain S3 Object Lock: Governance vs Compliance mode

| Feature | Governance Mode | Compliance Mode |
|---------|----------------|-----------------|
| **Protection** | WORM (Write Once Read Many) | WORM (Write Once Read Many) |
| **Override** | Special permission can bypass | NO override, even by root |
| **Use case** | Protect from most users, allow admin override | Regulatory compliance, absolute protection |
| **Permissions** | `s3:BypassGovernanceRetention` + `x-amz-bypass-governance-retention:true` header | Cannot bypass during retention period |
| **Retention change** | Can shorten/remove retention with permission | Cannot remove or shorten retention |
| **Deletion** | Possible with bypass permission | Impossible until retention expires |

**Legal Hold:**
- Independent of retention period
- On/off toggle per object version
- Requires `s3:PutObjectLegalHold` to add/remove
- Prevents deletion until removed

**Configuration:**
```bash
# Enable Object Lock (must be done at bucket creation)
aws s3api create-bucket \
  --bucket compliance-bucket \
  --object-lock-enabled-for-bucket

# Set default retention
aws s3api put-object-lock-configuration \
  --bucket compliance-bucket \
  --object-lock-configuration '{
    "ObjectLockEnabled": "Enabled",
    "Rule": {
      "DefaultRetention": {
        "Mode": "COMPLIANCE",
        "Days": 2555
      }
    }
  }'
```

**Exam scenario:** "Financial records must be retained for 7 years with no possibility of deletion, even by administrators." → S3 Object Lock (Compliance mode) with 7-year retention.

---

### Q12: How does cross-account S3 access work? Identity vs resource policies.

**Key concept:** Cross-account requires BOTH policies to allow (unlike same-account where either is sufficient).

**Architecture:**
```
Account A (111111111111)              Account B (222222222222)
├── IAM User: Alice               →   ├── S3 Bucket: shared-bucket
├── Identity Policy: Allow            └── Bucket Policy: Allow Account A
└── Calls: s3:GetObject
```

**Account A: Identity-based policy (Alice):**
```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::shared-bucket/*"
}
```

**Account B: Resource-based policy (bucket):**
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::111111111111:user/Alice"
  },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::shared-bucket/*"
}
```

**Alternative: Assume Role approach:**
```
Account A: Alice → AssumeRole(Account B Role) → Access S3
  ├── Alice needs sts:AssumeRole permission
  ├── Role has trust policy allowing Account A
  └── Role has permission policy for S3 access
```

**Trust policy (in Account B role):**
```json
{
  "Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::111111111111:root"},
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {"sts:ExternalId": "unique-external-id"}
  }
}
```

**Exam tip:** 
- Bucket policy approach = simpler, direct access
- Role assumption = centralized permission management, audit trail
- SSE-KMS cross-account = also need KMS key policy in Account B

---

### Q13: What causes high S3 costs and how to reduce them?

**Common cost drivers:**

1. **Storage of old versions** → Versioning without lifecycle policy
   - **Solution:** Lifecycle rule to expire noncurrent versions after 30/90 days
   
2. **Incomplete multipart uploads** → Failed uploads not aborted
   - **Solution:** Lifecycle rule to abort incomplete uploads after 7 days
   
3. **Wrong storage class** → Standard for rarely accessed data
   - **Solution:** Intelligent-Tiering or lifecycle transitions to IA/Glacier
   
4. **Many small objects** → Storage class minimum billing (128 KB for IA)
   - **Solution:** Aggregate small objects or use Standard class
   
5. **Frequent GET requests** → High request costs
   - **Solution:** CloudFront caching, client-side caching
   
6. **Data transfer out** → Large downloads to internet
   - **Solution:** CloudFront (cheaper egress), S3 Transfer Acceleration
   
7. **Cross-region transfer** → Replication or VPC endpoint
   - **Solution:** Same-region replication if possible
   
8. **Retrieval fees** → Glacier/IA frequent retrievals
   - **Solution:** Use correct tier, bulk retrievals, cache frequently accessed

**Cost optimization CLI:**
```bash
# Find incomplete multipart uploads
aws s3api list-multipart-uploads --bucket my-bucket

# Lifecycle policy to abort incomplete uploads
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration '{
  "Rules": [{
    "Id": "AbortIncompleteUploads",
    "Status": "Enabled",
    "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7},
    "Filter": {"Prefix": ""}
  }]
}'

# Expire old versions
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration '{
  "Rules": [{
    "Id": "ExpireOldVersions",
    "Status": "Enabled",
    "NoncurrentVersionExpiration": {"NoncurrentDays": 90},
    "Filter": {"Prefix": ""}
  }]
}'
```

**Use S3 Storage Lens** for bucket-level cost and usage analytics.

---

## Quick Reference Card

### Storage Class Decision Tree

```
Start: What's the access pattern?

├─ Frequently accessed (>1/month) → S3 Standard
│
├─ Unknown/changing access → S3 Intelligent-Tiering
│  └─ Enable archive tiers if >90/180 days without access acceptable
│
├─ Infrequent (1/month to 1/quarter) → S3 Standard-IA
│  ├─ Data is critical → Standard-IA (multi-AZ)
│  └─ Data is re-creatable → One Zone-IA (cheaper, single-AZ)
│
├─ Archive (1/quarter to 1/year) → Glacier Instant Retrieval
│  └─ Need millisecond access for rare retrieval
│
├─ Archive (occasional, hours OK) → Glacier Flexible Retrieval
│  ├─ Expedited: 1-5 minutes ($$$)
│  ├─ Standard: 3-5 hours ($$)
│  └─ Bulk: 5-12 hours ($)
│
└─ Long-term archive (rarely/never) → Glacier Deep Archive
   └─ Standard: 12 hours, Bulk: 48 hours
```

### Key Limits and Specifications

| Limit | Value | Notes |
|-------|-------|-------|
| **Max object size** | 50 TB | Announced Dec 2025; old limit was 5 TB |
| **Single PUT max** | 5 GB | Use multipart for larger |
| **Multipart part size** | 5 MiB – 5 GiB | Final part can be smaller |
| **Multipart max parts** | 10,000 | Plan part size accordingly |
| **Bucket name length** | 3–63 characters | Lowercase, numbers, hyphens only |
| **Max buckets/account** | 100 (default) | Soft limit, request increase |
| **Max keys/bucket** | Unlimited | No limit on number of objects |
| **Max tags/object** | 10 | Key-value pairs |
| **Prefix performance** | 3,500 PUT/POST/DELETE<br/>5,500 GET/HEAD per second | Per prefix, scales horizontally |

### Consistency Guarantees

| Operation | Consistency | Notes |
|-----------|------------|-------|
| **PUT (new object)** | Strong read-after-write | Immediate consistency |
| **PUT (overwrite)** | Strong read-after-write | Immediate consistency |
| **DELETE** | Strong read-after-write | Immediate consistency |
| **LIST** | Strong consistency | Shows latest state |
| **Replication** | Asynchronous | Eventual (seconds to minutes) |
| **Event notifications** | Asynchronous | Eventual (typically <1 second) |

### Versioning + Lifecycle Combinations

| Configuration | Behavior | Use Case |
|---------------|----------|----------|
| **Versioning OFF** | Overwrite/delete removes object permanently | Simple storage, no history needed |
| **Versioning ON, no lifecycle** | All versions kept forever | Audit requirements, unlimited retention |
| **Versioning ON + expire noncurrent** | Keep current version, delete old after N days | Balance protection + cost |
| **Versioning ON + transition noncurrent** | Current = Standard, old → IA/Glacier | Cheaper long-term version storage |
| **Versioning ON + MFA Delete** | Require MFA to delete versions | High-security environments |
| **Versioning ON + Object Lock** | WORM retention, compliance | Regulatory compliance (SEC, HIPAA) |

### Common CLI Commands

```bash
# Upload with storage class
aws s3 cp file.txt s3://bucket/key --storage-class INTELLIGENT_TIERING

# Multipart upload (automatic with high-level API)
aws s3 cp large-file.zip s3://bucket/key

# Sync directory to S3
aws s3 sync ./local-dir s3://bucket/prefix/ --delete

# Download specific version
aws s3api get-object --bucket bucket --key file.txt --version-id VERSION_ID output.txt

# Generate presigned URL (expires in 1 hour)
aws s3 presign s3://bucket/key --expires-in 3600

# Copy between buckets (same or cross-account)
aws s3 cp s3://source-bucket/key s3://dest-bucket/key

# List all versions including delete markers
aws s3api list-object-versions --bucket bucket --prefix key

# Restore Glacier object (Standard tier, available for 7 days)
aws s3api restore-object --bucket bucket --key key \
  --restore-request '{"Days":7,"GlacierJobParameters":{"Tier":"Standard"}}'

# Check replication status
aws s3api head-object --bucket bucket --key key | jq '.ReplicationStatus'

# Enable versioning
aws s3api put-bucket-versioning --bucket bucket \
  --versioning-configuration Status=Enabled

# Enable encryption (SSE-S3)
aws s3api put-bucket-encryption --bucket bucket \
  --server-side-encryption-configuration '{
    "Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]
  }'
```

### Permission Cheat Sheet

| Operation | IAM Action | Resource ARN | Common Issue |
|-----------|-----------|--------------|--------------|
| **List bucket** | `s3:ListBucket` | `arn:aws:s3:::bucket` | Using `bucket/*` instead of `bucket` |
| **Get object** | `s3:GetObject` | `arn:aws:s3:::bucket/*` | Missing `kms:Decrypt` for SSE-KMS |
| **Put object** | `s3:PutObject` | `arn:aws:s3:::bucket/*` | Block Public Access preventing write |
| **Delete object** | `s3:DeleteObject` | `arn:aws:s3:::bucket/*` | Versioning creates delete marker |
| **Delete version** | `s3:DeleteObjectVersion` | `arn:aws:s3:::bucket/*` | MFA Delete requires MFA token |
| **Get bucket location** | `s3:GetBucketLocation` | `arn:aws:s3:::bucket` | Needed for SDK region detection |
| **List versions** | `s3:ListBucketVersions` | `arn:aws:s3:::bucket` | Different from `ListBucket` |

---

## Exam Tips Summary

### Must-Know Concepts (SAA-C03)

1. **Strong Consistency** → S3 provides read-after-write consistency for all operations (changed ~2020)

2. **50 TB Object Max** → Updated Dec 2025 (exam may still reference old 5 TB limit)

3. **Multipart Upload** → Required for >5GB, recommended for >100MB

4. **Storage Class Selection** → Match access pattern to class; consider retrieval costs not just storage

5. **Versioning ≠ Backup** → Versioning protects against accidental delete/overwrite, not regional failure

6. **Object Lock Modes** → Governance (can bypass with permission) vs Compliance (cannot bypass, even root)

7. **Cross-Account Access** → Requires BOTH identity policy AND bucket policy (unlike same-account)

8. **SSE-KMS Permissions** → Need both S3 permissions AND `kms:Decrypt`/`kms:Encrypt` on key

9. **Replication Requirements** → Both buckets must have versioning enabled; use IAM role with KMS permissions

10. **S3 vs EFS vs EBS** → Object storage vs shared filesystem vs block storage

11. **Presigned URLs** → Inherit signer's permissions at generation time; expire when temporary creds expire

12. **Lifecycle Policies** → Can transition or expire objects; cannot move FROM Glacier Flexible/Deep Archive

13. **Event Notifications** → Asynchronous, may duplicate; make processors idempotent

14. **Transfer Acceleration** → Uses CloudFront edges; not compatible with VPC endpoints

15. **MFA Delete** → Only root can enable; requires versioning; protects against version deletion

### Common Exam Scenarios

| Scenario | Solution |
|----------|----------|
| **Global users uploading to centralized bucket slowly** | S3 Transfer Acceleration |
| **Protect financial records from deletion for 7 years** | S3 Object Lock (Compliance mode) with 7-year retention |
| **Cost optimization for infrequently accessed data** | Lifecycle policy to transition to Standard-IA or Intelligent-Tiering |
| **Old object versions consuming excessive storage** | Lifecycle policy to expire noncurrent versions after N days |
| **Cross-region disaster recovery** | S3 Cross-Region Replication with versioning enabled |
| **Share bucket data with another AWS account** | Bucket policy allowing cross-account access + identity policy in other account |
| **Static website with global edge caching** | S3 origin + CloudFront distribution with OAC |
| **Automatic image processing when uploaded** | S3 Event Notification → Lambda function |
| **Query large CSV without downloading entire file** | S3 Select with SQL filtering |
| **Incomplete uploads causing high costs** | Lifecycle rule to abort incomplete multipart uploads after 7 days |
| **Prevent accidental public exposure** | Enable S3 Block Public Access (account and bucket level) |
| **Encrypt data at rest with own keys** | SSE-KMS with customer managed CMK |
| **Temporary upload access without AWS credentials** | Presigned URL (PUT) with short expiration |
| **Archive data with hours-scale retrieval acceptable** | Glacier Flexible Retrieval (Standard tier = 3-5 hours) |
| **Ensure data integrity during upload** | Use MD5 checksum or AWS checksums (SHA256, CRC32) |

### Red Flags (Anti-Patterns)

❌ **Single AZ for critical data only copy** → Use S3 Standard (multi-AZ), not One Zone-IA

❌ **Versioning without lifecycle** → Old versions accumulate, high cost

❌ **S3 as database** → Use RDS/DynamoDB for transactional workloads

❌ **Many small objects in IA classes** → Minimum billable size (128 KB); use Standard or aggregate

❌ **Presigned URLs with long expiration** → Security risk; use 5-15 minutes for uploads

❌ **S3 Event → Lambda writes to same prefix** → Infinite loop; use different bucket/prefix

❌ **Lifecycle transition to then from Glacier** → Cannot transition FROM Glacier Flexible/Deep Archive

❌ **Expecting synchronous replication** → Replication is asynchronous (seconds to minutes)

❌ **Cross-account access without both policies** → Need identity policy AND resource policy

❌ **SSE-KMS without KMS permissions** → Need both s3:GetObject AND kms:Decrypt

❌ **Public bucket for static website** → Use CloudFront + private bucket with OAC instead

❌ **ListBucket permission on bucket/*** → Should be on bucket ARN (no /*)

❌ **Object Lock on existing bucket** → Must enable at bucket creation time

❌ **MFA Delete without versioning** → MFA Delete requires versioning enabled

❌ **Transfer Acceleration with VPC endpoint** → Incompatible; choose one or the other

---

## 11. Quick revision

- Object store; not block storage or an ordinary shared filesystem.
- Strong object consistency; asynchronous replication/notifications.
- Current advertised object maximum = 50 TB.
- Versioning, replication, lifecycle, and Object Lock solve different problems.
- Encryption and authorization are separate.
- Choose storage by access/recovery needs, not just per-GB price.

## 13. AWS sources

- [S3 consistency](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [50 TB announcement](https://aws.amazon.com/about-aws/whats-new/2025/12/amazon-s3-maximum-object-size-50-tb/)
- [Object size semantics](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingObjects.html)
- [Multipart quotas](https://docs.aws.amazon.com/AmazonS3/latest/userguide/qfacts.html)
- [Storage classes](https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html)
- [Object Lock](https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lock.html)
- [Replication](https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html)
- [Encryption](https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingEncryption.html)
- [Object Ownership](https://docs.aws.amazon.com/AmazonS3/latest/userguide/about-object-ownership.html)

**Related:** [KMS](aws-kms-cheatsheet.md), [Lambda](aws-lambda-cheatsheet.md), [API Gateway](aws-api-gateway-cheatsheet.md).
