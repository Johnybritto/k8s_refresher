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

---

# 17. Velero and Portworx — Restoring EKS PV/PVC in DR

**Follow-up to Section 7: “DRS Is Not the Normal EKS PVC DR Mechanism.”**

**Short answer:** Yes — Velero and Portworx Backup are Kubernetes-aware tools that can help restore **both** Kubernetes objects (PVC, workloads, and where appropriate PVs) **and their backed-up volume data** into a DR cluster. The storage restoration method and destination CSI/StorageClass configuration decide how the *new* PV and underlying disk are created and bound.

**Do not confuse:**

- **AWS DRS** protects/relaunches EC2 servers and their attached disks. It does **not** independently recreate EKS PVC/PV/StatefulSet/CSI bindings.
- **Velero** is an **open-source** Kubernetes backup/restore tool: backs up Kubernetes resource definitions and optionally their persistent data through CSI snapshots, a CSI snapshot **data mover**, or file-system backup.
- **Portworx Backup (PX-Backup)** is a commercial Kubernetes application backup/restore tool; it protects Kubernetes resources and supported PVC data, including cross-cluster restore with a compatible target storage provisioner.
- **Portworx Enterprise / PX-DR** is **different from PX-Backup**: it can supply storage and Kubernetes application replication/failover through supported synchronous or asynchronous DR architectures.

## 17.1 The mental model — who restores the PVC?

| DR method | How the DR EBS volume gets there | How the DR PV/PVC appears |
|---|---|---|
| DRS applied to EC2 + EBS | DRS launches a **recovered EC2 instance with recovered disks**; reusing a disk in EKS needs additional careful detach/clone workflow | **Not automatic**. Create a static PV with the *DR* EBS volume ID, bind a PVC, and deploy the workload |
| EBS snapshot copied to DR | Snapshot is copied and a new DR EBS volume is restored (or restored via a supported CSI snapshot workflow) | Static PV/PVC **or** a configured CSI-aware restore tool provisions and binds volume |
| **Velero** backup/restore | From CSI snapshots available in DR, or transferred backup data (CSI Snapshot Data Movement / file-system backup) | Velero restores the PVC and other selected Kubernetes objects; CSI/restoration workflow creates/binds the usable target volume/PV |
| **Portworx Backup** | Restores backed-up application data using a supported destination storage integration | Restores PVC and application resources; compatible destination StorageClass/CSI provisions/binds PV |
| **Portworx Enterprise PX-DR** | Replicates supported application data between paired Kubernetes clusters | Supported DR workflow coordinates Kubernetes resource migration and application activation |

**Key rule:** Backing up YAML **alone** only recreates PVC *definitions*. To recover actual application data, you must configure **volume-data backup/replication and restoration**. The original EBS volume ID from the primary region must not simply be reused in the secondary region.

## 17.2 Velero example — EKS primary to DR cluster

~~~mermaid
flowchart TB
    subgraph P["Primary Region / EKS"]
      APP["Application Pod"]
      PVC["PVC + EBS-backed PV"]
      V["Velero Backup"]
      APP --> PVC --> V
    end
    B[("Backup Location<br/>Kubernetes objects + volume backup data")]
    subgraph R["DR Region / EKS"]
      VR["Velero Restore"]
      K["Restore PVC and application objects"]
      CSI["Target EBS CSI + StorageClass<br/>Provision / restore volume and bind PV"]
      POD["Application Pod mounts recovered data"]
      VR --> K --> CSI --> POD
    end
    V -->|"Copy / data movement<br/>accessible across Regions"| B
    B --> VR
~~~

**Prepare before disaster:**

