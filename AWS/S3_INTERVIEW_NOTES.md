# Amazon S3 — Interview Notes

## 1. What is Amazon S3?

Amazon S3 (Simple Storage Service) is AWS **object storage**.

It stores data as:

```text
Bucket
  |
  +-- Object
  +-- Object
  +-- Object
```

Example:

```text
my-company-bucket
   |
   +-- reports/2026/report.pdf
   +-- images/logo.png
   +-- backup/app.tar.gz
```

An object consists conceptually of:

```text
Object =
Data
+ Key
+ Metadata
+ optional Version ID
```

Important interview distinction:

```text
S3  = Object Storage
EBS = Block Storage
EFS = File Storage
```

Typical S3 use cases:
- Backups
- Logs
- Application files
- Static content
- Data lakes
- Artifacts
- Media
- Archive data
- Terraform state

---

## 2. Is S3 Inside a VPC?

No.

S3 is a regional AWS managed service. A bucket is not created inside a VPC or subnet.

```text
Your VPC
   |
EC2 / EKS
   |
   +--------> Amazon S3
```

Workloads can access S3 through:
- Public AWS endpoints
- S3 VPC endpoints

---

## 3. Is S3 Public or Private by Default?

S3 should be treated as private by default.

A key security control is:

```text
S3 Block Public Access
```

Typical secure configuration:

```text
S3 Bucket
   |
Block Public Access = ON
   |
IAM / Bucket Policy
   |
Authorized users/applications only
```

---

## 4. How Do I Make an S3 Bucket Private?

Keep:

```text
Block Public Access = ON
```

Then grant only the required permissions through:
- IAM Policy
- Bucket Policy
- Access Point
- VPC Endpoint Policy

Example IAM permission for reading objects:

```json
{
  "Effect": "Allow",
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::my-bucket/*"
}
```

For listing the bucket:

```json
{
  "Effect": "Allow",
  "Action": "s3:ListBucket",
  "Resource": "arn:aws:s3:::my-bucket"
}
```

Important ARN distinction:

```text
Bucket:
arn:aws:s3:::my-bucket

Objects:
arn:aws:s3:::my-bucket/*
```

---

## 5. How Do I Make S3 Public?

If there is a real requirement to expose objects publicly, the relevant Block Public Access setting must not block public policies, and an explicit bucket policy can allow public reads.

