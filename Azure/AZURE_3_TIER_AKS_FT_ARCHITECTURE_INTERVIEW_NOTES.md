# Azure 3-Tier AKS Fault-Tolerant Architecture — Interview Notes

## 1. Basic Azure 3-Tier Design

Assume:

```text
VNet: 10.20.0.0/16

AZ-1
AZ-2
AZ-3
```

A clean subnet layout:

```text
Application Gateway Subnet
10.20.1.0/24

AKS Private Subnet
10.20.10.0/22

Database Subnet
10.20.20.0/24

Private Endpoint Subnet
10.20.30.0/24
```

Important Azure difference:

In AWS:

```text
Public subnet -> route to Internet Gateway
```

In Azure:

```text
Public-facing resource
-> Public IP / Front Door / Application Gateway / Public Load Balancer
```

Azure does not use the same public-subnet + Internet Gateway model as AWS.

---

# 2. Basic High-Level Architecture

```text
                         Internet
                            |
                            v
                       Azure DNS
                            |
                            v
                 Application Gateway
                  Public IP + WAF
                            |
                            v
===============================================================
                VNet: 10.20.0.0/16
===============================================================

                Application Gateway Subnet
                     10.20.1.0/24
                            |
                            v

                    AKS Private Subnet
                    10.20.10.0/22

               +------------------------+
               | AKS Worker Nodes       |
               |                        |
               | Node-1   Node-2 Node-3 |
               |   |        |      |    |
               | Pods     Pods    Pods   |
               +-----------+------------+
                           |
                    Kubernetes Service
                           |
                   Application Pods
                           |
                  +--------+---------+
                  |                  |
                  v                  v
             Azure SQL          Storage Account
             / PostgreSQL       / Blob Storage
                  |
                  v
              DB Subnet
             10.20.20.0/24
```

Basic flow:

```text
User
 ↓
Azure DNS
 ↓
Application Gateway
 ↓
AKS Ingress
 ↓
Kubernetes Service
 ↓
Application Pod
 ↓
Database / Storage
```

---

# 3. Where Components Are Placed

| Component | Placement |
|---|---|
| Azure DNS | Azure-managed, outside VNet |
| Azure Front Door | Azure-managed global service |
| Azure WAF | Attached to Front Door/Application Gateway |
| Azure API Management | Azure-managed service; network mode depends on tier/config |
| Application Gateway | Dedicated subnet inside VNet |
| Public IP | Attached to internet-facing App Gateway/LB |
| AKS control plane | Microsoft-managed |
| AKS worker nodes | Private AKS subnet |
| Pods | On AKS worker nodes |
| NAT Gateway | Associated directly with private subnet |
| Private Endpoint | Private IP inside selected subnet |
| Azure SQL/PostgreSQL | Managed service; private connectivity through Private Endpoint or delegated subnet, depending on service |
| Storage Account | Azure-managed, outside VNet |
| Blob private access | Private Endpoint |
| Key Vault private access | Private Endpoint |

---

# 4. Azure NAT Gateway

Azure NAT Gateway placement differs from AWS.

AWS:

```text
Private Subnet
   |
NAT Gateway
in Public Subnet
   |
Internet Gateway
```

Azure:

```text
AKS Private Subnet
       |
       | subnet association
       v
Azure NAT Gateway
       |
       v
Public IP
       |
       v
Internet
```

Example:

```text
AKS Subnet: 10.20.10.0/22
       |
       +-- NAT Gateway
              |
              +-- Public IP
```

Interview answer:

> “In Azure, NAT Gateway is associated directly with the subnet that requires outbound internet connectivity. Unlike AWS, I do not deploy the NAT Gateway in a separate public subnet.”

---

# 5. Private AKS Worker Connectivity

```text
AKS Worker Node
10.20.10.x
      |
      +---- Internal VNet communication
      |
      +---- NAT Gateway -> Internet
      |
      +---- Private Endpoint -> Azure PaaS
```

