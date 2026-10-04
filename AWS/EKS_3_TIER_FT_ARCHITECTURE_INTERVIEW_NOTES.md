# EKS 3-Tier Fault-Tolerant Architecture — Interview Notes

This note consolidates the 3-tier EKS design discussed for interview preparation, including:

- VPC and subnet layout
- Public, private, and DB tiers
- NAT Gateway placement
- EKS worker nodes and control-plane connectivity
- EKS-managed ENIs
- Route 53
- Application ingress
- Application API traffic through API Gateway / Apigee
- S3 private connectivity
- Database connectivity
- CloudFront CDN
- AWS WAF
- Traffic-flow diagrams
- Interview-ready explanations

---

# 1. Core Design

## VPC

```text
VPC CIDR: 10.0.0.0/16
Architecture: 3 Availability Zones
```

Example:

```text
AZ-A
AZ-B
AZ-C
```

The goal is to avoid a single-AZ dependency and provide fault-tolerant application capacity.

---

# 2. Subnet Design

## Public Subnets

```text
AZ-A: 10.0.1.0/24
AZ-B: 10.0.2.0/24
AZ-C: 10.0.3.0/24
```

Used for:
- Internet-facing ALB
- NAT Gateway in each AZ

## Private Application / EKS Subnets

```text
AZ-A: 10.0.11.0/24
AZ-B: 10.0.12.0/24
AZ-C: 10.0.13.0/24
```

Used for:
- EKS worker nodes
- Application pods
- Kubernetes services
- EKS-managed cluster ENIs

## DB Subnets

```text
AZ-A: 10.0.21.0/24
AZ-B: 10.0.22.0/24
AZ-C: 10.0.23.0/24
```

Used for:
- RDS
- Aurora
- Database subnet group

Normally the DB tier has no direct internet route.

---

# 3. High-Level Diagram — Base 3-Tier EKS Architecture

```text
                               Internet
                                  |
                              Route 53
                                  |
                          *.apps.example.com
                                  |
                                  v
                          Internet-facing ALB
                         (spans public subnets)
                                  |
                                  v
                          Kubernetes Ingress
                                  |
                                  v
                          Kubernetes Service
                                  |
      =================================================================
      |                     VPC 10.0.0.0/16                            |
      |                                                                |
      |    AZ-A              AZ-B              AZ-C                    |
      |                                                                |
      | PUBLIC             PUBLIC             PUBLIC                    |
      | 10.0.1.0/24       10.0.2.0/24       10.0.3.0/24               |
      |                                                                |
      | NAT GW-A          NAT GW-B          NAT GW-C                    |
      |      |                 |                 |                     |
      |------|-----------------|-----------------|---------------------|
      |      |                 |                 |                     |
      | PRIVATE APP       PRIVATE APP       PRIVATE APP                 |
      | 10.0.11.0/24     10.0.12.0/24     10.0.13.0/24                |
      |                                                                |
      | Worker-A          Worker-B          Worker-C                    |
      |   Pods              Pods              Pods                     |
      |      \                |                /                       |
      |       +------ Kubernetes Services -----+                       |
      |                     |                                         |
      |             +-------+--------+                                |
      |             |                |                                |
      |             v                v                                |
      |      S3 Connectivity      DB Connectivity                     |
      |                                                                |
      | DB SUBNET          DB SUBNET          DB SUBNET                 |
      | 10.0.21.0/24      10.0.22.0/24      10.0.23.0/24              |
      |                                                                |
      |              RDS / Aurora Multi-AZ                            |
      =================================================================
```

### Explanation

- Public ALB accepts application traffic.
- EKS workers and pods stay in private subnets.
- NAT Gateways provide outbound internet access for private workloads when required.
- RDS/Aurora stays in DB subnets.
- Application-to-DB traffic uses VPC-local routing.
- S3 traffic should preferably use an S3 Gateway VPC Endpoint rather than NAT.

---

# 4. Route Tables

## Public Subnet Route Table

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

## Private App Subnet Route Table

Each private subnet routes to the NAT Gateway in the same AZ.

Example AZ-A:

```text
10.0.0.0/16  -> local
0.0.0.0/0    -> NAT Gateway-A
S3 PrefixList -> S3 Gateway Endpoint
```

AZ-B:

```text
0.0.0.0/0 -> NAT Gateway-B
```

AZ-C:

```text
0.0.0.0/0 -> NAT Gateway-C
```

## DB Subnet Route Table

```text
10.0.0.0/16 -> local
```

Normally no default internet route.

---

# 5. NAT Gateway Placement

Deploy one NAT Gateway per AZ for production fault tolerance.

