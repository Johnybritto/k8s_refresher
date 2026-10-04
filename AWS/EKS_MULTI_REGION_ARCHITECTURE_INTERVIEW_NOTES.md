# Multi-Region EKS Architecture — Interview Notes

## 1. Design Goal

Build a **multi-region EKS architecture** where:

- Each AWS region has its own independent EKS cluster.
- Each regional EKS cluster spans multiple Availability Zones.
- Route 53 controls regional application traffic.
- Region A is the primary region.
- Region B is the DR / warm-standby region.
- The database is replicated across regions.
- S3 data is replicated across regions.
- Application APIs and Kubernetes control-plane APIs remain separate.

---

# 2. High-Level Multi-Region Architecture

```text
                                 USERS
                                   |
                               Route 53
                    Failover Routing + Health
                          /                 \
                         /                   \
                        v                     v
              PRIMARY - Region A      SECONDARY - Region B
                  us-east-1               us-west-2

              VPC 10.10.0.0/16        VPC 10.20.0.0/16
              =================        =================

              Public Subnets           Public Subnets
              10.10.1.0/24             10.20.1.0/24
              10.10.2.0/24             10.20.2.0/24
              10.10.3.0/24             10.20.3.0/24
                   |                         |
              ALB + NAT GW               ALB + NAT GW
                   |                         |
                   v                         v
              Private EKS              Private EKS
              10.10.11.0/24            10.20.11.0/24
              10.10.12.0/24            10.20.12.0/24
              10.10.13.0/24            10.20.13.0/24
                   |                         |
              Worker Nodes              Worker Nodes
              Pods / Services           Pods / Services
                   |                         |
                   +-----------+-------------+
                               |
                        Application Data
                               |
                Aurora Global Database
                       /               \
                      /                 \
                PRIMARY Writer       Secondary
                  Region A           Region B
```

Each region must be independently resilient across multiple Availability Zones.

---

# 3. Regional VPC Layout

## Region A

```text
VPC: 10.10.0.0/16

AZ-A              AZ-B              AZ-C
 |                  |                 |
Public             Public            Public
10.10.1/24         10.10.2/24       10.10.3/24
 NAT-A              NAT-B             NAT-C
   |                  |                 |
Private            Private           Private
10.10.11/24        10.10.12/24      10.10.13/24
 Worker-A           Worker-B          Worker-C
   |                  |                 |
  Pods               Pods              Pods

DB subnets:
10.10.21/24
10.10.22/24
10.10.23/24
```

## Region B

```text
VPC: 10.20.0.0/16

AZ-A              AZ-B              AZ-C
 |                  |                 |
Public             Public            Public
10.20.1/24         10.20.2/24       10.20.3/24
 NAT-A              NAT-B             NAT-C
   |                  |                 |
Private            Private           Private
10.20.11/24        10.20.12/24      10.20.13/24
 Worker-A           Worker-B          Worker-C
   |                  |                 |
  Pods               Pods              Pods

DB subnets:
10.20.21/24
10.20.22/24
10.20.23/24
```

Use non-overlapping VPC CIDRs so future inter-region connectivity through Transit Gateway or VPC peering remains possible.

---

# 4. Fully Integrated Diagram

```text
                              INTERNET / CLIENTS
                                      |
                                  Route 53
                       Health Check + Failover Policy
                       /                        \
                      /                          \
                     v                            v

             REGION A - PRIMARY             REGION B - DR
               us-east-1                     us-west-2
                    |                            |
          +---------+---------+        +---------+---------+
          |                   |        |                   |
          v                   v        v                   v
    *.apps.example.com   api.apps   *.apps.example.com  api.apps
          |                   |        |                   |
          v                   v        v                   v
      Public ALB        API Gateway Public ALB        API Gateway
          |                   |        |                   |
          |               VPC Link     |               VPC Link
          |                   |        |                   |
          v                   v        v                   v
      EKS Ingress        Internal LB EKS Ingress       Internal LB
          |                   |        |                   |
          v                   v        v                   v
       Service            API Svc    Service             API Svc
          |                   |        |                   |
          v                   v        v                   v
       App Pods            API Pods   App Pods           API Pods
          |                   |        |                   |
          +---------+---------+        +---------+---------+
                    |                            |
             S3 Gateway Endpoint          S3 Gateway Endpoint
                    |                            |
                    v                            v
               Regional S3                 Regional S3

                    \                            /
                     \                          /
                      +---- Aurora Global -----+
                           Database
                          /        \
                         /          \
                  Writer Region A   Secondary Region B
```

The EKS control planes remain independent.

