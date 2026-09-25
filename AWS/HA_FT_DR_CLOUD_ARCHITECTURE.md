# HA, FT & DR in Cloud Architecture — AWS & Azure Interview Notes

> Source basis: consolidated from the uploaded HA/FT/DR document, with additional AWS and Azure implementation examples added for interview preparation.

---

# 1. High Availability (HA)

High Availability focuses on minimizing downtime and keeping services accessible when some components fail.

The main mechanism is **redundancy**:
- multiple instances
- multiple Availability Zones
- automated health checks
- automated failover

The key objective is to remove single points of failure.


## Types of High Availability

HA can be implemented in different ways depending on where redundancy is introduced.

### 1. Active-Passive HA

One instance or site actively serves traffic while another remains on standby.

```text
Active Server  ---> serving traffic
Passive Server ---> standby

Failure
   ↓
Passive becomes Active
```

**Typical use cases**
- Primary/standby databases
- Some appliance or middleware clusters
- RDS Multi-AZ style failover patterns

**Advantages**
- Simpler architecture
- Lower cost than full active-active

**Disadvantages**
- Small failover delay may occur
- Standby capacity may remain underutilized

### 2. Active-Active HA

Multiple healthy instances serve traffic at the same time.

```text
          Load Balancer
          /          \
      Server-A      Server-B
       Active        Active
```

If one fails, the remaining instances continue serving traffic.

**AWS examples**
- ALB + EC2 across AZs
- EKS replicas across AZs

**Azure examples**
- Application Gateway / Load Balancer + VMSS or AKS

### 3. Multi-AZ HA

Workloads are distributed across Availability Zones within the same region.

```text
Region
 |
 +-- AZ-A -> App
 |
 +-- AZ-B -> App
 |
 +-- AZ-C -> App
```

This protects mainly against:
- instance failure
- rack/data-center failure
- AZ failure

This is the most common cloud HA pattern.

### 4. Multi-Region / Global HA

The application is deployed in multiple regions.

```text
Global DNS / Traffic Manager
          |
     -------------
     |           |
 Region-A     Region-B
```

**AWS**
- Route 53
- Global Accelerator

**Azure**
- Front Door
- Traffic Manager

This provides stronger resilience but costs more and begins to overlap with DR and fault-tolerant designs.

### 5. Application-Level HA

The application itself runs multiple replicas.

```text
ALB
 |
 +-- App-1
 +-- App-2
 +-- App-3
```

For Kubernetes:

```text
Deployment replicas: 3
Pod-1 -> AZ-A
Pod-2 -> AZ-B
Pod-3 -> AZ-C
```

### 6. Database HA

The database uses primary/standby or replica-based redundancy.

**AWS**
- RDS Multi-AZ
- Aurora replicas

**Azure**
- Azure SQL zone redundancy
- Azure Database for PostgreSQL HA

### 7. Network HA

Network paths and devices are also redundant.

Examples:
- Load balancers across AZs
- One NAT Gateway per AZ
- Multiple VPN tunnels
- Multiple Direct Connect / ExpressRoute paths
- Redundant firewalls and routers

**Interview answer**

> “HA can be active-passive or active-active, and it can be implemented at the application, database, network, AZ, or region level. In cloud environments, Multi-AZ HA is the most common baseline.”

---

## Core idea

```text
One component fails
       ↓
Another healthy component takes over
       ↓
Minimal service interruption
```

## Typical architecture

```text
                    Users
                      |
                Load Balancer
                 /         \
                /           \
             AZ-1           AZ-2
           App-1           App-2
                \           /
                 \         /
                  Database
                 Multi-AZ
```

---

## HA in AWS

Typical AWS services and components:

- Multiple Availability Zones
- Application Load Balancer (ALB)
- Network Load Balancer (NLB)
- Auto Scaling Groups
- EC2 across multiple AZs
- EKS node groups across multiple AZs
- RDS Multi-AZ
- Aurora Multi-AZ
- ElastiCache replicas
- Route 53 health checks
- S3
- Multiple NAT Gateways

### Example

```text
Route 53
   |
ALB
 |
 +------ EC2 / EKS AZ-a
 |
 +------ EC2 / EKS AZ-b
 |
 +------ EC2 / EKS AZ-c
          |
          v
      RDS Multi-AZ
```

If an application instance fails:

```text
ALB Health Check
      ↓
Marks instance unhealthy
      ↓
Stops sending traffic
      ↓
Traffic continues to healthy targets
```

