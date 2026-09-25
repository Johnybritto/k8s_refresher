# Cloud Cost Optimization — Cut AWS / Azure Infrastructure Spend by 40%

## Interview Scenario

**Question:**  
Leadership wants to cut cloud infrastructure spending by 40%. How would you achieve this, specifically for AWS and Azure?

---

# 1. How to Frame the Problem

A 40% cost reduction is aggressive.

Do not start with:

- "Buy Reserved Instances"
- "Move everything to Spot"
- "Reduce all server sizes"
- "Remove redundancy"

A stronger response is:

> "A 40% reduction is aggressive, so I would treat it as a FinOps and architecture program rather than randomly cutting infrastructure. First I would baseline where the money is going, then eliminate waste, right-size usage, introduce autoscaling, and only after workloads stabilize would I purchase commitments such as Savings Plans or Reservations. Otherwise I may commit to infrastructure that I should have removed."

The ordering matters:

~~~text
Baseline spend
    |
    v
Eliminate waste
    |
    v
Right-size
    |
    v
Autoscale
    |
    v
Determine steady-state baseline
    |
    v
Commit predictable usage
    |
    v
Optimize storage / network / DB / licensing / logging
    |
    v
Architecture optimization
    |
    v
Continuous FinOps governance
~~~

A simple memory line:

> **First eliminate waste, then reduce usage, and only then reduce the rate you pay for that usage.**

---

# 2. Establish the Current Cost Baseline

Before making changes, identify exactly where money is being spent.

## AWS

Start with:

~~~text
AWS Cost Explorer
AWS Cost Optimization Hub
AWS Compute Optimizer
AWS Cost and Usage Report (CUR)
AWS Budgets
Cost allocation tags
~~~

## Azure

Start with:

~~~text
Azure Cost Management
Azure Advisor
Azure Cost Optimization Workbook
Budgets
Tags
Management Groups / Subscriptions
~~~

Break spend into major categories:

~~~text
Compute
Database
Storage
Network
Kubernetes
Licensing
Managed services
Observability
Non-production
~~~

Then identify the top services/resources responsible for most of the bill.

Do not spend days optimizing a service that contributes only a tiny percentage of total spend.

---

# 3. First Attack — Idle Resources

Idle resources are usually the safest savings.

## Typical AWS Waste

~~~text
Idle EC2 instances
Unused EBS volumes
Old EBS snapshots
Unused Elastic IPs
Idle Load Balancers
Old AMIs/snapshots
Idle RDS instances
Unused NAT Gateways
Overprovisioned EKS nodes
Dev/Test environments running 24x7
~~~

## Typical Azure Waste

~~~text
Idle VMs
Unattached Managed Disks
Old snapshots
Unused Public IPs
Idle Load Balancers
Unused App Service Plans
Idle SQL resources
Overprovisioned AKS node pools
Dev/Test resources running continuously
~~~

---

# 4. Shut Down Non-Production When Not Needed

A common example:

~~~text
Dev / QA

Current:
24 x 7

Actual business usage:
08:00 – 20:00 weekdays
~~~

Instead:

~~~text
Start before business hours
Stop after business hours
Keep off during nights/weekends unless required
~~~

This can significantly reduce non-production compute cost.

The principle:

> If a system is not being used, it should not consume full-time infrastructure cost.

---

# 5. Rightsize Before Buying Discounts

Example:

~~~text
m6i.4xlarge

Average CPU = 12%
Memory       = 25%
~~~

Do not immediately purchase a long-term commitment for that instance.

First determine whether the workload can run on:

~~~text
m6i.2xlarge
or
m6i.xlarge
~~~

For Azure the same applies to oversized VM SKUs.

Review:

~~~text
CPU
Memory
Network
Disk IOPS
Peak utilization
P95 / P99 usage
Business seasonality
~~~

Do not make decisions based only on average CPU.

The correct sequence:

~~~text
Rightsize
   |
   v
Observe
   |
   v
Stabilize
   |
   v
Commit predictable baseline
~~~

---

# 6. Autoscaling

A common cost problem:

~~~text
20 servers running 24x7
~~~

Even though demand varies.