Example:

```text
AKS Pod
 |
 +--> Database through Private Endpoint
 |
 +--> Blob Storage through Private Endpoint
 |
 +--> Key Vault through Private Endpoint
 |
 +--> Internet through NAT Gateway
```

---

# 6. AKS Control Plane

The AKS control plane is Microsoft-managed.

```text
            Microsoft-managed
             AKS Control Plane
                    |
                    |
          API Server Connectivity
                    |
=================================================
                 Your VNet
=================================================
                    |
              AKS Subnet
                    |
             Worker Nodes
                    |
                  Pods
```

For a private AKS cluster:

```text
kubectl / Worker
       |
       v
Private DNS
       |
       v
Private API Endpoint
       |
       v
AKS Managed Control Plane
```

Inside VNet:

```text
YES:
- Worker nodes
- Pods
- Application Gateway
- Private Endpoints
- Internal Load Balancers

NO:
- Actual AKS control-plane infrastructure
```

---

# 7. Application Ingress

Use:

```text
*.apps.example.com
```

for normal application ingress.

```text
*.apps.example.com
        |
        v
     Azure DNS
        |
        v
Application Gateway
      + WAF
        |
        v
AKS Ingress
        |
        v
Kubernetes Service
        |
        v
      Pods
```

Examples:

```text
portal.apps.example.com
shop.apps.example.com
orders.apps.example.com
```

---

# 8. Application API Traffic

Azure-native API management component:

```text
Azure API Management (APIM)
```

Use:

```text
api.apps.example.com
```

for application/business API traffic.

```text
API Client
    |
    v
 Azure DNS
    |
    v
Azure API Management
    |
Authentication
Rate Limiting
Policies
API Analytics
    |
    v
Private Backend Connectivity
    |
    v
Internal Application Gateway / Load Balancer
    |
    v
AKS Service
    |
    v
API Pods
```

APIM handles business API traffic, not the AKS Kubernetes API.

---

# 9. Three Different Traffic Paths

## Normal Application

```text
*.apps.example.com
        |
    Azure DNS
        |
 Azure Front Door
        |
      WAF
        |
Application Gateway
        |
    AKS Ingress
        |
     Service
        |
       Pod
```

## Business API

```text
api.apps.example.com
        |
    Azure DNS
        |
       APIM
        |
     Policies
        |
Private Backend
        |
Internal LB/App Gateway
        |
   AKS Service
        |
     API Pod
```

## Kubernetes API

```text
kubectl
   |
AKS API Endpoint
   |
AKS Control Plane
   |
Worker Nodes
```

APIM is not involved in Kubernetes control-plane traffic.

---

# 10. Database Connectivity

For Azure SQL:

```text
AKS Pod
   |
   v
AKS Private Subnet
   |
   v
Private Endpoint
10.20.30.x
   |
   v
Azure SQL
```

Depending on the managed database service, Azure may use:
- Private Endpoint
- Delegated subnet / private access model

Interview answer:

> “I keep the database off the public internet and connect AKS to it using Azure private networking.”

Security controls include:
- NSG
- Private DNS
- DB firewall
- Managed Identity where applicable

---

# 11. Blob Storage Connectivity

Azure equivalent to the S3 private access pattern:

```text
AKS Pod
   |
   v
Private DNS
   |
   v
Private Endpoint
10.20.30.10
   |
   v
Azure Blob Storage
```

No internet path is required.

Comparison:

```text
AWS:
EKS -> S3 Gateway Endpoint -> S3

Azure:
AKS -> Private Endpoint -> Blob Storage
```

---

# 12. Key Vault Connectivity

```text
AKS Pod
   |
Managed Identity
   |
Private Endpoint
   |
Azure Key Vault
```

This keeps secret access private.

---

# 13. Basic Integrated Azure Diagram