### Interview answer

> “For high availability in AWS, I distribute workloads across multiple Availability Zones, put an ALB or NLB in front, use Auto Scaling or EKS node groups across zones, and use an HA database such as RDS Multi-AZ. This removes single points of failure and minimizes downtime.”

---

## HA in Azure

Typical Azure services and components:

- Availability Zones
- Virtual Machine Scale Sets
- Azure Load Balancer
- Application Gateway
- Azure Front Door
- AKS across Availability Zones
- Azure SQL zone redundancy
- Azure Database for PostgreSQL HA
- Azure Storage ZRS
- Traffic Manager

### Example

```text
Azure Front Door
       |
Application Gateway
       |
-------------------------
|                       |
Zone-1                  Zone-2
VM / AKS                VM / AKS
   \                     /
    \                   /
     Azure SQL / DB
     Zone Redundant
```

### Interview answer

> “In Azure, I use Availability Zones, zone-aware VMSS or AKS, Application Gateway or Load Balancer, and a zone-redundant database. The design goal is the same: remove single points of failure and keep the service running when a component or zone fails.”

---

# 2. Fault Tolerance (FT)

Fault Tolerance goes further than HA.

The objective is to continue service with **near-zero or zero visible interruption** even when components fail.

FT normally requires:
- deeper redundancy
- real-time or near-real-time replication
- active-active components
- automatic failure masking


## Types / Patterns of Fault Tolerance

The original source explains FT as extensive redundancy plus real-time replication so that failures are masked from users. The following are common implementation patterns used in practice.

### 1. Active-Active Fault Tolerance

Multiple systems are active and processing traffic simultaneously.

```text
             Traffic
               |
        ----------------
        |              |
     Active-A       Active-B
```

If one component fails, the others continue immediately.

**Examples**
- Multi-AZ application replicas
- Multi-region active-active applications
- DynamoDB Global Tables
- Cosmos DB multi-region writes

### 2. Synchronized Active-Standby

A standby remains continuously synchronized with the active system and takes over immediately when failure is detected.

```text
Active
  |
  | continuous replication
  v
Standby
```

This approaches FT when the failover is seamless and data loss is effectively zero or near-zero.

### 3. Component-Level Fault Tolerance

Individual infrastructure components are redundant.

Examples:
- Redundant disks
- Multiple NICs
- Multiple power supplies
- Multiple network paths
- Redundant database nodes
- Redundant Kubernetes nodes

### 4. Multi-AZ Fault Tolerance

Enough active capacity exists across AZs so the application continues even if an entire AZ fails.

```text
AZ-A  Active
AZ-B  Active
AZ-C  Active
```

### 5. Multi-Region Fault Tolerance

Multiple regions actively serve users.

```text
Global Routing
     |
-------------------
|                 |
Region-A        Region-B
Active          Active
```

This is expensive but provides protection beyond a single region.

**Interview answer**

> “Fault tolerance is commonly implemented using active-active systems, synchronized standby, component-level redundancy, Multi-AZ designs, or Multi-Region active-active designs. The key difference from HA is that FT tries to mask the failure so the user sees no interruption.”

---

## HA vs FT

```text
HA:
Failure occurs
→ failover happens
→ small interruption may occur

FT:
Failure occurs
→ redundant component continues immediately
→ user ideally notices nothing
```

## Typical FT design

```text
                    Traffic
                      |
                 Load Balancer
                /           \
               /             \
        Active System     Active System
            AZ-A              AZ-B
               \              /
                \            /
                Real-Time
                Replication
```

---

## Fault Tolerance in AWS

Typical AWS components:

- Multi-AZ active-active application instances
- EKS across multiple AZs
- Global Accelerator
- Route 53
- Aurora replicas
- Aurora Global Database
- DynamoDB Global Tables
- S3
- SQS / Kinesis for decoupling
- Multiple NAT Gateways
- Redundant Direct Connect / VPN paths
- Multi-region architecture

### Multi-region example

```text
             AWS Global Accelerator
                     |
            ---------------------
            |                   |
         Region A            Region B
            |                   |
           ALB                 ALB
         /     \             /     \
       AZ-a   AZ-b         AZ-a   AZ-b
```

### Database examples

```text
Aurora
 |
 +-- Writer
 +-- Reader AZ-a
 +-- Reader AZ-b
```

For higher regional resilience:

```text
Aurora Global Database
```

or:

```text
DynamoDB Global Tables
```

### Interview answer

