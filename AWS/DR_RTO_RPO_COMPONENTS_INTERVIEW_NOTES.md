# AWS Disaster Recovery — Component-Level RTO & RPO Interview Notes

> Purpose: interview-ready reference for moving an application from a failed primary AWS Region to a DR Region.
>
> These values are **practical design targets / typical service characteristics**, not universal guarantees. Actual RTO and RPO must be validated through DR testing for the specific application.

---

# 1. Core Principle

For an application to recover in another Region, do not think only about EKS or EC2.

You must recover every critical dependency:

~~~text
DNS / Traffic
      |
Load Balancer
      |
Compute / EKS
      |
Application
      |
Persistent Storage
      |
Database
      |
Object Storage
      |
Secrets / Keys
      |
Queues / Cache / External Dependencies
~~~

The application can only meet the RTO/RPO of its critical path.

---

# 2. RTO and RPO Reminder

## RPO — Recovery Point Objective

How much data can the business afford to lose?

~~~text
RPO = 15 minutes
~~~

means approximately up to 15 minutes of the most recent data could be lost after a disaster.

## RTO — Recovery Time Objective

How long can the service remain unavailable?

~~~text
RTO = 15 minutes
~~~

means the application must be restored within 15 minutes.

---

# 3. Example Regional DR Architecture

~~~text
                          Route 53 / Global Routing
                          /                      \
                         /                        \
                PRIMARY REGION                 DR REGION
                --------------                 ---------

                     ALB                           ALB
                      |                             |
                     EKS                           EKS
                      |                             |
                  Application                  Application
                      |                             |
                     PVC                           PVC
                      |                             |
               EBS / EFS  -----------------> DR Storage
                      |
                      +---- Aurora ----------> Aurora Secondary
                      |
                      +---- S3 CRR ----------> DR S3
                      |
                      +---- Secrets ---------> DR Secret
                      |
                      +---- ECR -------------> DR ECR
~~~

For a low RTO design, the DR Region should already have the baseline platform available.

---

# 4. Component-Level RTO / RPO

| Component | DR Mechanism | Practical RPO Target | Practical RTO Target | Important Note |
|---|---|---:|---:|---|
| Route 53 / DNS | Failover/health-based routing | N/A | ~1–5 min | DNS caching and health-check timing affect recovery |
| ALB / NLB | Pre-created in DR | N/A | Near-zero to a few minutes | Do not wait to create it during disaster for low RTO |
| VPC / Subnets / SG / IAM | Pre-created with Terraform/IaC | ~0 for config | Near-zero if already deployed | IaC repo is the source of truth |
| EKS cluster | Warm standby cluster | ~0 for cluster config | ~5–10 min to scale app capacity | Creating EKS from scratch increases RTO significantly |
| Kubernetes manifests | GitOps / Git | ~0 for declared config | ~1–5 min | ArgoCD/Flux can redeploy quickly |
| ECR images | Cross-Region replication | Seconds/minutes in normal conditions | Near-zero if image already present | Validate critical images exist in DR |
| Aurora Global Database | Cross-Region replication | Typically seconds | Typically minutes | AWS documents RPO in seconds and RTO in minutes |
| S3 | CRR + optional S3 RTC | Up to ~15 min target with RTC(replication time control) | Minutes | RTC provides a predictable 15-minute replication objective |
| EFS | EFS cross-Region replication | ~15 min for most file systems | ~5–15 min if DR mount targets are ready | Destination networking must be pre-created |
| EBS snapshot DR | Snapshots + cross-Region copy | Snapshot interval + copy lag; often ~1 hour in simple designs | ~30–60+ min | Restore volume, PV/PVC, attach, mount, validate |
| EC2 + EBS using AWS DRS | Continuous block replication | Sub-second / seconds typical | ~5–20 min typical | Strong fit for server-level DR |
| EKS + EBS-backed PVC | Backup/app-aware replication | Depends on chosen data protection method | Often tens of minutes with snapshot restore | DRS is not the natural PVC DR mechanism |
| Secrets Manager | Cross-Region secret replication / automation | Usually minutes; validate actual lag | ~1–5 min if replica exists | Secret and KMS dependencies must both be ready |
| KMS | Pre-created key strategy / multi-Region keys where appropriate | N/A | Near-zero if ready | DR resources must be able to decrypt data |
| Cache / Redis | Global datastore / rebuild / app strategy | Seconds to data-loss-tolerant depending design | Minutes | Decide whether cache data must survive |
| Queue / Messaging | Service-specific multi-Region/app strategy | Architecture-specific | Architecture-specific | Region-specific queues need explicit DR design |
| Monitoring / Alerts | Deploy in both Regions | N/A | Immediate if already active | Must detect primary failure before failover starts |