```text
AWS-managed EKS CP-A                AWS-managed EKS CP-B
        |                                   |
 Managed ENIs A                       Managed ENIs B
        |                                   |
 EKS Workers A                        EKS Workers B
```

There is **no single EKS control plane spanning multiple AWS regions**.

---

# 5. Route 53 Records

## Application Ingress

Create two records with the same name:

```text
*.apps.example.com
```

### Primary record

```text
Type: A/AAAA Alias
Routing Policy: Failover
Failover Type: PRIMARY
Target: Region-A ALB
Evaluate Target Health: YES
```

### Secondary record

```text
Type: A/AAAA Alias
Routing Policy: Failover
Failover Type: SECONDARY
Target: Region-B ALB
Evaluate Target Health: YES
```

Normal behavior:

```text
*.apps.example.com
       |
       v
Region-A ALB
```

If Region A becomes unhealthy:

```text
*.apps.example.com
       |
       v
Region-B ALB
```

---

# 6. Business API Records

Use:

```text
api.apps.example.com
```

Create:

```text
PRIMARY
api.apps.example.com
 -> Region-A API Gateway

SECONDARY
api.apps.example.com
 -> Region-B API Gateway
```

Conceptually:

```text
api.apps.example.com
        |
     Route 53
        |
   Failover policy
     /       \
Region-A    Region-B
API GW      API GW
```

---

# 7. EKS Kubernetes API

Do not hide two independent EKS clusters behind one common Route 53 failover record.

Each EKS cluster has its own API endpoint.

```text
EKS Cluster A
-> its own API endpoint

EKS Cluster B
-> its own API endpoint
```

Optional internal naming:

```text
eks-use1.internal.example.com
eks-usw2.internal.example.com
```

Administrators deliberately choose the intended cluster.

---

# 8. Normal Application Traffic Flow

While Region A is healthy:

```text
User
 ↓
Route 53
 ↓
*.apps.example.com
 ↓
PRIMARY Record
 ↓
Region-A ALB
 ↓
Ingress
 ↓
Service
 ↓
EKS Pods
 ↓
Aurora Writer - Region A
```

Region B stays ready:

```text
Region B
EKS + ALB + DB Replica
        |
     Standby
```

---

# 9. Business API Traffic Flow

```text
Client
 ↓
api.apps.example.com
 ↓
Route 53
 ↓
Region-A API Gateway
 ↓
VPC Link
 ↓
Internal ALB/NLB
 ↓
EKS Service
 ↓
API Pod
 ↓
Aurora Writer
```

If Region A is unhealthy, Route 53 directs new DNS resolutions to Region B.

---

# 10. S3 Multi-Region Design

Each cluster should normally access the S3 service in its own region through a Gateway VPC Endpoint.

## Region A

```text
EKS Pod
 ↓
S3 Gateway Endpoint
 ↓
S3 Bucket - Region A
```

## Region B

```text
EKS Pod
 ↓
S3 Gateway Endpoint
 ↓
S3 Bucket - Region B
```

For critical replicated objects:

```text
S3 Region A
     |
Cross-Region Replication
     |
     v
S3 Region B
```

S3 traffic through the Gateway Endpoint does not require NAT.

---

# 11. Database Multi-Region Design

Example using Aurora Global Database:

```text
Region A
Aurora Writer
     |
 replication
     v
Region B
Aurora Secondary
```

Normal state:

```text
Region A = Writer
Region B = Secondary
```

During a regional outage:

```text
Region A Failure
      |
      v
Promote Region-B Database
      |
      v
Region B becomes Writer
```

The data failover must be coordinated with application traffic failover.

---

# 12. Regional Failure Flow

```text
Region A Failure
       |
       v
ALB / API Health Fails
       |
       v
Route 53 Detects Unhealthy PRIMARY
       |
       +-----------------------------+
       |                             |
       v                             v
Promote DB in Region B         Route Traffic to
to New Writer                  Region-B Endpoints
       |                             |
       +-------------+---------------+
                     |
                     v
             Region-B EKS
                     |
                     v
                Application
```

A practical failover sequence:

```text
1. Detect regional failure
2. Validate Region-B application readiness
3. Promote Region-B database
4. Confirm dependencies
5. Route application/API traffic to Region B
6. Validate user transactions
7. Monitor application and data health
```

---

# 13. Route 53 Alone Is Not Enough

Route 53 can redirect application traffic:

```text
Region A
   ↓
Region B
```

It does not automatically:
- Promote the database
- Scale Region-B EKS
- Synchronize S3
- Restore queues
- Rebuild caches
- Validate external dependencies

The DR process must coordinate these components.

---

# 14. Database Connection After Failover

Avoid hardcoding regional database endpoints in application code.