> “Fault tolerance goes beyond HA. In HA, failover may cause a small interruption. In a fault-tolerant design, the failure is effectively masked from the user using active-active redundancy, synchronous or near-real-time replication, and multiple independent components.”

---

## Fault Tolerance in Azure

Typical Azure components:

- Zone-redundant VM / AKS
- Azure Front Door
- Application Gateway
- Azure Load Balancer
- Cosmos DB multi-region writes
- Azure SQL failover groups
- Geo-replicated storage
- Service Bus
- Event Grid

### Example

```text
Azure Front Door
       |
---------------------
|                   |
Region A           Region B
AKS                AKS
 |                  |
Cosmos DB Multi-Region
```

### Interview answer

> “In Azure, a fault-tolerant design can use Front Door across regions, AKS across zones or regions, and a globally replicated database such as Cosmos DB. The goal is to keep serving traffic without a noticeable interruption.”

---

# 3. Disaster Recovery (DR)

Disaster Recovery is used to recover services after a major event that affects an entire data center, Availability Zone group, or region.

Typical events:
- Region outage
- Data center loss
- Major network outage
- Natural disaster
- Severe data corruption
- Large-scale infrastructure failure

## Core idea

```text
Primary Region
      |
      | replication / backup
      v
Secondary Region
```

If the primary fails:

```text
Secondary Region
      ↓
Becomes active
```

DR is mainly about:
- business continuity
- preserving data
- restoring services within agreed recovery objectives

---


## Types of Disaster Recovery

The standard cloud DR patterns are selected mainly based on **RTO, RPO, and cost**.

### 1. Backup and Restore

Backups are stored separately and infrastructure is rebuilt or restored after a disaster.

```text
Primary
   |
 Backup
   |
DR Storage
```

- Lowest cost
- Highest RTO
- Higher RPO compared with continuous replication

### 2. Pilot Light

Only the most critical core components remain running in the DR region.

```text
Primary Region        DR Region
Full Stack            DB / Core only
Running               Running
```

During disaster, the remaining application capacity is started or scaled.

- Lower cost than warm standby
- Medium RTO
- Low-to-medium RPO

### 3. Warm Standby

A smaller but complete working environment always runs in the DR region.

```text
Primary Region        DR Region
Full Capacity         Reduced Capacity
Running               Running
```

During disaster, scale up the DR environment and redirect traffic.

- Medium/high cost
- Low RTO
- Low RPO

### 4. Multi-Site Active-Active

Both regions actively serve production traffic.

```text
          Global Routing
          /            \
     Region-A        Region-B
      Active          Active
```

- Highest cost
- Lowest RTO
- Lowest RPO

### Quick memory

```text
Backup & Restore  -> cheapest, slowest
Pilot Light       -> critical core only
Warm Standby      -> smaller running environment
Active-Active     -> both regions live
```

**Interview answer**

> “The main DR patterns are Backup and Restore, Pilot Light, Warm Standby, and Multi-Site Active-Active. The choice is driven by the required RTO, RPO, business criticality, and cost.”

---

# 4. RTO and RPO

## RTO — Recovery Time Objective

How long can the service remain unavailable?

Example:

```text
RTO = 30 minutes
```

Meaning:

> Service must be restored within 30 minutes.

## RPO — Recovery Point Objective

How much data loss can the business tolerate?

Example:

```text
RPO = 5 minutes
```

Meaning:

> The business can tolerate losing approximately 5 minutes of data.

Easy memory:

```text
RTO = Time to Recover
RPO = Amount of Data You Can Lose
```

---

# 5. AWS Disaster Recovery Strategies

## A. Backup and Restore

The lowest-cost DR approach.

### Architecture

```text
Primary Region
     |
   Backup
     |
S3 / AWS Backup
     |
Secondary Region
```

During disaster:

```text
Restore Infrastructure
      ↓
Restore Database
      ↓
Start Application
      ↓
Update DNS / Traffic
```

### AWS components

- AWS Backup
- S3
- S3 Cross-Region Replication
- EBS snapshots
- RDS snapshots
- AMIs
- Terraform
- CloudFormation

### Characteristics

```text
Cost: Low
RTO: High
RPO: High compared with active replication
```

### Best for

- Non-critical workloads
- Cost-sensitive systems
- Applications that can tolerate longer recovery

---

## B. Pilot Light

Only the critical core is continuously running in the DR region.

### Example

```text
Primary Region
-------------------
App
DB
Cache
Services

DR Region
-------------------
DB Replica          running
Core Services       minimal
App Capacity        stopped / minimal
```