```text
Private AZ-A -> NAT-A
Private AZ-B -> NAT-B
Private AZ-C -> NAT-C
```

Each NAT Gateway is placed in the public subnet of the same AZ.

### Why

- Avoids single-AZ dependency
- Reduces cross-AZ traffic
- Keeps other AZs functional if one NAT/AZ fails

### Important

NAT is not needed for communication between subnets in the same VPC.

```text
Private Subnet -> Public Subnet
       |
       +--> VPC local route
```

NAT is needed for:

```text
Private Subnet -> Internet
```

---

# 6. EKS Control Plane and Worker Nodes

The EKS control plane is AWS-managed.

It does not run on the worker nodes and the actual control-plane infrastructure is not deployed inside the customer VPC.

AWS creates EKS-managed network interfaces in the subnets selected for the cluster.

## Diagram — EKS Control Plane Connectivity

```text
              AWS-managed EKS Control Plane
        -----------------------------------------
        | API Server / etcd / controllers       |
        | scheduler                              |
        -----------------------------------------
                       |
                       v
                 EKS-managed ENIs
              /         |         \
             /          |          \
      Private AZ-A  Private AZ-B  Private AZ-C
      10.0.11/24    10.0.12/24    10.0.13/24
          |             |             |
       Worker-A       Worker-B       Worker-C
          |             |             |
         Pods          Pods          Pods
```

### Cluster subnet selection

For this 3-AZ design, provide:

```text
subnet-private-a
subnet-private-b
subnet-private-c
```

during EKS cluster creation.

These are the subnets EKS can use for managed cluster ENIs.

The same three private subnets can also be used for the managed node groups.

### Interview answer

> “For my 3-AZ EKS cluster, I provide three private subnet IDs during cluster creation, one per AZ. EKS uses those selected subnets for its managed network interfaces, while the worker nodes are also launched across the three private subnets.”

---

# 7. EKS API Traffic vs Application API Traffic

This distinction is critical.

There are two different meanings of the word API.

## A. Kubernetes / EKS API

Used by:
- kubectl
- kubelet
- Kubernetes controllers
- cluster administration

```text
kubectl / kubelet
      |
      v
AWS-provided EKS API Endpoint
      |
      v
AWS-managed EKS Control Plane
      |
      v
EKS-managed ENIs
      |
      v
Worker Nodes
```

Amazon API Gateway is not used in this path.

## B. Application / Business API

Examples:
- /orders
- /payments
- /users
- /inventory

This is where API Gateway or Apigee belongs.

```text
API Client
   |
   v
API Gateway / Apigee
   |
   v
Application backend in EKS
```

---

# 8. About `api` and `*.apps` DNS Names

In OpenShift, it is common to think in terms of:

```text
api.cluster.example.com
*.apps.cluster.example.com
```

In EKS, the mapping is different.

## Application Ingress

Use:

```text
*.apps.example.com
```

for application ingress.

Examples:

```text
shop.apps.example.com
portal.apps.example.com
orders.apps.example.com
```

## Kubernetes API

EKS already provides an AWS-managed Kubernetes API endpoint.

For interview purposes, do not treat EKS as if it has a self-managed OpenShift-style API VIP.

Use:

```text
AWS-provided EKS API endpoint
```

for control-plane traffic.

## Application API Domain

If the application has business APIs, use something such as:

```text
api.apps.example.com
```

and route that traffic through Amazon API Gateway or Apigee.

---

# 9. High-Level Diagram — Application API Gateway Integration

```text
                         USERS / CLIENTS
                               |
                            Route 53
                         /            \
                        /              \
             *.apps.example.com    api.apps.example.com
                     |                    |
                     v                    v
               Public ALB          Amazon API Gateway
                     |                    |
                     |                 VPC Link
                     |                    |
                     |                    v
                     |              Internal ALB/NLB
                     |                    |
                     v                    v
                  Ingress           Kubernetes Service
                     |                    |
                     v                    v
              Kubernetes Service        API Pods
                     |
                     v
                  App Pods


                     SEPARATE CONTROL-PLANE PATH

                   kubectl / kubelet
                         |
                         v
                  EKS API Endpoint
                         |
                         v
               AWS-managed Control Plane
                         |
                         v
                 EKS-managed ENIs
                         |
                         v
                    Worker Nodes
```

### Explanation

- `*.apps.example.com` is normal web/application ingress.
- `api.apps.example.com` is business API traffic.
- Business API traffic can go through API Gateway.
- API Gateway reaches private EKS backends through VPC Link.
- Kubernetes control-plane traffic uses the EKS API endpoint and does not go through API Gateway.

---

