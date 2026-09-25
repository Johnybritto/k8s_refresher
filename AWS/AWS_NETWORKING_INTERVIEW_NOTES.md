# AWS Networking Interview Notes

## 1. VPC Fundamentals

A VPC is a logically isolated network in AWS.

Example:

```text
VPC CIDR: 10.0.0.0/16
```

Why use a custom VPC:
- Control IP address planning and subnet sizing.
- Separate public, private, and database tiers.
- Design for multi-AZ high availability.
- Control routing, internet access, NAT, and security boundaries.
- Integrate with EKS, peering, Transit Gateway, VPN, Direct Connect, and on-premises networks.

Important:

```text
10.0.0.0/16 -> local
```

The VPC local route provides reachability between subnets inside the same VPC.

---

## 2. Example 3-AZ VPC Design

```text
VPC: 10.0.0.0/16

                         Internet
                            |
                            v
                     Internet Gateway
                            |
          ---------------------------------------
          |                 |                   |
        AZ-a              AZ-b                AZ-c
          |                 |                   |
   Public Subnet      Public Subnet       Public Subnet
   10.0.1.0/24       10.0.2.0/24        10.0.3.0/24
      NAT-A              NAT-B               NAT-C
          |                 |                   |
          v                 v                   v
   Private Subnet     Private Subnet      Private Subnet
   10.0.11.0/24      10.0.12.0/24       10.0.13.0/24
      App/EKS            App/EKS             App/EKS
          |                 |                   |
          +-----------------+-------------------+
                            |
                       VPC local routing
                            |
          ---------------------------------------
          |                 |                   |
      DB Subnet         DB Subnet          DB Subnet
   10.0.21.0/24      10.0.22.0/24       10.0.23.0/24
```

A subnet belongs to one Availability Zone only.

AWS reserves 5 IP addresses per subnet.

---

## 3. What Makes a Subnet Public or Private?

### Public subnet

A subnet is public when its route table has a default route to an Internet Gateway:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

Typical resources:
- Public ALB
- NAT Gateway
- Bastion host, if required

A route to the IGW is the key routing characteristic. A directly internet-reachable EC2 instance also needs a public or Elastic IP and suitable Security Group rules.

### Private subnet

A private subnet has no direct route to the IGW.

Typical route table:

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> NAT Gateway
```

Typical resources:
- EKS worker nodes
- EC2 application servers
- Internal services

### Database subnet

Usually isolated:

```text
10.0.0.0/16 -> local
```

Normally no default internet route.

---

## 4. Internet Gateway

An Internet Gateway connects a VPC to the internet.

```text
Internet
   |
   v
Internet Gateway
   |
   v
Public Subnet
```

For an EC2 instance to be directly reachable from the internet, normally all of these are needed:
- Public subnet
- Route to Internet Gateway
- Public IP or Elastic IP
- Security Group allowing the traffic

---

## 5. NAT Gateway

A NAT Gateway allows private resources to initiate outbound internet connections without becoming directly internet reachable.

```text
Private EC2 / EKS
      |
      v
Private Route Table
0.0.0.0/0 -> NAT Gateway
      |
      v
NAT Gateway in Public Subnet
      |
      v
Internet Gateway
      |
      v
Internet
```

Key points:
- NAT Gateway is deployed in a public subnet.
- It uses an Elastic IP for IPv4 internet access.
- A NAT Gateway is created in one AZ through the subnet it belongs to.
- For production HA, commonly deploy one NAT Gateway per AZ.
- Route each private subnet to the NAT Gateway in the same AZ.
- This avoids cross-AZ dependency and unnecessary cross-AZ traffic.

Example:

```text
Private subnet AZ-a -> NAT-A
Private subnet AZ-b -> NAT-B
Private subnet AZ-c -> NAT-C
```

The NAT Gateway itself does not have its own route table. It uses the route table associated with its public subnet.

---

## 6. Route Tables

### Public subnet

```text
Destination      Target
10.0.0.0/16      local
0.0.0.0/0        IGW
```

### Private subnet - AZ-a

```text
Destination      Target
10.0.0.0/16      local
0.0.0.0/0        NAT-A
```

### Private subnet - AZ-b

```text
Destination      Target
10.0.0.0/16      local
0.0.0.0/0        NAT-B
```

### Private subnet - AZ-c

```text
Destination      Target
10.0.0.0/16      local
0.0.0.0/0        NAT-C
```

### DB subnet

```text
Destination      Target
10.0.0.0/16      local
```

Important interview line:

> Route tables provide reachability. Security Groups and NACLs determine whether traffic is allowed.

---

## 7. Security Groups

Security Groups are stateful firewalls associated with ENIs/resources.

Example:

```text
Internet
   |
   | 443
   v