During disaster:

```text
Scale Application
      ↓
Start Services
      ↓
Promote Database
      ↓
Switch Traffic
```

### AWS components

- RDS cross-region replica
- Aurora Global Database
- DynamoDB Global Tables
- S3 replication
- Route 53
- Auto Scaling
- Terraform

### Characteristics

```text
Cost: Medium-Low
RTO: Medium
RPO: Low to Medium
```

---

## C. Warm Standby

A smaller but fully functional copy of production is always running.

### Example

```text
Primary Region
--------------
10 App Servers
Full DB Capacity

DR Region
--------------
2 App Servers
Replica DB
```

During disaster:

```text
Scale DR Environment
      ↓
Promote Database
      ↓
Redirect Traffic
```

### AWS components

- ALB
- EC2 / EKS
- Auto Scaling
- Aurora / RDS replicas
- Route 53
- Global Accelerator
- S3 Cross-Region Replication

### Characteristics

```text
Cost: Medium to High
RTO: Low
RPO: Low
```

---

## D. Multi-Site Active-Active

Both regions actively serve production traffic.

### Example

```text
             Route 53
                 |
         Global Accelerator
          /             \
         /               \
   Region A             Region B
      ALB                  ALB
     EKS/EC2             EKS/EC2
       |                   |
       ----- Global DB -----
```

### AWS components

- Route 53
- Global Accelerator
- ALB / NLB
- EKS / EC2
- Aurora Global Database
- DynamoDB Global Tables
- S3 replication

### Characteristics

```text
Cost: Highest
RTO: Near Zero
RPO: Near Zero
```

---

# 6. Azure DR Equivalents

## Backup and Restore

Typical components:

- Azure Backup
- Recovery Services Vault
- Blob Storage
- Managed Disk snapshots

## Pilot Light / Warm Standby

Typical components:

- Azure Site Recovery
- Azure SQL geo-replication
- Azure Storage GRS / GZRS
- Azure Database replicas
- AKS deployed at reduced capacity
- Terraform / Bicep

## Active-Active

Typical components:

- Azure Front Door
- Traffic Manager
- Cosmos DB multi-region
- Azure SQL Failover Groups
- Geo-redundant Storage
- Multi-region AKS

### Example

```text
Azure Front Door
       |
---------------------
|                   |
Region A           Region B
AKS                AKS
 |                  |
Cosmos DB / SQL
Geo Replication
```

---

# 7. HA vs FT vs DR

| Concept | Main Goal | Typical Failure Scope | Downtime | Cost |
|---|---|---|---|---|
| HA | Minimize downtime | Instance / AZ | Very low | Medium |
| FT | Continue with no visible interruption | Component / AZ / sometimes Region | Near zero | High |
| DR | Recover from major disaster | Region / Site | Depends on RTO | Varies |

Important:

> A fault-tolerant system is highly available, but a highly available system is not necessarily fault tolerant.

---

# 8. Full AWS Resilient Architecture Example

## HA within one region

```text
                      Route 53
                          |
                  Global Accelerator
                          |
                        ALB
                  /              \
                 /                \
              AZ-A                AZ-B
               |                    |
          EKS / EC2            EKS / EC2
               |                    |
               +---------+----------+
                         |
                   Aurora Multi-AZ
                         |
                   Redis Replicas
```

## DR across regions

```text
                       Route 53
                    Failover Routing
                     /             \
                    /               \
            Primary Region      DR Region
                 |                  |
                ALB                ALB
                 |                  |
               EKS                EKS
                 |                  |
                 ----- DB Replication
```

---

# 9. Full Azure Resilient Architecture Example

## HA

```text
                  Azure Front Door
                        |
               Application Gateway
                 /              \
                /                \
          Zone 1                Zone 2
          AKS / VM             AKS / VM
                \               /
                 \             /
                  Azure SQL
                Zone Redundant
```

## DR

```text
                 Azure Front Door
                       |
          --------------------------
          |                        |
      Region A                  Region B
        AKS                       AKS
          \                       /
           \                     /
           Cosmos DB / SQL
            Geo Replication
```

---

# 10. Kubernetes HA

## AWS EKS

```text
EKS Cluster
 |
 +-- Node Group AZ-a
 |
 +-- Node Group AZ-b
 |
 +-- Node Group AZ-c
```

Use multiple application replicas:

```yaml
replicas: 3
```

Additional controls:

- Pod Anti-Affinity
- Topology Spread Constraints
- PodDisruptionBudget
- Multi-AZ node groups
- Multi-AZ load balancers

Goal:

```text
Pod-1 -> AZ-a
Pod-2 -> AZ-b
Pod-3 -> AZ-c
```

If one AZ fails, replicas in other AZs continue serving traffic.

## Azure AKS

Use:

- Availability Zones
- Multiple node pools
- Topology Spread Constraints
- Pod Anti-Affinity
- PodDisruptionBudget
- Application Gateway / Azure Load Balancer

---

# 11. Database HA vs DR

This is a common interview question.

## AWS

### HA

```text
RDS Multi-AZ
Aurora Multi-AZ
```

Protects mainly against:
- instance failure
- storage failure
- AZ failure

### DR

```text
RDS Cross-Region Replica
Aurora Global Database
DynamoDB Global Tables
```

Protects against:
- regional failure

## Azure

### HA

```text
Azure SQL Zone Redundant
Azure Database HA
```

### DR

```text
Azure SQL Failover Groups
Geo Replication
Cosmos DB Multi-Region
```

Easy interview rule:

> Multi-AZ is primarily HA. Multi-region is primarily DR.

---

# 12. Storage HA and DR

## AWS

### HA

- S3
- EFS Multi-AZ

### DR

- S3 Cross-Region Replication
- AWS Backup
- EBS snapshot copy
- Cross-region snapshot copy

## Azure

### HA

- ZRS

### DR

- GRS
- GZRS
- RA-GRS

---

# 13. Network HA

## AWS

Typical components:

- Multiple AZ subnets
- One NAT Gateway per AZ
- ALB / NLB across AZs
- Multiple Direct Connect links
- Redundant VPN tunnels
- Transit Gateway
- Route 53
- Global Accelerator

## Azure

Typical components:

- Zone-redundant Load Balancer
- Application Gateway
- Azure Front Door
- VPN Gateway active-active
- ExpressRoute redundancy
- Traffic Manager
- Virtual WAN

---

# 14. HA Is Not Backup

These concepts solve different problems.

```text
HA
↓
Keeps service running

Backup
↓
Restores lost or corrupted data

DR
↓
Restores business service after major disaster
```

A mature design may need all three.

---

# 15. Interview-Ready Answer

## Explain HA, FT and DR

> “High Availability is about minimizing downtime by running redundant components, normally across multiple Availability Zones. Fault Tolerance goes further and aims to continue service without interruption when components fail, usually through active-active redundancy and real-time or near-real-time replication. Disaster Recovery handles major failures such as a complete region outage and uses a secondary region, backups, data replication, and agreed RTO and RPO values.”

## AWS example

> “In AWS, I could use an ALB with EC2 or EKS across three AZs and RDS Multi-AZ for HA. For DR, I could replicate to another region using Aurora Global Database or cross-region replicas and use Route 53 or Global Accelerator for failover.”

## Azure example

> “In Azure, I would use Availability Zones, Application Gateway or Azure Front Door, AKS across zones, Azure SQL zone redundancy for HA, and geo-replication or failover groups for regional DR.”

---

# 16. Key Interview Questions

## Q1. What is the difference between HA and FT?

HA minimizes downtime using redundancy and automated failover. FT aims for no visible interruption by maintaining active redundant components and real-time replication.

## Q2. What is the difference between HA and DR?

HA usually protects against component or AZ failures within a region. DR protects against large failures such as losing an entire region or site.

## Q3. What is RTO?

The maximum acceptable time to restore a service after a failure.

## Q4. What is RPO?

The maximum acceptable amount of data loss measured in time.

## Q5. What is the difference between Multi-AZ and Multi-Region?

Multi-AZ is mainly used for HA within a region. Multi-region is mainly used for DR or global fault-tolerant designs.

## Q6. Is RDS Multi-AZ a DR solution?

Primarily no. It is mainly an HA solution within a region. Regional DR generally requires cross-region replication, snapshots, or Aurora Global Database.

## Q7. Which AWS DR strategy is cheapest?

Backup and Restore.

## Q8. Which AWS DR strategy gives the lowest RTO/RPO?

Multi-site active-active.

## Q9. Why is active-active expensive?

Because full production capacity, networking, application stacks, observability, and replicated data must be maintained in multiple sites or regions.

## Q10. Does HA replace backups?

No. HA protects service availability. Backups protect against deletion, corruption, ransomware, or data-loss scenarios.