---

# 5. EBS-backed PVC — Backup/Restore DR

For an EKS application using an EBS-backed PVC:

~~~text
PRIMARY
Pod
 |
PVC
 |
PV
 |
EBS
 |
Snapshot
 |
Cross-Region Copy
 |
========================
DR
Latest Snapshot
 |
Create EBS Volume
 |
Create / bind PV
 |
PVC
 |
DR Pod
~~~

A simple practical target might be:

~~~text
EBS-backed PVC

RPO -> ~1 hour
RTO -> ~30–60+ minutes
~~~

if snapshots are taken hourly.

This is not an AWS fixed value.

The real RPO is:

~~~text
Snapshot Schedule
      +
Outstanding Copy Lag
~~~

The real RTO includes:

~~~text
Find recovery point
      |
Restore EBS volume
      |
Create/reconcile PV and PVC
      |
Attach volume
      |
Mount filesystem
      |
Start application
      |
Validate data
~~~

---

# 6. EC2 + EBS with AWS Elastic Disaster Recovery

For a normal EC2/server workload, DRS gives a much more aggressive recovery target than periodic snapshots.

~~~text
PRIMARY
EC2
 |
EBS
 |
AWS DRS Agent
 |
Continuous changed-block replication
 |
==============================
DR REGION
 |
Staging EBS Volumes
 |
Launch Recovery
 |
Recovered EC2 + EBS
~~~

AWS documents typical characteristics as:

~~~text
RPO -> typically sub-second / seconds
RTO -> typically 5–20 minutes
~~~

This is a strong answer when an interviewer asks:

> “What if one-hour snapshot RPO is not acceptable?”

Answer:

> “For an EC2 workload I would move from periodic EBS snapshots to AWS Elastic Disaster Recovery, which continuously replicates block changes to the DR Region.”

---

# 7. Important — DRS Is Not the Normal EKS PVC DR Mechanism

For EKS:

~~~text
Pod
 |
PVC
 |
PV
 |
EBS CSI
 |
EBS Volume
~~~

Kubernetes can move the EBS attachment between worker nodes.

AWS DRS is source-server oriented and does not reproduce:

~~~text
PVC
PV
StorageClass
StatefulSet
CSI relationships
~~~

Therefore:

~~~text
EC2 + EBS
 -> DRS

EKS + EBS PVC
 -> Protect the persistent DATA separately
 -> Recreate PV/PVC in DR
 -> Deploy app using GitOps/IaC
~~~

If the business requires seconds-level RPO for an EKS workload, strongly consider moving critical state into a service with native/application-aware cross-Region replication.

---

# 8. EFS-backed PVC

For an EFS-backed PVC:

~~~text
PRIMARY EKS
    |
   PVC
    |
   EFS-A
    |
EFS Replication
    |
=================
DR EFS-B
    |
   PVC
    |
DR EKS
~~~

AWS documents approximately:

~~~text
RPO -> ~15 minutes for most EFS file systems
~~~

For a warm DR environment with mount targets, security groups and EKS already prepared, a practical design target can be:

~~~text
RTO -> ~5–15 minutes
~~~

The RTO becomes much longer if the DR EFS mount targets, network, cluster or node capacity must be created during the incident.

---

# 9. Database — Aurora Global Database

For a stateful application, the database often matters more than Kubernetes itself.

~~~text
Primary Aurora
      |
Cross-Region replication
      |
Aurora Secondary
~~~

AWS documents:

~~~text
RPO -> typically seconds
RTO -> typically minutes
~~~

For a planned switchover, synchronization can provide zero data loss.

For an unplanned regional failure, some asynchronous replication lag can remain.

---

# 10. S3

~~~text
Primary S3
    |
CRR
    |
DR S3
~~~

If the business needs a predictable target, S3 Replication Time Control can be used.