ALB-SG
   |
   | 8080
   v
APP-SG
   |
   | 5432
   v
DB-SG
```

Best practice:
- Prefer SG-to-SG references over broad CIDRs.

Example:

```text
DB-SG inbound
TCP 5432
Source = APP-SG
```

Instead of:

```text
Source = 10.0.0.0/16
```

Security Groups:
- Stateful
- Allow rules only
- Attached to ENIs/resources
- Return traffic for an allowed connection is automatically permitted

---

## 8. NACLs

Network ACLs apply at the subnet level.

| Security Group | NACL |
|---|---|
| Resource/ENI level | Subnet level |
| Stateful | Stateless |
| Allow only | Allow and deny |
| Return traffic automatic | Return path rules required |
| Primary workload control | Additional subnet boundary |

Interview answer:

> I normally use Security Groups as the primary workload firewall and NACLs when subnet-level explicit allow/deny filtering is required.

---

## 9. Private Subnet to DB Subnet Communication

Private and DB subnets communicate through the VPC local route.

```text
App: 10.0.11.10
      |
      | 10.0.0.0/16 -> local
      v
DB: 10.0.21.20:5432
```

NAT Gateway and Internet Gateway are not involved.

Security controls the traffic:

```text
DB-SG:
Allow TCP 5432
Source = APP-SG
```

Important interview line:

> The local route provides reachability, but reachability does not mean access.

---

## 10. How to Restrict Traffic When the Local Route Reaches All Subnets

The VPC local route cannot normally be deleted to selectively block one subnet from another.

Restrict traffic using:
- Security Groups
- NACLs
- AWS Network Firewall
- Separate VPCs where hard isolation is required

For databases, prefer:

```text
DB-SG allows 5432 only from APP-SG
```

Not:

```text
DB-SG allows 5432 from 10.0.0.0/16
```

---

## 11. Application Traffic Flow

### Internet to application to DB

```text
Internet User
    |
    v
Internet Gateway
    |
    v
Public ALB
    |
    v
Private App / EKS
    |
    v
Database Subnet
```

### Private workload outbound

```text
Private App / EKS
    |
    v
Private Route Table
0.0.0.0/0 -> NAT
    |
    v
Same-AZ NAT Gateway
    |
    v
Internet Gateway
    |
    v
Internet
```

---

## 12. ALB vs NLB

### ALB

Application Load Balancer:
- Layer 7
- HTTP/HTTPS
- Host-based routing
- Path-based routing
- Common for web applications and EKS ingress

### NLB

Network Load Balancer:
- Layer 4
- TCP/UDP/TLS
- Very high performance
- Static IP support
- Useful when source IP preservation or non-HTTP protocols are needed

---

## 13. Public vs Internal Load Balancer

### Internet-facing load balancer

- Uses public subnets
- Reachable from the internet
- Front-end path goes through Internet Gateway

### Internal load balancer

- Uses private IPs
- Reachable only from the VPC or connected networks

---

## 14. Route 53 and DNS

Route 53 provides DNS services.

### Public Hosted Zone

Used for internet-facing DNS names.

### Private Hosted Zone

Used for DNS names that should resolve only inside associated VPCs.

### Route 53 Resolver

Used for:
- VPC DNS resolution
- Hybrid DNS integration
- Forwarding DNS queries between AWS and on-premises networks

Useful VPC settings:
- enableDnsSupport
- enableDnsHostnames

---

## 15. VPC Endpoints

A VPC Endpoint lets workloads in a VPC access supported AWS services privately, without needing to go through the public internet, Internet Gateway, or NAT Gateway.

### Without VPC endpoint

```text
Private EC2 / EKS
      |
      v