Conceptually:

```text
Application
     |
     v
Database Writer Endpoint
     |
     +--> Region A before failover
     |
     +--> Region B after failover
```

The objective is to minimize application configuration changes during DR.

---

# 15. Route 53 Records — Quick Table

| DNS Name | Region A | Region B | Routing |
|---|---|---|---|
| `*.apps.example.com` | ALB-A | ALB-B | Failover |
| `api.apps.example.com` | API Gateway-A | API Gateway-B | Failover |
| EKS Kubernetes API | Cluster-A Endpoint | Cluster-B Endpoint | Separate |
| Database | Primary writer | Secondary / promoted writer | Database failover mechanism |

---

# 16. Active-Active Alternative

Instead of warm standby:

```text
PRIMARY / SECONDARY
```

you can deploy both regions actively.

Possible Route 53 strategies:
- Latency-based routing
- Weighted routing
- Geolocation / geoproximity where required
- Health-aware routing

Example:

```text
                    Route 53
                  Latency Based
                   /         \
                  /           \
             Region A       Region B
                |              |
              EKS            EKS
```

The data layer becomes the difficult part.

For true multi-region active-active writes, a data platform designed for that model, such as DynamoDB Global Tables, may be easier than a single-primary relational database design.

---

# 17. Interview Q&A

## Q1. Why use two EKS clusters instead of one EKS cluster stretched across regions?

EKS is regional. A single EKS control plane does not span AWS regions. Therefore I deploy an independent cluster in each region and deploy the same applications to both.

---

## Q2. How does Route 53 know Region A has failed?

Use failover routing with health evaluation. The regional endpoint is monitored, and when the primary is unhealthy Route 53 returns the secondary endpoint for new DNS resolutions.

---

## Q3. Is Route 53 failover instantaneous?

No. Route 53 is DNS based, so failover depends on health detection, DNS TTL, and resolver/client caching.

For stricter failover requirements, AWS Global Accelerator can also be considered.

---

## Q4. What happens to existing connections during failover?

Existing TCP or HTTP sessions are not automatically moved to Region B.

They can fail and must reconnect. New DNS resolutions are sent to the healthy region.

---

## Q5. Why not use only Global Accelerator?

Global Accelerator can provide static anycast IPs and network-level global traffic steering.

Route 53 provides DNS-based routing.

The choice depends on:
- Failure requirements
- Static-IP requirements
- Latency
- Protocol
- Cost

They can also be used together in some architectures.

---

## Q6. What happens to the database when Region A fails?

The secondary regional database must be promoted to writer before or as application traffic moves to the DR region.

For Aurora Global Database, Region B normally runs as a secondary and can be promoted during regional failure.

---

## Q7. Can Route 53 switch traffic before the DB is ready?

Yes.

That is why the failover process should coordinate database promotion and application traffic routing. Sending users to Region B before its data tier is ready can create application failures.

---

## Q8. How does the application know which database to use after failover?

Do not hardcode a regional database endpoint in application code.

Use an abstracted database endpoint/configuration strategy so applications connect to the current writer after failover.

---

## Q9. What is the RTO and RPO?

They depend on the DR pattern.

Warm standby:
- Relatively low RTO
- Relatively low RPO

Active-active:
- Lowest RTO/RPO
- Highest complexity and cost

The business requirement should define the target RTO/RPO first.

---

## Q10. How is S3 handled across regions?

Use Cross-Region Replication for critical data.

Each EKS cluster accesses its regional S3 endpoint privately through an S3 Gateway VPC Endpoint.

---

## Q11. What happens if S3 replication is behind during failover?

S3 Cross-Region Replication is asynchronous, so replication lag may exist.

That lag must be included in the RPO and application recovery design.

---

## Q12. How do you keep application deployments identical in both regions?

Use:
- Git
- CI/CD
- Argo CD / Flux
- Helm
- Kustomize

Example:

```text
Git
 |
 +--> EKS Region A
 |
 +--> EKS Region B
```

Region-specific configuration can be stored as overlays or values.

---

## Q13. How do you manage secrets across regions?

Use a controlled secrets strategy, for example:
- AWS Secrets Manager
- KMS
- Infrastructure as Code
- Replicated or independently provisioned regional secrets

Do not manually copy Kubernetes Secrets between clusters.

---

## Q14. What happens if only one AZ fails?

Do not fail over the entire region.

The local EKS cluster should continue running across the other healthy Availability Zones.

Regional failover should occur only when the regional application or its critical dependencies become unavailable.

---

## Q15. How much capacity should the standby region have?

Enough to run critical services and meet the agreed RTO.