A better model:

~~~text
Minimum = 6
Normal  = 10
Peak    = 20
~~~

Capacity should follow demand.

## AWS Autoscaling Options

~~~text
EC2 Auto Scaling Groups
ECS Service Auto Scaling
EKS Cluster Autoscaler
EKS Karpenter
Lambda
Aurora Serverless where appropriate
~~~

## Azure Autoscaling Options

~~~text
VM Scale Sets
Azure Autoscale
AKS Cluster Autoscaler
KEDA
Azure Functions
Azure Container Apps
Azure SQL Serverless where appropriate
~~~

The principle:

> **Do not pay continuously for peak capacity if the workload is not continuously at peak.**

---

# 7. Commitments — Only After Optimization

Once the infrastructure is rightsized and the steady-state baseline is understood, purchase commitments.

## AWS

Options include:

~~~text
Savings Plans
Reserved Instances where appropriate
~~~

A practical model:

~~~text
Total compute usage
       |
       +---- Stable baseline ------> Savings Plan / RI
       |
       +---- Variable demand ------> On-Demand
       |
       +---- Interruptible --------> Spot
~~~

Example allocation:

~~~text
70% predictable baseline -> commitment
20% variable              -> On-Demand
10% interruptible         -> Spot
~~~

The exact percentages depend on the workload.

Do not necessarily cover 100% of usage with commitments.

---

# 8. Azure Commitments

Azure equivalents include:

~~~text
Azure Reservations
Azure Savings Plan for Compute
~~~

Again:

~~~text
Rightsize FIRST
      |
      v
Determine stable baseline
      |
      v
Purchase commitment
~~~

Stable predictable workloads may be good candidates for reservations.

More dynamic compute may benefit from Savings Plans.

---

# 9. Use Spot for Suitable Workloads

Spot can generate major savings, but not every workload is suitable.

## Good Candidates

~~~text
Batch processing
CI/CD runners
Stateless workers
Image/video processing
Data processing
Non-critical EKS/AKS worker pools
Queue consumers
Fault-tolerant jobs
~~~

## AWS

~~~text
EC2 Spot
EKS Spot node groups
Mixed Instances ASG
~~~

## Azure

~~~text
Azure Spot VMs
AKS Spot node pools
VMSS Spot
~~~

## Poor Candidates

~~~text
Single-instance databases
Critical stateful workloads
Workloads unable to tolerate interruption
~~~

Use Spot only where interruption is acceptable and the application can recover.

---

# 10. Kubernetes Is Often Overprovisioned

EKS and AKS often contain significant hidden waste.

Example:

~~~text
Node capacity = 100 CPUs

Actual application usage = 30 CPUs

Pod requests = 75 CPUs
~~~

The scheduler sees:

~~~text
75 CPUs requested
~~~

even though actual usage is only 30 CPUs.

This can prevent cluster scale-down.

Review:

~~~text
Requests vs actual usage
Limits
Idle namespaces
Replica counts
DaemonSets
Oversized nodes
Node-pool fragmentation
PDBs blocking scale-down
Affinity rules
Topology constraints
GPU utilization
~~~

---

# 11. EKS Cost Optimization

For EKS:

~~~text
Karpenter / Cluster Autoscaler
Spot node groups
Graviton where compatible
Right-size pod requests
Right-size limits
Separate workload node pools
Scale non-production pools down
Review DaemonSet overhead
Review PDBs preventing consolidation
~~~

A cluster running at low utilization may have major savings potential.

---

# 12. AKS Cost Optimization

For AKS:

~~~text
Cluster Autoscaler
KEDA
Spot node pools
Right-size requests
Right-size limits
Separate system/user pools
Scale user pools down when unused
Review VM SKU sizing
Review unused node pools
~~~

Again, the biggest issue is often not the Kubernetes control plane cost.

The bigger issue is overprovisioned worker compute and over-requested applications.

---

# 13. Database Optimization

Database cost can represent a large portion of the bill.

## AWS

Review:

~~~text
RDS instance sizing
Aurora sizing
Reserved DB instances
Aurora Serverless where suitable
Multi-AZ only where required
Read replicas actually needed
Storage configuration
Provisioned IOPS
Old snapshots
Dev databases running 24x7
~~~