1. Deploy a DR EKS cluster (warm standby for lower RTO), EBS CSI driver, adequate node capacity, IAM/KMS permissions, and a working **DR StorageClass**.
2. Install/configure Velero on both clusters. Configure the backup location/repository so the DR cluster can access the backups.
3. Choose a data-protection method that **actually makes volume data available in the DR Region**: for example CSI snapshot data movement or file-system backup to a cross-region-accessible repository, or a suitable snapshot-copy workflow. An EBS snapshot that exists **only in the original Region** is not enough.
4. Ensure the target StorageClass and CSI driver are compatible; remap classes when needed. Make sure the restored EBS volume and scheduled pod are in the same AZ.
5. For databases, ensure an **application-consistent** recovery strategy (hooks, quiescing, or application-native backup) rather than assuming a crash-consistent disk snapshot is always sufficient.

**On failover (simplified):**

~~~bash
# In the DR EKS context; backup must be visible to DR Velero
velero backup get
velero restore create app-dr --from-backup app-backup
velero restore describe app-dr --details

# Confirm the storage is bound and the workload can mount it
kubectl get pvc,pv -n myapp
kubectl get pods -n myapp
~~~

The restore creates/recreates the selected Kubernetes objects and invokes the configured CSI/data-restoration workflow. **It does not mean the original source-region EBS volume moves to the DR Region.** A fresh target volume is provisioned/restored from the recoverable data.

Verify restored file/database content, application readiness and dependency connectivity **before routing users**.

## 17.3 Where Portworx fits

**Portworx Backup (backup/restore):**

~~~text
Primary Kubernetes app + PVC + persistent data
       |
  PX-Backup job → backup destination
       |
  PX-Backup restore on DR cluster
       |
  Restore app resources + PVC and backed-up data
       |
  Destination storage/CSI binds PV → start app
~~~

**Portworx Enterprise PX-DR (replication/failover):**

~~~text
Primary Kubernetes cluster + Portworx storage
       |
  Supported synchronous or asynchronous replication
       |
  Secondary Kubernetes cluster + Portworx storage
       |
  DR workflow activates protected application + data
~~~

PX-DR provides capabilities different from simply running periodic backups. **Synchronous** versus **asynchronous** DR has different infrastructure, latency and RPO implications; do not assert that any Portworx installation gives zero RPO. Product/version/storage-class restrictions must be checked.

## 17.4 RPO / RTO — interview distinction

- **Velero and PX-Backup with periodic backups:** RPO generally depends on the backup interval, completion time and availability of the backup data in DR. RTO includes recreating resources, restoring data, volume binding/mounting and app checks.
- **Portworx PX-DR replication:** RPO/RTO depend on **synchronous vs asynchronous** DR setup, network characteristics, supported platforms and tested failover procedures.
- **DRS:** EC2-level recovery targets do not imply that EKS PV/PVC and application wiring are recovered within the same RTO.
- **Regardless of tooling:** verify data consistency, access modes, AZ placement, KMS, storage class mapping, application startup and failback.

**Interview answer:**

> “DRS can recover the EC2 server and disk, but it doesn't rebuild the Kubernetes PVC/PV mapping. For EKS persistent volumes I'd normally use a Kubernetes-aware tool such as Velero or Portworx Backup to restore both workload definitions and backed-up volume data into a DR cluster. Velero can combine resource restoration with CSI snapshots, snapshot data movement or filesystem backup. Portworx also offers PX-DR replication capabilities, which are distinct from backup/restore. The destination CSI driver provisions or binds storage, and I confirm PVC Bound, correct data, and application readiness. RPO and RTO depend on the backup or replication method — not merely the tool's name.”

**Official references:**

- Velero CSI Snapshot Data Movement: https://velero.io/docs/v1.18/csi-snapshot-data-movement/
- Velero CSI integration: https://velero.io/docs/main/csi/
- Portworx Backup restore documentation: https://docs.portworx.com/portworx-backup-on-prem/use-px-backup/backup-restore/restore-backup/restore-namespace-backup
- Portworx Enterprise disaster recovery options: https://docs.portworx.com/portworx-enterprise/operations/disaster-recovery