NAT Gateway
      |
      v
Internet Gateway
      |
      v
AWS Service
```

### With VPC endpoint

```text
Private EC2 / EKS
      |
      v
VPC Endpoint
      |
      v
AWS Service
```

### Gateway Endpoints

Commonly used for:
- S3
- DynamoDB

They work through route tables.

Example:

```text
10.0.0.0/16 -> local
S3 prefix list -> VPC Gateway Endpoint
0.0.0.0/0   -> NAT
```

S3 traffic uses the endpoint, while other internet traffic can still use NAT.

### Interface Endpoints

Powered by AWS PrivateLink.

They create ENIs with private IP addresses inside selected subnets.

Common examples:
- ECR
- STS
- Secrets Manager
- CloudWatch
- EC2 APIs
- Many other AWS services

Typical flow:

```text
EKS Node
   |
   v
Private DNS
   |
   v
Interface Endpoint ENI
   |
   v
AWS PrivateLink
   |
   v
AWS Service
```

Benefits:
- Private connectivity
- Reduced NAT dependence
- Better control over service access
- Traffic remains on AWS networking

---

## 16. VPC Peering

VPC Peering provides direct private connectivity between two VPCs.

```text
VPC-A <------> VPC-B
```

Key points:
- CIDRs must not overlap.
- Route tables must be configured on both sides.
- Security Groups/NACLs must allow traffic.
- VPC peering is not transitive.

```text
A <-> B
B <-> C

does NOT mean

A <-> C
```

---

## 17. Transit Gateway

Transit Gateway is a central routing hub for multiple VPCs and hybrid networks.

```text
        VPC-A
          |
VPC-B -- TGW -- VPC-C
          |
       On-Prem
```

Use it for:
- Many VPCs
- Hub-and-spoke designs
- VPN
- Direct Connect
- Centralized routing and inspection

---

## 18. VPN vs Direct Connect

### Site-to-Site VPN

- Encrypted tunnel over the internet
- Faster to deploy
- Useful for backup or lower-throughput hybrid connectivity

### Direct Connect

- Dedicated private connectivity
- More predictable bandwidth/latency
- Common for enterprise hybrid networking

Common pattern:

```text
On-Prem
   |
Direct Connect
   |
Transit Gateway
   |
Multiple VPCs

Backup:
Site-to-Site VPN
```

---

## 19. BGP Basics

BGP dynamically exchanges routes between AWS and external networks.

Common with:
- Direct Connect
- Site-to-Site VPN

It reduces dependency on manually maintained static routes.

---

## 20. EKS Networking Basics

With the AWS VPC CNI:
- EKS worker nodes use ENIs in your VPC.
- Pods receive VPC-routable IP addresses.
- Pod density is influenced by ENI and IP limits.
- Subnet IP exhaustion must be considered.

Common mitigations:
- Larger private subnets
- Secondary VPC CIDRs
- Prefix delegation
- Monitoring available subnet IPs

---

## 21. EKS Ingress Traffic Flow

Conceptual flow:

```text
Internet
   |
   v
Route 53
   |
   v
ALB
   |
   v
Ingress Controller / Kubernetes routing
   |
   v
Kubernetes Service
   |
   v
Pod
```

---

## 22. How the EKS Control Plane Connects to EC2 Worker Nodes

The EKS control plane is managed by AWS and runs outside the customer-managed worker-node VPC environment.

EKS creates managed network interfaces in the cluster subnets selected during cluster creation. These provide network connectivity between the managed control plane and resources in the VPC.

Conceptually:

```text
AWS Managed EKS Control Plane
--------------------------------
API Server
Scheduler
Controller Manager
etcd
        |
        v
EKS-managed network interfaces
================================
Customer VPC
        |
        v
Private Subnets
        |
        v
EC2 Worker Nodes
        |
        v
Pods
```

Node registration flow:

```text
EC2 worker starts
    |
    v
Bootstrap runs
    |
    v
kubelet starts
    |
    v
kubelet connects to EKS API server
    |
    v
Node registers
    |
    v