# 10. Where Does API Gateway Sit?

Amazon API Gateway is an AWS-managed regional service.

It does not sit inside your VPC subnet.

```text
Internet / Client
       |
       v
Amazon API Gateway
(AWS-managed service)
       |
       | VPC Link
       v
================ YOUR VPC ================
       |
Internal ALB / NLB
       |
Kubernetes Service
       |
API Pods
```

For private APIs, an Interface VPC Endpoint can provide private access to the API Gateway service.

The API Gateway service itself remains AWS-managed.

---

# 11. Apigee Placement

If the organization uses Apigee instead of Amazon API Gateway:

```text
Client
  |
  v
Apigee
  |
Private / Hybrid Connectivity
  |
================ AWS VPC ================
  |
Internal ALB / NLB
  |
EKS Service
  |
API Pods
```

Use Apigee for:
- API policies
- Authentication/authorization
- Rate limiting
- Quotas
- API analytics
- API lifecycle management

Use Amazon API Gateway for the AWS-native equivalent pattern.

---

# 12. S3 Connectivity from EKS

Amazon S3 itself is not inside the VPC.

For private EKS workloads, use an S3 Gateway VPC Endpoint.

## Diagram — S3 Private Access

```text
EKS Pod / Worker
      |
      v
Private Subnet
      |
      v
Private Subnet Route Table
      |
      | S3 Prefix List
      v
S3 Gateway VPC Endpoint
      |
      v
Amazon S3
```

### Important

The S3 Gateway Endpoint is not placed inside a subnet as an ENI.

Instead:
- Create it at the VPC level.
- Associate it with the route tables used by the private subnets.

Example private route table:

```text
10.0.0.0/16   -> local
S3 PrefixList -> S3 Gateway Endpoint
0.0.0.0/0     -> NAT Gateway
```

Therefore:

```text
S3 traffic      -> VPC Endpoint
Internet traffic -> NAT Gateway
```

No NAT or Internet Gateway is required for S3 traffic through the Gateway Endpoint.

---

# 13. Application-to-Database Connectivity

Application pods communicate with the DB over VPC-local routing.

```text
Application Pod
      |
      v
Private App Subnet
      |
10.0.0.0/16 -> local
      |
      v
DB Subnet
      |
      v
RDS / Aurora
```

NAT Gateway and Internet Gateway are not involved.

### Security

Example:

```text
DB-SG
Allow TCP 5432
Source = Application Security Group
```

or for MySQL:

```text
TCP 3306
Source = Application Security Group
```

---

# 14. Complete Application Traffic Flows

## Web Application Traffic

```text
User
 |
Route 53
 |
*.apps.example.com
 |
Public ALB
 |
Ingress Controller
 |
Kubernetes Service
 |
Application Pod
```

## Business API Traffic

```text
API Client
 |
Route 53
 |
api.apps.example.com
 |
Amazon API Gateway / Apigee
 |
VPC Link / Private Connectivity
 |
Internal ALB/NLB
 |
Kubernetes Service
 |
API Pod
```

## Kubernetes Control-Plane Traffic

```text
kubectl / kubelet
 |
EKS API Endpoint
 |
AWS-managed EKS Control Plane
 |
EKS-managed ENIs
 |
Worker Nodes
```

## Application-to-S3

```text
Application Pod
 |
IAM Role / Pod Identity
 |
S3 Gateway Endpoint
 |
Amazon S3
```

## Application-to-DB

```text
Application Pod
 |
VPC Local Routing
 |
RDS / Aurora
```

## Private Workload-to-Internet

```text
Worker / Pod
 |
0.0.0.0/0
 |
Same-AZ NAT Gateway
 |
Internet Gateway
 |
Internet
```

---

# 15. CloudFront CDN and AWS WAF Placement

CloudFront and AWS WAF are AWS-managed services and do not sit inside application subnets.

CloudFront is the CDN.

AWS WAF is associated with supported resources such as:
- CloudFront
- API Gateway
- ALB

WAF is not a subnet-level network appliance.

---

# 16. High-Level Diagram — CDN + WAF + API Gateway + EKS

