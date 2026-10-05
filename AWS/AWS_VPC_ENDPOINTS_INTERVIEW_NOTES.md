# AWS VPC Endpoints — Interview Notes

## 1. What Is a VPC Endpoint?

A VPC Endpoint lets resources inside a VPC access supported AWS services privately, without routing traffic through a NAT Gateway or Internet Gateway.

For interview purposes, remember the two main types:

1. Gateway VPC Endpoint
2. Interface VPC Endpoint

---

# 2. Gateway VPC Endpoint

Primarily used for:

- Amazon S3
- DynamoDB

High-level flow:

```text
EKS Pod / EC2
     |
Private Subnet
     |
Route Table
     |
S3 Prefix List -> VPC Endpoint
     |
     v
   Amazon S3
```

A Gateway Endpoint is not an ENI and is not placed inside a subnet.

It works through route-table integration.

---

# 3. Interface VPC Endpoint

Used for many AWS services such as:

- Secrets Manager
- ECR
- CloudWatch
- STS
- KMS
- SSM
- Private API Gateway APIs

It uses AWS PrivateLink.

```text
EKS Pod / EC2
      |
      v
Private Subnet
      |
      v
Interface Endpoint
Private IP / ENI
      |
   PrivateLink
      |
      v
AWS Service
```

AWS creates an ENI with a private IP in the selected subnet.

With Private DNS enabled, the normal AWS service hostname can resolve to that private endpoint IP.

---

# 4. Gateway vs Interface Endpoint

| Gateway Endpoint | Interface Endpoint |
|---|---|
| Mainly S3 and DynamoDB | Many AWS services |
| Route-table based | ENI/private-IP based |
| No ENI | Creates ENI |
| No endpoint hourly charge | Usually hourly + data processing charge |
| Uses route table | Uses PrivateLink |

---

# 5. Sample Route Table for EKS Private Subnet

Example private route table:

```text
Private Route Table - AZ-A
-----------------------------------------------
Destination            Target
-----------------------------------------------
10.0.0.0/16            local
pl-xxxxxxxx             vpce-0abc1234
0.0.0.0/0              nat-0aaa1111
```

Meaning:

```text
10.0.0.0/16 -> local
```

Used for communication within the VPC.

Examples:

```text
EKS Pod -> RDS
Worker -> Internal ALB
Private Subnet A -> Private Subnet B
```

```text
pl-xxxxxxxx -> vpce-0abc1234
```

This is the S3 route.

```text
0.0.0.0/0 -> NAT Gateway
```

All other internet-bound traffic goes through NAT.

---

# 6. Traffic Selection Example

```text
EKS Pod
   |
   +--> 10.0.21.10
   |       |
   |       +--> local -> RDS
   |
   +--> S3
   |       |
   |       +--> S3 Prefix List -> VPC Endpoint
   |
   +--> google.com
           |
           +--> 0.0.0.0/0 -> NAT Gateway
```

For three AZs:

```text
Private RT AZ-A
0.0.0.0/0 -> NAT-A
S3 Prefix  -> S3 Endpoint

Private RT AZ-B
0.0.0.0/0 -> NAT-B
S3 Prefix  -> S3 Endpoint

Private RT AZ-C
0.0.0.0/0 -> NAT-C
S3 Prefix  -> S3 Endpoint
```

The same S3 Gateway Endpoint can be associated with the private route tables for all three AZs.

---

# 7. What Is `vpce-0abc1234`?

`vpce-0abc1234` is not another VPC.

It is the ID of the VPC Endpoint resource.

Example:

```text
VPC
10.0.0.0/16
|
+-- Private Subnet AZ-A
|
+-- Private Subnet AZ-B
|
+-- Private Subnet AZ-C
|
+-- S3 Gateway VPC Endpoint
    ID: vpce-0abc1234
```

Useful ID mapping:

```text
vpc-xxxx     = VPC ID
subnet-xxxx  = Subnet ID
vpce-xxxx    = VPC Endpoint ID
nat-xxxx     = NAT Gateway ID
```

---

# 8. What Is `pl-xxxxxxxx`?

`pl-xxxxxxxx` is an AWS Prefix List ID.

For S3, AWS maintains the IP ranges used by S3 in that region.

Instead of manually adding all S3 CIDRs, AWS represents them with a managed prefix list.

Example:

```text
Destination          Target
--------------------------------
10.0.0.0/16          local
pl-12345678           vpce-0abc1234
0.0.0.0/0            nat-0aaa1111
```

Meaning:

```text
Traffic to VPC CIDR
-> local

Traffic to S3 IP ranges
-> S3 Gateway Endpoint

Everything else
-> NAT Gateway
```

Conceptually:

```text
pl-xxxx
   |
   +--> AWS-managed list of S3 IP ranges
```

and:

```text
vpce-xxxx
   |
   +--> Your VPC Endpoint
```

AWS updates the managed prefix list when the service IP ranges change.

---

# 9. Underlying Mechanism of an S3 Gateway VPC Endpoint

The mechanism is route-table based.

AWS publishes an S3 prefix list that contains the S3 IP ranges for the region.

When a workload sends traffic to S3, the route table performs route matching.

The S3 prefix-list route matches the S3 destination, so the packet is sent to the Gateway VPC Endpoint instead of following the default NAT route.

High-level flow:

```text
EKS Pod / EC2
      |
      v
Private Subnet
      |
      v
Route Table
      |
      | destination matches S3 prefix list
      v
pl-xxxx  ->  vpce-xxxx
      |
      v
S3 Gateway Endpoint
      |
      v
AWS Internal Network
      |
      v
Amazon S3
```

---

# 10. Route-Matching Logic

Example route table:

```text
Destination              Target
-----------------------------------------
10.0.0.0/16              local
pl-S3                     vpce-S3
0.0.0.0/0                NAT Gateway
```

Suppose the application accesses:

```text
s3.ap-south-1.amazonaws.com
```

The destination IP belongs to the AWS-managed S3 prefix list.

Therefore:

```text
Destination IP
      |
      v
Matches pl-S3
      |
      v
Route to vpce-S3
```

The default NAT route is not selected.

---

# 11. Important Difference: Gateway vs Interface Endpoint Mechanism

```text
Gateway Endpoint
= Route-table integration
= No ENI
= No private IP in subnet
```

```text
Interface Endpoint
= ENI + Private IP
= AWS PrivateLink
= ENI created in selected subnet
```

---

# 12. Security / Authorization

Private network connectivity alone does not grant access.

Access still depends on controls such as:

```text
IAM Role / Pod Identity
        +
VPC Endpoint Policy
        +
S3 Bucket Policy
```

So the full request flow is:

```text
DNS Resolution
      ↓
Route Table Matches S3 Prefix List
      ↓
Traffic Routed to Gateway VPC Endpoint
      ↓
AWS Internal Network Carries Traffic to S3
      ↓
IAM / Endpoint Policy / Bucket Policy Evaluated
      ↓
S3 Allows or Denies Request
```

---

# 13. Interview-Ready Answer

> “A VPC Endpoint provides private connectivity from resources inside a VPC to supported AWS services without using a NAT Gateway or Internet Gateway. Gateway endpoints are route-table based and are mainly used for S3 and DynamoDB, while Interface endpoints use PrivateLink and create private ENIs inside selected subnets.”

For the underlying S3 mechanism:

> “An S3 Gateway Endpoint works by adding an AWS-managed S3 prefix-list route to selected VPC route tables. When S3-bound traffic matches that prefix list, the route table sends it to the VPC Endpoint instead of the default NAT route, and AWS carries the traffic privately to S3.”
