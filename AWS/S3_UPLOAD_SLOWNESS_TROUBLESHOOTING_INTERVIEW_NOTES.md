# S3 Upload Slowness Troubleshooting — Interview Notes

## Scenario

> A client is trying to upload a document to Amazon S3 and is experiencing slowness. How would you troubleshoot it?

The key is to troubleshoot **layer by layer** instead of immediately assuming S3 is the problem.

---

## 1. Understand the Traffic Path

Typical path:

```text
Client / Application
        |
        | DNS
        v
S3 Endpoint
bucket.s3.<region>.amazonaws.com
        |
        | HTTPS / TLS
        v
Amazon S3
        |
        +--> IAM authorization
        |
        +--> Optional KMS encryption
        |
        +--> Object stored
```

If the client is running inside AWS:

```text
EC2 / Application
      |
      +---- S3 Gateway Endpoint ----> S3
      |
      OR
      |
      +---- NAT Gateway / Internet ----> S3
```

---

# 2. First Clarify What "Slow" Means

Determine:

```text
One file or all files?
Small files or large files?
One client or everyone?
Started recently or always slow?
One location/region or all?
Are downloads normal?
```

Also compare object sizes:

```text
10 MB upload taking 30 seconds
vs
50 GB upload taking several minutes
```

The second may simply be constrained by available bandwidth.

---

# 3. Determine Whether It Is Client-Specific or Widespread

Test the same upload from another system:

```bash
time aws s3 cp testfile.bin s3://my-bucket/testfile.bin
```

Example:

```text
Client A → 2 MB/s
Client B → 80 MB/s
```

Interpretation:

```text
Only Client A slow
→ investigate Client A and its network path

All clients slow
→ investigate shared network, configuration, region, or AWS-side behavior
```

---

# 4. Check the Client First

The source system itself may be the bottleneck.

Check CPU:

```bash
top
```

Check memory:

```bash
free -m
```

Check disk performance:

```bash
iostat -xz 1 5
```

Check NIC/network usage:

```bash
sar -n DEV 1 5
```

Important example:

```text
Local disk read rate = 20 MB/s
Network capacity      = 1 Gbps
```

The S3 upload cannot be faster than the source can read the file.

---

# 5. Check DNS, TCP and TLS

Test DNS:

```bash
dig bucket-name.s3.ap-south-1.amazonaws.com
```

Test connection timing:

```bash
curl -s -o /dev/null \
-w "dns=%{time_namelookup} tcp=%{time_connect} tls=%{time_appconnect} total=%{time_total}\n" \
https://bucket-name.s3.ap-south-1.amazonaws.com
```

Interpretation:

```text
DNS slow
→ resolver / DNS path issue

TCP slow
→ routing / firewall / network issue

TLS slow
→ proxy / TLS inspection / handshake issue

Connection setup fast but upload slow
→ investigate bandwidth, upload method, retries, client resources
```

Memory model:

```text
DNS
 ↓
TCP
 ↓
TLS
 ↓
HTTP / S3
```

---

# 6. Check Bandwidth, Latency and Packet Loss

Possible causes:

```text
Bandwidth saturation
Packet loss
High WAN latency
Corporate proxy
VPN
Firewall inspection
WAN link congestion
```

Useful commands:

```bash
ping <reachable-endpoint>
mtr <destination>
```

Always compare expected throughput with link capacity.

Example:

```text
Client connection = 100 Mbps
Theoretical maximum ≈ 12.5 MB/s
```

If the user expects 100 MB/s, the expectation is already above the link capacity.

Packet loss matters because:

```text
Packet loss
   ↓
TCP retransmission
   ↓
Congestion window reduces
   ↓
Upload throughput falls
```

---

# 7. Check Client Region vs S3 Bucket Region

Example:

```text
Client = India
S3 bucket = us-east-1
```

Path:

```text
India Client
    |
    | Long WAN path
    | Higher RTT
    v
US-East S3
```

Check bucket region:

```bash
aws s3api get-bucket-location --bucket my-bucket
```

Compare this with where the client/application is running.

Preferred pattern:

```text
Application
   ↓
Appropriate / nearest AWS Region
   ↓
S3 in same Region
```

For globally distributed users, S3 Transfer Acceleration may help:

```text
Remote Client
     |
     v
Nearest AWS Edge
     |
     v
AWS Backbone
     |
     v
S3 Bucket
```

Do not enable it blindly. First prove geographic/network latency is the issue.

---

# 8. Large File? Check Multipart Upload

Large files should normally use multipart upload.

```text
Large File
   |
   +--> Part 1 ----\
   +--> Part 2 -----\
   +--> Part 3 ------> S3
   +--> Part 4 -----/
   +--> Part 5 ----/
```

Benefits:

```text
Parallel uploads
Higher throughput
Independent retry of failed parts
Better handling of large objects
```

If an application is doing:

```text
Single PUT
   ↓
Huge object
```

while another uses multipart upload, performance can differ significantly.

AWS CLI high-level S3 commands and SDK transfer managers support multipart behavior for large transfers.

---

# 9. Check Retries and Timeouts

A client may report:

> "S3 upload takes 40 seconds."