```text
                           INTERNET / USERS
                                  |
                               Route 53
                           /        |        \
                          /         |         \
                         v          v          v

             *.apps.example.com  api.apps.example.com   EKS API Endpoint
                     |                   |                     |
                     v                   v                     |
                CloudFront         API Gateway                 |
                   CDN                  |                      |
                     |                AWS WAF                   |
                  AWS WAF               |                      |
                     |               VPC Link                   |
                     v                  |                       v
              Internet-facing ALB       v                AWS-managed EKS
                     |            Internal ALB/NLB        Control Plane
                     |                  |                       |
============================= VPC 10.0.0.0/16 =============================
                     |                  |                       |
                     v                  v                EKS-managed ENIs
                 Ingress          Kubernetes Service           |
                     |                  |                       |
                     v                  v                       v
             Kubernetes Service      API Pods              Worker Nodes
                     |
                     v
                  App Pods
                     |
             +-------+--------+
             |                |
             v                v
      S3 Gateway Endpoint   DB Connectivity
             |                |
             v                v
         Amazon S3       RDS / Aurora
```

---

# 17. Web Traffic with CloudFront and WAF

```text
User
 ↓
Route 53
 ↓
*.apps.example.com
 ↓
CloudFront CDN
 ↓
AWS WAF
 ↓
Internet-facing ALB
 ↓
Ingress
 ↓
Kubernetes Service
 ↓
Pod
```

CloudFront provides:
- Edge caching
- Reduced latency
- Content distribution
- TLS integration
- Origin protection patterns

WAF provides:
- SQL injection protection
- XSS protection
- IP blocking
- Geo restrictions
- Rate-based rules
- Bot protection

---

# 18. API Traffic with WAF

```text
API Client
 ↓
Route 53
 ↓
api.apps.example.com
 ↓
API Gateway
 ↓
AWS WAF
 ↓
VPC Link
 ↓
Internal ALB/NLB
 ↓
Kubernetes Service
 ↓
API Pod
```

API Gateway provides:
- Authentication
- Authorization
- Throttling
- API policies
- Request/response handling
- API observability

WAF provides request filtering and web attack protection.

---

# 19. Where Each Component Sits

| Component | Placement |
|---|---|
| Route 53 | AWS-managed global/regional DNS service, outside VPC subnets |
| CloudFront | AWS-managed edge/CDN service, outside VPC |
| AWS WAF | Associated with CloudFront/API Gateway/ALB, not placed in subnet |
| API Gateway | AWS-managed service, outside customer VPC |
| Internet Gateway | Attached to VPC |
| Public ALB | Spans public subnets across AZs |
| NAT Gateway | Public subnet, one per AZ |
| Internal ALB/NLB | Private subnets |
| EKS Control Plane | AWS-managed |
| EKS-managed ENIs | Selected private cluster subnets |
| EKS Worker Nodes | Private app subnets |
| Pods | Run on EKS worker nodes |
| S3 Gateway Endpoint | VPC-level endpoint associated with private route tables |
| Amazon S3 | AWS regional managed service, outside VPC |
| RDS / Aurora | DB subnets |
| DB subnet group | DB subnets across multiple AZs |

---

# 20. Fault-Tolerance Controls

For this architecture:

## Compute

```text
Worker-A -> AZ-A
Worker-B -> AZ-B
Worker-C -> AZ-C
```

## Application

Use:
- Multiple replicas
- Topology Spread Constraints
- Pod Anti-Affinity
- PodDisruptionBudget

Example:

```yaml
replicas: 3
```

## Ingress

ALB spans multiple AZs.

## NAT

One NAT Gateway per AZ.

## Database

Use:
- RDS Multi-AZ
- Aurora Multi-AZ / appropriate cluster topology

## DNS / Edge

Use:
- Route 53
- CloudFront
- WAF

---

# 21. Three-Tier Mapping

## Tier 1 — Presentation / Ingress

Components:
- Route 53
- CloudFront
- AWS WAF
- Internet-facing ALB
- API Gateway for business APIs

## Tier 2 — Application

Components:
- EKS worker nodes
- Kubernetes ingress
- Kubernetes services
- Application pods
- API pods

## Tier 3 — Data

Components:
- RDS / Aurora
- S3 where object storage is needed
- DB subnets
- S3 Gateway VPC Endpoint for private S3 access

---

# 22. Interview-Ready Summary

> “I would create a /16 VPC and split it into public, private application, and database subnets across three Availability Zones. The internet-facing ALB and one NAT Gateway per AZ are associated with the public tier. EKS worker nodes, pods, and EKS-managed cluster ENIs are in private subnets. RDS or Aurora uses dedicated DB subnets. Normal application traffic enters through Route 53, CloudFront and WAF, then the ALB, ingress, service and pod. Business API traffic is kept separate and goes through API Gateway or Apigee, then private connectivity/VPC Link to an internal load balancer and EKS API service. Kubernetes control-plane traffic uses the AWS-managed EKS API endpoint directly and does not go through API Gateway. Applications access S3 privately through an S3 Gateway Endpoint and access the database over VPC-local routing. NAT is only required for outbound internet access from private workloads.”