```text
                              INTERNET
                                 |
                            Azure DNS
                                 |
                                 v
                      Application Gateway
                           Public IP
                              + WAF
                                 |
====================================================================
                    VNet 10.20.0.0/16
====================================================================

                    App Gateway Subnet
                       10.20.1.0/24
                              |
                              v
                       AKS Ingress
                              |
                              v

                        AKS Subnet
                      10.20.10.0/22

            +-------------+-------------+
            |             |             |
          Node-A        Node-B        Node-C
           AZ-1          AZ-2          AZ-3
            |             |             |
          Pods          Pods          Pods
            \             |            /
             +------ Kubernetes ------+
                         Services
                             |
                 +-----------+-----------+
                 |                       |
                 v                       v
           Private Endpoint       Private Endpoint
                 |                       |
                 v                       v
            Azure SQL              Blob Storage


AKS Subnet
    |
    +------ NAT Gateway ------> Internet
```

---

# 14. Fully Integrated Azure Architecture

```text
                               INTERNET / USERS
                                      |
                                  Azure DNS
                           /             |              \
                          /              |               \
                         v               v                v

             *.apps.example.com   api.apps.example.com   AKS API
                     |                    |                 |
                     v                    v                 |
              Azure Front Door           APIM               |
                 CDN + WAF          API Management           |
                     |                    |                  |
                     v                    |                  v
              Application Gateway        |           Private AKS API
                   + WAF                  |              Endpoint
                     |                    |                  |
                     |             Private Backend           |
                     |                    |                  |
=======================================================================
                         VNet 10.20.0.0/16
=======================================================================

       APPLICATION GATEWAY SUBNET
            10.20.1.0/24
                  |
                  v
          Application Gateway
                  |
             AKS Ingress
                  |
                  v

              AKS SUBNET
            10.20.10.0/22

      AZ-1              AZ-2              AZ-3
        |                 |                 |
     Worker-A          Worker-B          Worker-C
        |                 |                 |
       Pods              Pods              Pods
        \                 |                 /
         +--------- Kubernetes -----------+
                       Services
                          |
          +---------------+-------------------+
          |               |                   |
          v               v                   v
      Database         Storage             Key Vault
      Private          Private             Private
      Endpoint         Endpoint            Endpoint
          |               |                   |
          v               v                   v
      Azure SQL       Blob Storage        Key Vault


             AKS Private Subnet
                    |
                    v
               NAT Gateway
                    |
                 Public IP
                    |
                    v
                 Internet
```

---

# 15. API Management Path

```text
API Client
    |
api.apps.example.com
    |
Azure DNS
    |
Azure API Management
    |
JWT / OAuth
Rate Limit
Policies
Transformation
Analytics
    |
Private Backend Connectivity
    |
Internal LB / App Gateway
    |
AKS Service
    |
API Pods
```

APIM is for application/business APIs, not:

```text
kubectl
AKS API server traffic
```

---

# 16. CDN + WAF Path

```text
User
 ↓
Azure DNS
 ↓
Azure Front Door
 ↓
CDN / Edge
 ↓
WAF
 ↓
Application Gateway
 ↓
AKS Ingress
 ↓
Service
 ↓
Pod
```

WAF can be used at:
- Azure Front Door
- Application Gateway

depending on the design.

---

# 17. Fully Private Backend Connectivity

```text
AKS
 |
 +--> SQL          via Private Endpoint
 |
 +--> Blob         via Private Endpoint
 |
 +--> Key Vault    via Private Endpoint
 |
 +--> ACR          via Private Endpoint
 |
 +--> Internet     via NAT Gateway
```

Memory line:

```text
NAT Gateway
= private workload -> external internet

Private Endpoint
= private workload -> Azure PaaS service
```

---

# 18. Private DNS

Private Endpoint connectivity typically requires private DNS.

Example:

```text
AKS Pod
   |
   v
sqlserver.database.windows.net
   |
Private DNS Zone
   |
Resolves to
10.20.30.x
   |
Private Endpoint
   |
Azure SQL
```