For warm standby:
- Run reduced capacity
- Keep minimum viable application capacity online
- Scale quickly during DR

Avoid relying entirely on scale-from-zero for critical applications if a low RTO is required.

---

## Q16. How do you scale Region B during failover?

Use:
- EKS Managed Node Groups
- Karpenter
- Cluster Autoscaler
- HPA

Maintain enough pre-existing capacity so failover does not wait entirely for new nodes.

---

## Q17. Do NAT Gateways need to exist in both regions?

Yes, if private workloads require internet egress.

Each region must remain independently functional.

In a 3-AZ architecture, private subnets normally use a same-AZ NAT Gateway where required.

---

## Q18. Why should VPC CIDRs be different?

Non-overlapping CIDRs make:
- VPC peering
- Transit Gateway
- Hybrid connectivity
- Cross-region communication

much easier.

Example:

```text
Region A: 10.10.0.0/16
Region B: 10.20.0.0/16
```

---

## Q19. Do EKS clusters need direct connectivity to each other?

Not necessarily.

They can remain independently functional.

If private cross-region communication is needed, options include:
- Transit Gateway inter-region peering
- VPC peering
- Other private connectivity patterns

---

## Q20. How do you know the DR region is actually ready?

Monitor application readiness, not just infrastructure.

Check:
- ALB health
- EKS node health
- Pod readiness
- Synthetic application/API tests
- Database replication lag
- S3 replication status
- External dependencies
- Queue health
- Required secrets/configuration

---

## Q21. How do you test DR?

Perform planned DR drills.

Example:

```text
Trigger/Simulate Failure
      ↓
Validate Region B
      ↓
Promote Database
      ↓
Switch Traffic
      ↓
Test Transactions
      ↓
Measure RTO/RPO
      ↓
Fail Back Safely
```

---

## Q22. How do you fail back to Region A?

Do not immediately switch traffic back.

First:
1. Verify Region A infrastructure
2. Resynchronize data
3. Validate database state
4. Deploy/validate application
5. Test dependencies
6. Switch database roles if required
7. Move traffic back
8. Monitor

---

## Q23. What happens to user sessions?

Avoid storing sessions locally in pods.

Prefer:
- Stateless applications
- Tokens
- External replicated/session stores where necessary

Otherwise users may lose sessions during regional failover.

---

## Q24. What about Redis / ElastiCache?

ElastiCache is regional.

The cache should generally not be the system of record.

In DR:
- Build a separate regional cache
- Replicate only when needed and supported
- Allow the DR application to rebuild cache from persistent data

---

## Q25. What about queues such as SQS?

SQS is regional.

A multi-region application may require:
- Independent regional queues
- Event replication
- Application-level routing
- Idempotent processing

Messaging is one of the areas that must be explicitly designed for DR.

---

## Q26. What happens if Region B is also unhealthy?

Route 53 cannot make an unhealthy secondary region healthy.

Both regional endpoints need monitoring.

If both are unhealthy, the problem must be handled by:
- Additional region/site
- Static degraded-mode service
- Operational recovery

depending on business requirements.

---

## Q27. Why choose warm standby instead of active-active?

Warm standby:
- Lower cost
- Simpler database design
- Easier operations
- Good RTO/RPO for many enterprise workloads

Active-active introduces more complexity around:
- Writes
- Consistency
- Sessions
- Messaging
- Conflict resolution
- Cost

---

## Q28. When would you choose active-active?

When the business requires:
- Very low RTO
- Global low latency
- Continuous service across regional failures

and is willing to accept the additional architecture and operational complexity.

---

## Q29. Which AWS data service is easier for true multi-region active-active?

DynamoDB Global Tables are designed for multi-region active-active access.

Relational databases typically require more careful write-primary and consistency design.

---

## Q30. What is the hardest part of multi-region EKS?

Usually not Kubernetes.

The difficult parts are:
- Database/state
- Messaging
- Caches
- S3/object consistency
- External integrations
- Capacity
- DNS/traffic routing
- Failback
- Operational testing

---

# 18. Strong Interview Summary

> “I deploy one independent EKS cluster in each region, and each cluster spans three Availability Zones. Route 53 uses primary and secondary failover records for application ingress and business APIs. Region A handles normal production traffic while Region B runs as warm standby. The database is replicated across regions and must be promoted in Region B before or during traffic failover. S3 uses Cross-Region Replication, while each cluster accesses S3 privately through a regional Gateway VPC Endpoint. I do not place the two EKS Kubernetes API endpoints behind one common failover record because the clusters are independent. The real complexity of multi-region EKS is not Kubernetes itself; it is state, database consistency, messaging, sessions, external dependencies, capacity, and controlled failover/failback.”