Node sends heartbeats/status and receives desired state
```

The control plane also needs connectivity to kubelet on the worker nodes for operations that require it.

---

## 23. EKS API Endpoint Modes

### Public endpoint

The Kubernetes API endpoint is internet-reachable subject to endpoint access restrictions.

Private worker nodes need a path to reach the public endpoint, normally via outbound connectivity.

### Public + Private

Workers inside the VPC can use private endpoint connectivity, while approved administrators may still use the public endpoint.

### Private only

The Kubernetes API is reachable only from the VPC or connected private networks such as:
- VPN
- Direct Connect
- Transit Gateway-connected networks

Important distinction:

> An EKS Interface VPC Endpoint for the AWS EKS service API is not the same as the Kubernetes API private endpoint.

The service interface endpoint is for AWS API operations such as:
- describe-cluster
- list-clusters

The Kubernetes API endpoint is separately configured on the EKS cluster.

---

## 24. Centralized Egress

In larger environments, internet egress can be centralized using:
- Transit Gateway
- Central networking/inspection VPC
- NAT Gateways
- AWS Network Firewall

Conceptually:

```text
Application VPCs
      |
      v
Transit Gateway
      |
      v
Inspection / Egress VPC
      |
      v
Network Firewall
      |
      v
NAT Gateway
      |
      v
Internet Gateway
```

---

## 25. AWS Network Firewall

AWS Network Firewall is a managed stateful firewall service.

Use cases:
- Centralized traffic inspection
- Stateful filtering
- IPS-style rules
- Domain filtering
- Egress controls
- East-west or north-south inspection designs

---

## 26. VPC Flow Logs

VPC Flow Logs provide metadata about network flows.

Useful fields include:
- Source IP
- Destination IP
- Source port
- Destination port
- Protocol
- ACCEPT / REJECT

They are useful for investigating:
- Security Group problems
- NACL problems
- Routing issues
- Unexpected network flows

---

# Interview Troubleshooting Sequence

When an application cannot connect:

```text
DNS
 ↓
Route Table
 ↓
Security Group
 ↓
NACL
 ↓
Load Balancer / Target Health
 ↓
VPC Flow Logs
 ↓
Application listening port
```

---

# Interview Q&A

## Q1. What makes a subnet public or private?

A public subnet has a default route to an Internet Gateway:

```text
0.0.0.0/0 -> IGW
```

A private subnet has no direct IGW route and usually sends default outbound traffic to a NAT Gateway:

```text
0.0.0.0/0 -> NAT Gateway
```

---

## Q2. How do you make a subnet public?

Create or associate a route table containing:

```text
0.0.0.0/0 -> Internet Gateway
```

A directly internet-reachable instance must also have a public/Elastic IP and suitable Security Group rules.

---

## Q3. How do you make a subnet private?

Do not add a direct route to the Internet Gateway.

For outbound internet access:

```text
0.0.0.0/0 -> NAT Gateway
```

---

## Q4. Why does NAT Gateway need to be in a public subnet?

Because the NAT Gateway must itself reach the Internet Gateway.

Its public subnet has:

```text
0.0.0.0/0 -> IGW
```

The private subnet points to the NAT.

---

## Q5. Is NAT Gateway AZ-specific?

Yes. A NAT Gateway is created in a subnet, and that subnet belongs to one AZ.

For production HA:

```text
AZ-a Private -> NAT-A
AZ-b Private -> NAT-B
AZ-c Private -> NAT-C
```

---

## Q6. What if one NAT Gateway or AZ fails?

Private workloads that depend on that NAT may lose outbound access.

Using one NAT Gateway per AZ limits the failure scope and avoids making all private subnets depend on one AZ.

---

## Q7. What routes are required in the public subnet?

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> Internet Gateway
```

---

## Q8. What routes are required in the private subnet?

```text
10.0.0.0/16 -> local
0.0.0.0/0   -> NAT Gateway
```

---

## Q9. Does the NAT Gateway need its own route table?

No.

The NAT Gateway is in a public subnet and uses that subnet's route table.

---

## Q10. How does the private subnet communicate with the DB subnet?

Through the VPC local route:

```text
10.0.0.0/16 -> local
```