## Azure

Review:

~~~text
Azure SQL tier
Managed Instance sizing
SQL Elastic Pools
SQL Serverless
Reserved capacity
Cosmos DB RU provisioning
Cosmos DB autoscale
PostgreSQL sizing
MySQL sizing
~~~

Do not remove production HA purely to hit a cost target.

Example of a bad optimization:

~~~text
Tier-1 production DB
Multi-AZ
    |
    v
Change to Single-AZ only to save money
~~~

That is risk transfer, not good optimization.

---

# 14. Storage Optimization

Storage often accumulates slowly and becomes expensive.

## AWS Storage

Review:

~~~text
S3
EBS
Snapshots
Backups
Log storage
Provisioned IOPS
Old AMIs
~~~

For S3:

~~~text
S3 Standard
    |
    v
Standard-IA
    |
    v
Glacier tiers
~~~

Use lifecycle policies where appropriate.

Also review:

~~~text
Unused EBS volumes
Old snapshots
Oversized provisioned IOPS
Logs retained indefinitely
~~~

---

# 15. Azure Storage

Review:

~~~text
Blob Storage
Managed Disks
Snapshots
Backup retention
Log Analytics retention
Storage Accounts
~~~

Blob tiers:

~~~text
Hot
 |
 v
Cool
 |
 v
Cold
 |
 v
Archive
~~~

Example:

~~~text
Application logs

Current:
5 years online retention

Better:
90 days online
+
archive older data
~~~

Retention should match operational and compliance requirements.

---

# 16. Network Cost — Often Overlooked

Many engineers focus only on compute.

Network can be a major cost area.

## AWS

Review:

~~~text
Cross-AZ traffic
Cross-region traffic
Internet egress
NAT Gateway processing
Duplicate data movement
CloudFront opportunities
Architecture repeatedly crossing AZ boundaries
~~~

Example:

~~~text
App AZ-A
   |
   v
Service AZ-B
   |
   v
Database AZ-A
~~~

This can continuously create inter-AZ traffic.

But do not remove AZ-level resiliency purely to reduce network cost.

Optimize placement and traffic patterns while retaining availability.

---

# 17. Azure Network Optimization

Review:

~~~text
Cross-region transfer
Inter-zone transfer
Internet egress
NAT Gateway
ExpressRoute traffic
Repeated cross-zone service calls
Inefficient data movement
~~~

Again:

> Optimize traffic paths without compromising HA.

---

# 18. Licensing Cost

Licensing can be a major cost component, especially for Windows and SQL workloads.

## Azure

Investigate:

~~~text
Azure Hybrid Benefit
Windows Server licensing
SQL Server licensing
License mobility where applicable
~~~

Azure Hybrid Benefit may reduce cost where existing eligible licenses can be used.

## AWS

Review:

~~~text
Windows licensing
SQL Server licensing
BYOL eligibility where applicable
Dedicated Hosts where justified
License-included vs alternative architectures
~~~

Licensing savings can sometimes reduce spend without changing application performance.

---

# 19. Observability Cost

Observability often becomes a hidden bill.

Review:

## AWS

~~~text
CloudWatch Logs
CloudWatch custom metrics
CloudWatch retention
High-cardinality metrics
Log ingestion
Tracing volume
~~~

## Azure

~~~text
Azure Monitor
Log Analytics
Application Insights
Metric ingestion
Retention
~~~

## Third-Party Tools

~~~text
Dynatrace
Splunk
Other APM/log platforms
~~~

Common problem:

~~~text
Everything
  |
  v
DEBUG logs
  |
  v
Central logging
  |
  v
Retain for years
~~~

Better:

~~~text
Production operational logs -> appropriate retention
Security/audit logs          -> compliance retention
Debug logs                   -> short retention
Cold historical logs         -> archive
~~~

Do not retain or ingest high-volume telemetry that nobody uses.

---

# 20. Architecture Optimization

After obvious waste is removed, consider structural changes.

Examples:

## VM to Container

~~~text
One VM per application
       |
       v
Shared container platform
~~~