But internally:

```text
PUT attempt
   ↓
timeout
   ↓
retry
   ↓
timeout
   ↓
retry
   ↓
success
```

The upload may appear slow even though the real issue is repeated retries.

Temporarily enable CLI diagnostics:

```bash
aws s3 cp file s3://bucket/ --debug
```

Look for:

```text
Retries
Timeouts
Connection resets
Slow TLS establishment
HTTP 5xx
HTTP 503
```

Do not leave verbose debug logging enabled unnecessarily in production.

---

# 10. Check the AWS Network Path

If the application is on EC2, determine how it reaches S3.

Preferred architecture:

```text
EC2
 |
 v
S3 Gateway VPC Endpoint
 |
 v
S3
```

Alternative path:

```text
EC2 private subnet
 |
 v
NAT Gateway
 |
 v
Public S3 endpoint
 |
 v
S3
```

Also look for unnecessary hops:

```text
EC2
 ↓
Proxy
 ↓
Firewall
 ↓
NAT
 ↓
S3
```

Each additional hop can introduce latency, throughput limits, or inspection overhead.

Check:

```bash
ip route
```

And review:

```text
Subnet route table
S3 VPC endpoint
NAT Gateway
Firewall
Proxy
```

For AWS-hosted workloads, prefer an appropriate S3 VPC endpoint instead of unnecessarily routing S3 traffic through a NAT path.

---

# 11. Check Encryption / KMS Only if Evidence Points There

If using:

```text
SSE-KMS
```

then KMS is involved in the encryption workflow.

```text
Application
     |
     v
S3
 |
 +--> KMS
 |
 v
Object stored
```

Check for:

```text
KMS throttling
KMS authorization errors
KMS-related latency/errors
```

Do not start with KMS unless metrics or logs indicate it.

---

# 12. Small Files vs Large Files

Uploading many small objects has different overhead from uploading one large object.

Example:

```text
100,000 × 5 KB files
```

Each request may involve:

```text
HTTP request
Authentication
TLS / connection handling
S3 request processing
```

This can be much less efficient than:

```text
1 × 500 MB object
```

Potential improvements:

```text
Parallelism
Connection reuse
Batching/aggregation where appropriate
```

---

# 13. End-to-End Troubleshooting Flow

```text
User reports slow S3 upload
          ↓
Confirm file size / timing / scope
          ↓
One client or everyone?
          ↓
Check client
CPU / disk / NIC
          ↓
Check DNS → TCP → TLS
          ↓
Check network bandwidth / latency / packet loss
          ↓
Check client region vs bucket region
          ↓
Check route
VPC Endpoint / NAT / Proxy
          ↓
Large object?
Check multipart / parallelism
          ↓
Check retries / SDK logs
          ↓
Check S3 / KMS errors or throttling
          ↓
Mitigate
          ↓
Verify actual upload throughput
```

---

# 14. Recovery / Mitigation

## Network bottleneck

```text
Increase available bandwidth
Fix packet loss
Remove or bypass problematic proxy path
Fix routing/firewall bottleneck
```

## Cross-region / geographic latency

```text
Use an S3 bucket in the appropriate region
or
Consider S3 Transfer Acceleration
```

## Large files

```text
Use multipart upload
Increase safe parallelism
```

## Many small files

```text
Use concurrency
Reuse connections
Batch/aggregate where possible
```

## EC2 unnecessarily using NAT

```text
Use S3 Gateway VPC Endpoint
```

## Retry storm

```text
Correct timeout settings
Use bounded retries
Use exponential backoff and jitter
```

## Client resource bottleneck

```text
Fix disk throughput
Fix CPU contention
Fix NIC saturation
```

---

# 15. Verify Recovery

Do not stop after making a change.

Verify:

```text
Upload throughput
Total upload time
Retry count
Packet loss
CPU
Disk throughput
Network utilization
Error rate
```

Example:

```text
Before:
Upload = 3 MB/s
Retries = frequent

After:
Upload = 70 MB/s
Retries = none
```

That provides evidence that the mitigation actually solved the problem.

---

# 16. Interview-Ready Answer

> **I would first establish whether the bottleneck is S3, the client, or the network path between them. I would compare the upload from another client, check the source machine's CPU, disk and network, and then test DNS, TCP and TLS connection times. Next I would look at bandwidth, packet loss, the client-to-bucket region distance, and whether an AWS-hosted workload is reaching S3 through a VPC endpoint or unnecessarily through NAT, proxies or inspection devices. For large objects I would verify multipart and parallel uploads, and I would inspect SDK retries because repeated timeouts can make S3 appear slow. Once I identify the bottleneck, I would mitigate it and verify recovery using actual upload throughput and customer-visible upload latency.**

---

# 17. Interview Memory Shortcut

```text
CLIENT
  ↓
LOCAL RESOURCE
  ↓
DNS
  ↓
NETWORK
  ↓
REGION / ROUTE
  ↓
UPLOAD METHOD
  ↓
RETRIES
  ↓
S3 / KMS
```

The key SRE principle is:

> **Do not start by assuming S3 is slow. Prove which layer is introducing the latency.**