Example:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-public-bucket/*"
    }
  ]
}
```

This means:

```text
Anyone
  |
  | s3:GetObject
  v
Objects in bucket
```

For production, prefer:

```text
Internet
   |
   v
CloudFront
   |
   | Origin Access Control
   v
Private S3 Bucket
```

Best interview answer:

> “I normally keep S3 private. If content must be publicly available, I prefer CloudFront with Origin Access Control in front of a private S3 bucket rather than making the bucket directly public.”

---

## 6. How Do I Access a Private S3 Bucket?

### From EC2

Use an IAM role attached to the instance.

```text
EC2
 |
IAM Role
 |
 v
S3
```

Do not store long-lived AWS access keys on the EC2 instance.

Examples:

```bash
aws s3 ls s3://my-bucket
aws s3 cp file.txt s3://my-bucket/
```

### From EKS

Prefer IAM-based workload identity instead of static AWS credentials.

```text
Pod
 |
IAM identity / role
 |
 v
S3
```

### From Lambda

```text
Lambda
   |
Execution Role
   |
   v
S3
```

### From a User / Laptop

```text
User
 |
IAM / SSO credentials
 |
AWS CLI / SDK / Console
 |
S3
```

---

## 7. How Does a Private Subnet Access S3?

Without a VPC endpoint:

```text
EC2 / EKS in Private Subnet
       |
       v
NAT Gateway
       |
       v
S3 Public Endpoint
```

Preferred for private AWS access:

```text
EC2 / EKS in Private Subnet
       |
       v
S3 Gateway VPC Endpoint
       |
       v
Amazon S3
```

No NAT Gateway is required for that S3 traffic.

---

## 8. S3 Gateway VPC Endpoint

S3 commonly uses a Gateway VPC Endpoint.

The route table may contain a route conceptually like:

```text
Destination         Target

10.0.0.0/16         local
S3 Prefix List      vpce-xxxx
0.0.0.0/0           NAT
```

Traffic behavior:

```text
Traffic to S3
     |
     v
VPC Endpoint
```

```text
Other Internet Traffic
     |
     v
NAT Gateway
```

Interview answer:

> “For EC2 or EKS running in private subnets, I can use an S3 Gateway Endpoint so S3 traffic does not need to traverse the NAT Gateway.”

---

## 9. IAM Policy vs Bucket Policy

### IAM Policy

Attached to:
- User
- Role
- Group

It answers:

> “What is this identity allowed to do?”

### Bucket Policy

Attached directly to the S3 bucket.

It answers:

> “Who can access this bucket and under what conditions?”

Common enterprise design uses both.

---

## 10. Bucket Policy Conditions

You can restrict bucket access by conditions such as:
- IAM principal
- VPC endpoint
- Source IP
- AWS account
- TLS
- AWS Organization

Example security pattern:

```text
HTTP request
   |
   X Denied

HTTPS request
   |
   ✓ Allowed
```

---

## 11. ACLs

Older S3 designs used ACLs extensively.

Modern designs generally prefer:

```text
IAM Policies
+
Bucket Policies
```

Interview answer:

> “For modern S3 security, I prefer IAM and bucket policies and generally keep ACLs disabled unless there is a specific legacy requirement.”

---

## 12. Presigned URLs

A presigned URL gives temporary access to a private S3 object without making the bucket public.

```text
Private S3 Object
        |
 Generate Presigned URL
        |
        v
Temporary URL
        |
        v
User downloads object
```

Example use case:

```text
User
  |
Application
  |
Generate Presigned URL
  |
User -> S3 Directly
```

---

## 13. Encryption

Think about:

```text
Encryption at Rest
Encryption in Transit
```

### At Rest

Common options:
- SSE-S3
- SSE-KMS

SSE-S3:

```text
AWS manages encryption keys
```

SSE-KMS:

```text
AWS KMS Key
    |
More control
Auditing
Key policies
```

### In Transit

Use HTTPS/TLS.

You can enforce HTTPS using a bucket policy.

---

## 14. Versioning

Versioning keeps multiple versions of objects.

```text
config.json v1
config.json v2
config.json v3
```

Benefits:
- Recover from accidental overwrite
- Recover from accidental delete
- Useful in DR and ransomware protection

Trade-off:
- Older versions consume storage until lifecycle rules clean them up

---

## 15. Lifecycle Policies

Lifecycle policies automatically transition or delete objects.

Example:

```text
Day 0
 |
S3 Standard
 |
30 days
 |
Standard-IA
 |
90 days
 |
Glacier
 |
7 years
 |
Delete
```

Common for:
- Logs
- Backups
- Compliance data
- Archives

---

## 16. S3 Storage Classes

| Storage Class | Typical Use |
|---|---|
| S3 Standard | Frequently accessed data |
| Intelligent-Tiering | Unknown or changing access pattern |
| Standard-IA | Infrequent access |
| One Zone-IA | Infrequent, recreatable data |
| Glacier Instant Retrieval | Archive needing fast retrieval |
| Glacier Flexible Retrieval | Archive |
| Glacier Deep Archive | Long-term lowest-cost archive |

Quick memory:

```text
Frequently Accessed -> Standard
Unknown Pattern     -> Intelligent-Tiering
Rarely Accessed     -> IA / Glacier
```

---

## 17. Versioning + Lifecycle

Common combination:

```text
Current Object
   |
Old Version
   |
30 days
   v
IA
   |
90 days
   v
Glacier
   |
365 days
   v
Delete
```

---

## 18. S3 Replication

Two important patterns:

```text
SRR = Same-Region Replication
CRR = Cross-Region Replication
```

Example DR pattern:

```text
Region A
S3 Bucket
   |
   | Cross-Region Replication
   v
Region B
S3 Bucket
```

Use cases:
- DR
- Compliance
- Data locality
- Separate-account copies

---

## 19. S3 Consistency

S3 provides strong read-after-write consistency.

```text
PUT Object
   |
Successful
   |
GET Object
   |
Latest Object Returned
```

Interview answer:

> “After a successful S3 write, I can immediately read the latest object.”

---

## 20. Static Website Hosting

S3 can host static content:
- HTML
- CSS
- JavaScript
- Images

But S3 is not an application server.

You cannot run long-lived backend processes inside S3.

Preferred production static-site pattern:

```text
Route 53
   |
CloudFront
   |
Private S3
```

---

## 21. S3 Event Notifications

S3 events can trigger processing.

Example:

```text
Object Uploaded
       |
       v
      S3
       |
       v
   Lambda
       |
       v
Process File
```

Other integrations:
- SQS
- SNS
- EventBridge

---

## 22. Multipart Upload

Multipart upload is used for large files.

```text
Part 1
Part 2
Part 3
...
   |
Upload Concurrently
   |
Combine in S3
```

Advantages:
- Faster uploads
- Parallel transfer
- Retry individual parts
- Better reliability for large objects

---

## 23. S3 Object Lock

Used when data must not be modified or deleted for a defined retention period.

Typical use cases:
- Financial records
- Audit records
- Regulatory archives

Think:

```text
Write Once
Read Many
```

---

## 24. Logging and Auditing

Common tools:
- CloudTrail
- S3 server access logging
- CloudWatch where applicable

CloudTrail can help answer:
- Who accessed the bucket?
- Who changed the bucket policy?
- Who deleted an object?

---

## 25. S3 Availability vs DR

High durability does not remove the need for DR planning.

Depending on requirements, use:

```text
Versioning
+
Cross-Region Replication
+
Backups
+
Object Lock
```

Example:

```text
Production Bucket - Region A
          |
          | CRR
          v
DR Bucket - Region B
```

---

## 26. Typical Enterprise S3 Architecture

For public content:

```text
Internet
   |
   v
CloudFront
   |
   v
Private S3 Bucket
```

For private workloads:

```text
Private EKS / EC2
       |
       v
S3 Gateway Endpoint
       |
       v
Private S3 Bucket
```

Security model:

```text
IAM Role
   +
Bucket Policy
   +
KMS
   +
Block Public Access
```

---

# Top S3 Interview Q&A

## Q1. Is S3 inside a VPC?

No. S3 is a regional managed AWS service and is not created inside your VPC or subnet.

## Q2. How do private EC2/EKS workloads access S3?

Either through NAT to the S3 public endpoint or, preferably, through an S3 Gateway VPC Endpoint.

## Q3. How do you make S3 private?

Keep Block Public Access enabled and allow only the required IAM roles and bucket policies.

## Q4. How do you make S3 public?

Allow public read access through a bucket policy after changing the relevant Block Public Access configuration. In production, prefer CloudFront in front of a private bucket.

## Q5. IAM Policy vs Bucket Policy?

IAM policy controls what an identity can do. Bucket policy controls who can access the bucket and under what conditions.

## Q6. How do you temporarily share a private object?

Use a presigned URL.

## Q7. How do you encrypt S3?

Use SSE-S3 or SSE-KMS at rest and HTTPS/TLS in transit.

## Q8. How do you protect against accidental overwrite or deletion?

Enable versioning.

## Q9. How do you reduce storage cost for old data?

Use lifecycle policies and appropriate storage classes.

## Q10. How do you provide regional DR for S3?

Use Cross-Region Replication, versioning, and an appropriate recovery design.

---

# Interview-Ready Summary

> “Amazon S3 is AWS object storage. I normally keep buckets private using S3 Block Public Access and control access with IAM roles and bucket policies. EC2, EKS, and Lambda should use IAM roles rather than static credentials. Private workloads can access S3 through an S3 Gateway VPC Endpoint without using NAT. If public content is required, I prefer CloudFront with Origin Access Control in front of a private bucket rather than exposing the bucket directly. For enterprise designs I also consider encryption with KMS, versioning, lifecycle rules, replication, auditing, and Object Lock depending on the business requirement.”