Potential benefits:

- better bin packing
- more efficient compute utilization
- faster scaling

But validate total cost.

---

## Always-On Compute to Serverless

~~~text
Low-volume API
      |
      v
Lambda / Azure Functions
~~~

For low or bursty workloads, this may reduce idle cost.

But serverless is not automatically cheaper for every workload.

---

## Database Consolidation

~~~text
Many underutilized dedicated databases
             |
             v
Shared/elastic database architecture where suitable
~~~

Example:

~~~text
Azure SQL Elastic Pool
~~~

where workload characteristics justify it.

---

# 21. Do Not Assume Kubernetes Is Always Cheaper

Do not migrate workloads to Kubernetes only because Kubernetes sounds efficient.

Kubernetes adds:

- worker nodes
- platform operations
- observability
- network components
- engineering overhead
- security tooling
- upgrade complexity

Evaluate total cost of ownership.

---

# 22. Do Not Assume Serverless Is Always Cheaper

Serverless can be excellent for:

~~~text
Bursty traffic
Low-volume APIs
Event-driven workloads
Short-running functions
~~~

But long-running high-volume workloads may be cheaper on:

~~~text
EC2
ECS
EKS
Azure VMs
AKS
Container Apps
~~~

depending on the workload.

Always compare actual TCO.

---

# 23. Build a Savings Waterfall

Do not simply add theoretical savings percentages.

Example current cloud spend:

~~~text
$1,000,000 per month
~~~

Possible savings waterfall:

~~~text
Current spend
$1.00M
   |
   | Idle resource cleanup / scheduling
   v
~$900K
   |
   | Rightsizing / autoscaling
   v
~$765K
   |
   | Savings Plans / Reservations
   v
~$650K
   |
   | Storage / network / licensing /
   | logging / architecture optimization
   v
~$600K
~~~

That reaches the 40% target.

The important point:

> Savings overlap.

Do not say:

~~~text
Rightsizing saves 20%
Savings Plans save 30%
Spot saves 50%

Therefore total savings = 100%
~~~

That is wrong because each optimization changes the cost base for the next one.

Use sequential savings.

---

# 24. Protect HA While Cutting Cost

This is critical.

Classify workloads.

## Tier 1

~~~text
Customer-facing
Revenue-generating
Critical systems
~~~

Protect:

~~~text
HA
FT
DR
Backups
Monitoring
Capacity headroom
~~~

---

## Tier 2

~~~text
Important business applications
~~~

Optimize aggressively but maintain required availability.

---

## Tier 3

~~~text
Dev
QA
Test
Sandbox
Training
~~~

Use aggressive controls:

~~~text
Auto-shutdown
Scheduled startup
Spot
Lower-cost SKUs
Scale-to-zero where possible
Short retention
~~~

---

# 25. What NOT to Do

Do not reach a 40% target by blindly doing this:

~~~text
3 production instances -> 1
Multi-AZ DB -> Single-AZ
Remove DR
Disable backups
Remove monitoring
Eliminate all spare capacity
~~~

That is not cost optimization.

That is transferring cloud cost into:

~~~text
Operational risk
Outage risk
Recovery risk
Business risk
~~~

---

# 26. AWS vs Azure Cheat Sheet

| Cost Lever | AWS | Azure |
|---|---|---|
| Cost analysis | Cost Explorer / Cost Optimization Hub | Cost Management / Advisor |
| Rightsizing | Compute Optimizer | Azure Advisor |
| Stable compute | Savings Plans / RI | Reservations / Savings Plan |
| Interruptible compute | EC2 Spot | Azure Spot VM |
| VM autoscaling | Auto Scaling Groups | VM Scale Sets |
| Kubernetes | EKS Autoscaler / Karpenter | AKS Cluster Autoscaler / KEDA |
| Storage tiering | S3 Lifecycle / Glacier | Blob Lifecycle / Cold / Archive |
| Serverless | Lambda | Azure Functions |
| Container services | ECS / EKS | Container Apps / AKS |
| License saving | BYOL/license optimization | Azure Hybrid Benefit |
| Budgets | AWS Budgets | Azure Budgets |
| Cost allocation | Tags / Cost Categories | Tags / Management Groups |
| Database optimization | RDS / Aurora / reserved DB capacity | Azure SQL / MI / elastic pools / reserved capacity |