Same concept applies to:
- Blob Storage
- Key Vault
- ACR

---

# 19. NSGs

Azure Network Security Groups provide network filtering at subnet/NIC level.

```text
Application Gateway
       |
       v
AKS
       |
       v
DB
```

Example:

```text
AKS Subnet:
Allow required traffic from Application Gateway

Private Endpoint / DB:
Allow only required application traffic
```

Avoid allowing the whole VNet unless required.

---

# 20. Azure Load Balancer vs Application Gateway

## Azure Load Balancer

```text
Layer 4
TCP / UDP
```

Use for:
- Network-level balancing
- Internal service exposure
- High-performance L4 traffic

## Application Gateway

```text
Layer 7
HTTP / HTTPS
```

Use for:
- Host routing
- Path routing
- TLS
- WAF
- Web applications

Rough mapping:

```text
AWS ALB -> Azure Application Gateway
AWS NLB -> Azure Load Balancer
```

---

# 21. Azure Front Door vs Application Gateway

## Azure Front Door

```text
Global
Edge-based
Multi-region
CDN
WAF
Global traffic routing
```

## Application Gateway

```text
Regional
Inside VNet
Layer 7
Backend routing
WAF
```

Typical combination:

```text
Internet
   |
Front Door
   |
Application Gateway
   |
AKS
```

---

# 22. Fault Tolerance

AKS worker nodes across three AZs:

```text
AZ-1 -> worker nodes + pods
AZ-2 -> worker nodes + pods
AZ-3 -> worker nodes + pods
```

Use:
- Multiple node replicas
- Pod topology spread
- Pod anti-affinity
- PodDisruptionBudget

Example:

```yaml
replicas: 3
```

Database:
- Azure SQL / PostgreSQL HA
- Zone redundancy where supported

Ingress:
- Zone-redundant Application Gateway

Outbound:
- Azure NAT Gateway associated with the AKS subnet

---

# 23. AWS vs Azure Memory Mapping

| AWS | Azure |
|---|---|
| VPC | VNet |
| Route 53 | Azure DNS |
| CloudFront | Azure Front Door / CDN |
| AWS WAF | Azure WAF |
| ALB | Application Gateway |
| NLB | Azure Load Balancer |
| API Gateway | API Management |
| EKS | AKS |
| NAT Gateway in public subnet | NAT Gateway associated with subnet |
| S3 | Blob Storage |
| S3 Gateway Endpoint | Private Endpoint for Storage |
| RDS/Aurora | Azure SQL / Azure Database |
| Secrets Manager | Key Vault |
| IAM Role | Managed Identity / Azure RBAC |
| Security Group | NSG |
| ECR | ACR |
| EKS API private endpoint | AKS private API endpoint |

---

# 24. Interview-Ready Summary

> “I would create a /16 VNet and separate the edge, AKS application, and database/private service connectivity into dedicated subnets. Application Gateway is deployed in its own subnet and fronts the private AKS workloads. AKS worker nodes are distributed across three Availability Zones. Azure NAT Gateway is associated with the AKS subnet for outbound internet access; unlike AWS, it does not sit in a separate public subnet.”

> “Normal application traffic goes through Azure DNS, Front Door with WAF, Application Gateway, AKS ingress, service and pods. Business APIs go through Azure API Management and then privately to the AKS backend. AKS applications access Azure SQL, Blob Storage, Key Vault and ACR through Private Endpoints and Private DNS. The AKS Kubernetes API is a separate control-plane path and can be exposed privately for an enterprise cluster.”

Key architecture:

```text
                        Azure DNS
                        /       \
                       /         \
                Front Door       APIM
                  + WAF           |
                     |            |
               App Gateway       Private
                     |            Backend
                     +------+
                            |
                           AKS
                            |
           +----------------+----------------+
           |                |                |
       Azure SQL       Blob Storage       Key Vault
           |                |                |
           +------ Private Endpoints --------+

AKS -> NAT Gateway -> Internet
```