The DB Security Group should allow only the required application Security Group and port.

---

## Q11. If the local route can reach any subnet, how do you restrict access?

Use Security Groups as the primary control.

For example:

```text
DB-SG:
Allow TCP 5432
Source = APP-SG
```

For subnet-level controls, use NACLs. For stronger isolation, use separate VPCs or centralized firewall inspection.

---

## Q12. Security Group vs NACL?

Security Group:
- Stateful
- Resource/ENI level
- Allow rules

NACL:
- Stateless
- Subnet level
- Allow and deny rules

---

## Q13. NAT Gateway vs Internet Gateway?

Internet Gateway:
- Provides internet connectivity for public resources.

NAT Gateway:
- Allows private resources to initiate outbound connections without becoming directly internet reachable.

---

## Q14. Why one NAT Gateway per AZ?

For:
- High availability
- Failure-domain isolation
- Avoiding cross-AZ dependency
- Avoiding unnecessary cross-AZ traffic

---

## Q15. How do private EKS nodes access S3 or ECR without using the internet?

Use VPC Endpoints.

Examples:
- S3 -> Gateway Endpoint
- ECR -> Interface Endpoints / PrivateLink

---

## Q16. What is a VPC Endpoint?

A VPC Endpoint provides private access from a VPC to supported AWS services without requiring the workload to traverse NAT or the public internet.

---

## Q17. Gateway Endpoint vs Interface Endpoint?

Gateway Endpoint:
- Route-table based
- Primarily S3 and DynamoDB

Interface Endpoint:
- PrivateLink based
- Creates ENIs with private IPs
- Used for many AWS services

---

## Q18. VPC Peering vs Transit Gateway?

VPC Peering:
- Direct VPC-to-VPC connection
- Non-transitive
- Good for a small number of VPCs

Transit Gateway:
- Central routing hub
- Scales to many VPCs and hybrid connections
- Supports hub-and-spoke designs

---

## Q19. ALB vs NLB?

ALB:
- Layer 7
- HTTP/HTTPS
- Host/path routing

NLB:
- Layer 4
- TCP/UDP/TLS
- High throughput
- Static IP options

---

## Q20. How does EKS control plane communicate with EC2 worker nodes?

The AWS-managed EKS control plane connects to the worker-node VPC using EKS-managed network interfaces created in the selected cluster subnets.

The kubelet on each worker node connects to the Kubernetes API server to register the node, send health/status, and receive desired state.

---

## Q21. Does the EKS control plane run in my VPC?

No. The control plane is AWS-managed.

Your worker nodes and pods run in your VPC.

Connectivity is provided through EKS-managed networking into the selected cluster subnets.

---

## Q22. What is the difference between an EKS Interface Endpoint and the Kubernetes API private endpoint?

An EKS Interface Endpoint provides private access to AWS EKS service APIs such as:

```text
aws eks describe-cluster
aws eks list-clusters
```

The Kubernetes API private endpoint is the private endpoint used by kubectl, kubelet, controllers, and Kubernetes clients.

They are different things.

---

## Q23. What are the common EKS API endpoint modes?

- Public
- Public + Private
- Private only

Private-only clusters require administrative access through the VPC or connected private networks.

---

## Q24. How do you troubleshoot AWS network connectivity?

Use this order:

```text
DNS
-> Route Table
-> Security Group
-> NACL
-> Load Balancer / Target Health
-> VPC Flow Logs
-> Application Port
```

---

# High-Priority Topics to Master Before Interview

1. VPC CIDR and subnet planning
2. Public vs private subnet
3. Internet Gateway
4. NAT Gateway and one-NAT-per-AZ design
5. Route tables and local routing
6. Security Group vs NACL
7. Full ALB -> private app -> DB traffic flow
8. VPC Endpoints / PrivateLink
9. VPC Peering vs Transit Gateway
10. VPN vs Direct Connect
11. ALB vs NLB
12. Route 53 and private DNS
13. EKS VPC CNI
14. EKS pod IP exhaustion
15. EKS control-plane-to-worker connectivity
16. EKS public/private API endpoint modes
17. VPC Flow Logs and troubleshooting
18. Centralized egress and AWS Network Firewall