---

# 27. Suggested Execution Plan

## Phase 1 — Visibility

~~~text
Tag resources
Build cost baseline
Identify top cost centers
Identify owners
Find unallocated spend
~~~

---

## Phase 2 — Waste Elimination

~~~text
Terminate idle resources
Remove unattached storage
Delete obsolete snapshots
Schedule non-prod
Remove unused load balancers/IPs
Review orphaned resources
~~~

---

## Phase 3 — Rightsizing

~~~text
Compute
Database
Kubernetes requests
Node pools
Storage IOPS
Managed services
~~~

---

## Phase 4 — Elasticity

~~~text
ASG / VMSS
Kubernetes autoscaling
KEDA
Serverless where justified
Scale non-prod down
~~~

---

## Phase 5 — Rate Optimization

~~~text
AWS Savings Plans
AWS Reserved Instances
Azure Reservations
Azure Savings Plans
Spot
Azure Hybrid Benefit
Licensing optimization
~~~

---

## Phase 6 — Architecture Optimization

~~~text
Container consolidation
Serverless
Database consolidation
Network path optimization
Storage lifecycle
Observability optimization
~~~

---

## Phase 7 — Governance

~~~text
Budgets
Alerts
Monthly FinOps review
Cost anomaly detection
Chargeback/showback
Tag enforcement
Cost ownership
Architecture reviews
~~~

---

# 28. Governance to Keep Savings From Coming Back

A one-time cleanup is not enough.

Implement:

~~~text
Budget alerts
Cost anomaly detection
Mandatory tagging
Resource ownership
Expiration dates for temporary environments
Monthly rightsizing reviews
Commitment utilization reviews
Reservation coverage reviews
Kubernetes utilization reviews
Storage lifecycle policies
Architecture cost reviews
~~~

Cloud cost optimization should become part of normal platform operations.

---

# 29. Questions to Ask Leadership

Before promising 40%, clarify:

~~~text
Is 40% against current monthly run rate?
What is the deadline?
Does it include licensing?
Does it include SaaS/APM?
Does it include data transfer?
Can non-prod availability be reduced?
Can we change architecture?
Can we make multi-year commitments?
What services are protected from reduction?
What are the RTO/RPO requirements?
What workloads can tolerate Spot interruption?
~~~

This helps avoid unrealistic assumptions.

---

# 30. Interview-Ready Answer

> "If leadership asks me to reduce AWS or Azure infrastructure spend by 40%, I would not immediately start deleting resources or purchasing reservations. First I would establish a baseline using AWS Cost Explorer and Cost Optimization Hub or Azure Cost Management and Advisor, and determine which services account for most of the spend.
>
> My first phase would be eliminating waste: idle VMs, unused disks, snapshots, load balancers and non-production resources running 24x7. Then I would right-size compute, databases and Kubernetes requests based on actual utilization and introduce autoscaling so we're not continuously paying for peak capacity.
>
> Once the environment is rightsized, I would identify the predictable baseline and cover that with AWS Savings Plans or Reserved Instances, or Azure Reservations and Savings Plans. I would keep variable demand on on-demand capacity and use Spot only for interruption-tolerant workloads.
>
> I would then optimize storage lifecycle, database tiers, network and cross-AZ or cross-region traffic, logging retention and licensing. For EKS or AKS I would pay particular attention to overprovisioned worker nodes and inflated pod requests because these often prevent cluster scale-down.
>
> I would track savings as a waterfall rather than adding theoretical percentages because the optimizations overlap. And I would classify workloads by criticality so the 40% target does not come at the expense of production HA, fault tolerance, backups or DR.
>
> So my approach is: eliminate waste, right-size, autoscale, commit stable usage, use Spot where safe, optimize storage/network/database/licensing, and continuously govern with FinOps."

---

# 31. One-Line Interview Summary

> **First eliminate waste, then reduce usage, then reduce the rate you pay for that usage — while protecting HA, FT and DR for critical workloads.**