Interview target:

~~~text
RPO -> up to ~15 minutes using S3 RTC target
RTO -> usually minutes because the DR bucket already exists
~~~

Remember that replication lag is different from application cutover time.

---

# 11. Warm Standby — Recommended for Low RTO

If leadership requires an overall RTO around 10–15 minutes, do not create the whole DR platform after the disaster.

Pre-create:

~~~text
DR VPC
DR Subnets
DR Security Groups
DR EKS
DR minimum worker capacity
DR ALB/NLB
DR Route 53 target
DR EFS mount targets if used
DR database replica
DR S3 bucket
DR ECR images
DR secrets/KMS
Observability
~~~

Then failover becomes:

~~~text
Detect failure
      |
Promote storage / database
      |
Scale application
      |
Verify pod readiness
      |
Validate application
      |
Shift traffic
~~~

---

# 12. Example 15-Minute RTO Timeline

One possible warm-standby target:

~~~text
0–2 min
Detect primary Region failure

2–5 min
Promote DB / prepare DR storage

5–8 min
Scale DR EKS application

8–10 min
Pods Ready + PVC mounted

10–12 min
Application validation

12–15 min
Traffic/DNS cutover
~~~

This is an engineering target that must be proven through DR exercises.

---

# 13. Overall Application RPO

The overall RPO is controlled by the slowest **critical stateful dependency**.

Example:

~~~text
Aurora        -> RPO seconds
S3            -> RPO ~15 min
EFS           -> RPO ~15 min
EBS snapshots -> RPO ~1 hour
~~~

If the application requires the EBS data:

~~~text
Overall Application RPO
        |
        v
~1 hour
~~~

You cannot claim a seconds-level application RPO simply because the database replicates in seconds.

---

# 14. Overall Application RTO

RTO follows the critical recovery path.

Example:

~~~text
DB promotion             3 min
Storage readiness        5 min
Scale/start application  5 min
Validation               2 min
DNS/traffic shift        2 min
~~~

Some activities can run in parallel, so overall RTO is not always the arithmetic sum.

The key point is:

> Identify the slowest sequential dependencies and validate the complete application recovery time, not only individual AWS service recovery times.

---

# 15. Two Interview Scenarios to Remember

## Scenario A — EKS + EBS PVC using snapshots

~~~text
EKS/App       RPO ~0 config       RTO ~5–10 min
Aurora        RPO seconds         RTO minutes
S3            RPO ~15 min         RTO minutes
EBS PVC       RPO ~1 hour         RTO ~30–60+ min
Route 53      RPO N/A             RTO ~1–5 min
~~~

Likely overall target:

~~~text
RPO -> ~1 hour
RTO -> ~30–60+ minutes
~~~

because EBS recovery is the limiting critical component.

## Scenario B — EKS + EFS PVC with warm standby

~~~text
EKS/App       RPO ~0 config       RTO ~5–10 min
Aurora        RPO seconds         RTO minutes
S3            RPO ~15 min         RTO minutes
EFS PVC       RPO ~15 min         RTO ~5–15 min
Route 53      RPO N/A             RTO ~1–5 min
~~~

A practical architecture target can be:

~~~text
RPO -> ~15 minutes
RTO -> ~15 minutes
~~~

subject to application validation and tested automation.

---

# 16. Interview-Ready Answer

> “For regional DR I calculate RTO and RPO for every critical dependency, not only for EKS. Kubernetes manifests can be recreated almost immediately from GitOps, but stateful services determine the real recovery objective. Aurora Global Database typically gives seconds-level RPO and minutes-level RTO. S3 with RTC can target replication within 15 minutes. EFS replication gives about a 15-minute RPO for most file systems. EBS snapshot-based recovery has an RPO based on snapshot frequency and usually a much longer RTO because the volume must be restored and rebound to the workload. For an EC2 server with EBS and aggressive objectives, AWS DRS provides continuous block replication with sub-second/seconds RPO and typically 5–20 minute RTO. For an EKS EBS-backed PVC, I would not rely on DRS as the normal PVC mechanism; I would protect the application data separately and recreate the Kubernetes storage objects in the DR cluster. The overall application RPO is set by the slowest critical stateful component, and the overall RTO is set by the critical recovery path.”
