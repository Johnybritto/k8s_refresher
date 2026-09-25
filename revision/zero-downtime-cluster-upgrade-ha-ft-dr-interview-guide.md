# Zero-Downtime Cluster Upgrade Strategy — Linux/EC2, VMware, Kubernetes

## Interview Scenario

**Question:**  
Upgrade the core software running in the cluster, upgrade the servers/nodes inside the cluster, and upgrade all applications running in the cluster. During the complete activity, the applications must remain online. Use HA, Fault Tolerance (FT), and Disaster Recovery (DR) wherever appropriate.

---

# 1. First Clarify the Cluster Type

Do not immediately assume "cluster" means Kubernetes.

A strong opening is:

> "Before proposing the detailed upgrade sequence, I would clarify what kind of cluster we are dealing with — for example Linux HA, VMware, Kubernetes, database, middleware, etc. The exact mechanics differ by platform. But the common strategy is the same: avoid a big-bang upgrade, preserve redundancy, upgrade one failure domain at a time, validate at every stage, and keep rollback and DR ready."

For this guide, assume three possibilities:

1. Linux servers running on AWS EC2
2. VMware vSphere cluster
3. Kubernetes cluster

---

# 2. Core Principle

Never upgrade all layers simultaneously.

~~~text
Pre-checks / Compatibility / Backups
                |
                v
        Establish HA capacity
                |
                v
      Upgrade core platform
                |
                v
      Upgrade servers/nodes
                |
                v
       Upgrade applications
                |
                v
      End-to-end validation
                |
                v
    Remove old capacity/version
~~~

The three upgrade domains are:

~~~text
1. Core cluster/platform software
2. Underlying servers/nodes/OS
3. Applications
~~~

Each domain must have its own:

- pre-check
- health validation
- rollback point
- go/no-go decision
- monitoring
- change window
- failure handling

---

# 3. What "Zero Downtime" Really Requires

An upgrade procedure alone cannot create zero downtime.

The environment must already be designed for redundancy.

Bad design:

~~~text
1 App Instance
      |
1 Server
~~~

If that server is rebooted, the application is unavailable.

Better design:

~~~text
               Load Balancer
                    |
        +-----------+-----------+
        |           |           |
      Node-1      Node-2      Node-3
        |           |           |
      App-1       App-2       App-3
~~~

During maintenance one node can be removed while the others continue serving traffic.

For critical upgrades, maintain **N+1 or N+2 capacity**.

Example:

~~~text
Normal:
Node-1
Node-2
Node-3

Upgrade window:
Node-1
Node-2
Node-3
Node-4  <-- temporary spare capacity
~~~

This protects the service if one node is intentionally unavailable and another node unexpectedly fails.

---

# 4. HA vs FT vs DR in This Scenario

## High Availability (HA)

**Purpose:** Keep the service available during planned maintenance or a normal component failure.

Examples:

- multiple application instances
- multiple cluster nodes
- load balancer
- active/active or active/passive design
- multi-AZ deployment
- redundant control-plane components
- redundant storage/database nodes

Interview line:

> "HA is what allows me to deliberately remove one component for maintenance without taking the application offline."

---

## Fault Tolerance (FT)

**Purpose:** Continue serving traffic even when another unexpected failure happens during the maintenance.

Example:

~~~text
4 nodes available

Node-4 -> planned maintenance
Node-2 -> unexpected failure

Node-1 + Node-3 must still carry the workload
~~~

FT requires:

- sufficient spare capacity
- redundant components
- failure detection
- automatic failover
- replicated state where needed
- avoidance of single points of failure

Interview line:

> "During an upgrade I do not want all remaining capacity fully consumed. I keep enough redundancy so that an additional unexpected failure can still be tolerated."

---

## Disaster Recovery (DR)

**Purpose:** Recover from a cluster/site/region-level failure.

Examples:

- secondary site/region
- replicated database
- infrastructure-as-code
- backups/snapshots
- configuration backup
- secondary cluster
- DNS/global load-balancer failover

DR is normally **not** the first tool for a routine rolling upgrade.

It is the final safety net if something catastrophic happens.

~~~text
Planned maintenance
       |
       v
      HA
       |
Unexpected component failure
       |
       v
      FT
       |
Cluster / site disaster
       |
       v
      DR
~~~

---

# 5. Generic Upgrade Flow

## Phase 0 — Pre-Upgrade Assessment

Before changing anything:

### Capacity

Check:

- CPU
- memory
- storage
- network
- connection count
- application throughput
- spare node capacity

Question:

> "Can the remaining nodes handle production traffic while one node is unavailable?"

If not, add temporary capacity before starting.

### Compatibility

Validate:

- current version -> target version support
- upgrade hops
- OS compatibility
- runtime compatibility
- drivers
- firmware
- cluster software
- application dependencies
- database compatibility
- agents such as monitoring/security/backup
- API/schema compatibility

### Recovery

Confirm:

- backups are recent
- restore has been tested
- configuration is version-controlled
- infrastructure can be rebuilt
- application rollback exists
- database rollback/failover procedure exists
- DR site/region is healthy if required

### Monitoring

Define success/failure metrics:

- availability
- latency
- error rate
- 5xx errors
- transaction success
- CPU/memory
- node health
- cluster health
- database replication
- queue depth
- critical business transactions

---

# 6. Linux Cluster on AWS EC2

There are two important cases:

1. Plain Linux application servers on EC2 behind an ALB/NLB, usually managed by an Auto Scaling Group
2. A real Linux HA cluster, such as Pacemaker/Corosync, running on EC2

The principles are similar, but quorum and fencing become important in the second case.

---

## 6.1 EC2 Application Cluster Behind a Load Balancer

Example:

~~~text
                    ALB
                     |
                Target Group
            /        |        \
         EC2-1     EC2-2     EC2-3
          App       App       App
         Linux     Linux     Linux
~~~

### Preferred Server Upgrade Method

For major OS/runtime changes, prefer **immutable replacement** rather than logging into every instance and upgrading it in place.

Example:

~~~text
New AMI
  |
  +-- patched OS
  +-- required runtime
  +-- monitoring agent
  +-- security agent
  +-- cluster/application prerequisites
  |
  v
New Launch Template version
  |
  v
Auto Scaling Group Instance Refresh
~~~

Example migration:

~~~text
Old:
EC2-1 v1
EC2-2 v1
EC2-3 v1

Add replacement:
EC2-1 v1
EC2-2 v1
EC2-3 v1
EC2-4 v2

Validate EC2-4
     |
     v
Drain/remove EC2-1

Continue:
v1 v1 v2
  ->
v1 v2 v2
  ->
v2 v2 v2
~~~

Advantages:

- clean rollback
- consistent servers
- easier automation
- avoids configuration drift
- avoids partially upgraded hosts
- old capacity can remain available until validation completes

---

## 6.2 Traffic Draining on EC2

Do not immediately terminate an instance that is serving production requests.

Flow:

~~~text
EC2 instance receives traffic
            |
            v
Deregister from ALB target group
            |
            v
Target enters draining state
            |
            v
Existing requests/connections finish
            |
            v
No new traffic
            |
            v
Upgrade / replace / terminate instance
~~~

Health checks ensure only healthy targets receive new traffic.

---

# 7. What Is the PDB Equivalent on Linux/EC2?

There is **no exact native equivalent** of a Kubernetes PodDisruptionBudget for plain Linux/EC2.

A Kubernetes PDB protects workload availability during voluntary disruption.

Example:

~~~yaml
minAvailable: 2
~~~

For three replicas, the intent is:

~~~text
At least 2 healthy replicas must remain available.
~~~

On EC2, the same objective is achieved using a combination of controls.

## Closest AWS Equivalent

~~~text
Auto Scaling Group desired capacity
          +
Instance Refresh MinHealthyPercentage
          +
replacement batch size
          +
ALB/NLB health checks
          +
target deregistration / connection draining
~~~

Example:

~~~text
Desired capacity = 3
Minimum healthy requirement = 100%

Intent:
Do not remove old healthy capacity until replacement
capacity is ready and healthy.
~~~

A less strict example may allow only a portion of instances to be unavailable, depending on application capacity.

### Concept Mapping

| Kubernetes | Linux / AWS EC2 |
|---|---|
| Deployment replicas | ASG desired capacity |
| PDB minAvailable | ASG minimum healthy percentage / minimum healthy capacity |
| PDB maxUnavailable | Instance Refresh replacement batch / allowed unavailable capacity |
| readinessProbe | ALB/NLB/application health check |
| termination grace period | target deregistration delay / graceful shutdown |
| RollingUpdate | ASG Instance Refresh |
| new node pool | new AMI + Launch Template + ASG capacity |
| reschedule pods | launch replacement EC2 / move service |
| topology spread | multi-AZ ASG placement |

Important:

> This is a conceptual equivalence, not a one-to-one feature mapping.

Interview answer:

> "Linux EC2 does not have a direct PDB object. I achieve the same objective using the Auto Scaling Group minimum healthy capacity, controlled Instance Refresh batches, load-balancer health checks and connection draining. If it is a real Linux HA cluster, I additionally have to protect cluster quorum and fencing."

---

# 8. Linux HA Cluster — Pacemaker/Corosync on EC2

Example:

~~~text
                 VIP / Load Balancer
                        |
          +-------------+-------------+
          |             |             |
        EC2-1         EC2-2         EC2-3
          |             |             |
       Pacemaker      Pacemaker      Pacemaker
       Corosync       Corosync       Corosync
~~~

Before upgrade verify:

- quorum
- current resource ownership
- failed resources
- replication
- fencing/STONITH status
- network latency between members
- application dependencies
- shared/replicated storage status
- backup
- failover health

### Upgrade One Node at a Time

~~~text
EC2-1 -> Active
EC2-2 -> Active
EC2-3 -> Maintenance
~~~

Sequence:

~~~text
Put EC2-3 into standby/maintenance
          |
          v
Move cluster resources away
          |
          v
Verify remaining cluster has quorum
          |
          v
Upgrade cluster software / OS
          |
          v
Reboot if required
          |
          v
Validate node and cluster membership
          |
          v
Return EC2-3 to service
          |
          v
Observe
          |
          v
Proceed to next node
~~~

## Quorum Is Critical

For a three-voter cluster:

~~~text
3 voting nodes
    |
1 node maintenance
    |
2 voting nodes remain
    |
quorum retained
~~~

Never intentionally take two nodes down at the same time unless the cluster design specifically supports it.

### Fencing / STONITH

A real HA cluster needs a reliable way to isolate a failed member so that two nodes do not both think they own the same resource.

This prevents **split brain**.

Interview line:

> "For Pacemaker/Corosync, I would not only check ASG or load-balancer health. I must also protect quorum and verify fencing/STONITH before maintenance, otherwise a network partition can create split-brain risk."

---

# 9. Linux Major OS Upgrade — In Place vs Replacement

For a minor patch:

~~~text
Take node out
   ->
patch
   ->
reboot
   ->
validate
   ->
return
~~~

For a major OS generation change, replacement is usually safer.

Example:

~~~text
Old Cluster                 New Capacity
RHEL 8 EC2-1                RHEL 9 EC2-4
RHEL 8 EC2-2                RHEL 9 EC2-5
RHEL 8 EC2-3                RHEL 9 EC2-6
~~~

Then migrate workloads gradually.

Advantages:

- easier rollback
- no half-upgraded server
- cleaner configuration
- easier repeatability
- old environment can remain available during validation

---

# 10. VMware Cluster Upgrade

Assume:

~~~text
                  vCenter
                     |
        +------------+------------+
        |            |            |
      ESXi-1       ESXi-2       ESXi-3
        |            |            |
       VMs          VMs          VMs
~~~

Key VMware features:

- vCenter
- ESXi
- vMotion
- DRS
- vSphere HA
- VMware FT where appropriate
- vSphere Lifecycle Manager
- shared storage / datastore availability
- backup/replication/DR tooling

---

# 11. VMware Upgrade Sequence

The exact sequence must follow the supported compatibility matrix, but a common architecture-level sequence is:

~~~text
Pre-check compatibility
       |
       v
Upgrade vCenter / management plane
       |
       v
Upgrade ESXi hosts one at a time
       |
       v
Validate cluster
       |
       v
Upgrade VMware Tools where required
       |
       v
Upgrade VM guest OS/applications
       |
       v
Upgrade virtual hardware only when needed
~~~

Do not blindly upgrade virtual hardware early because it may reduce rollback or backward compatibility options.

---

# 12. ESXi Host Rolling Upgrade

Suppose:

~~~text
Before:

ESXi-1
  VM1
  VM2

ESXi-2
  VM3

ESXi-3
  VM4
~~~

Evacuate ESXi-1:

~~~text
vMotion VM1 -> ESXi-2
vMotion VM2 -> ESXi-3
~~~

Now:

~~~text
ESXi-1
  EMPTY

ESXi-2
  VM1
  VM3

ESXi-3
  VM2
  VM4
~~~

Then:

~~~text
ESXi-1
   |
Maintenance Mode
   |
Upgrade ESXi / drivers / firmware / vendor image
   |
Reboot
   |
Health validation
   |
Exit Maintenance Mode
~~~

Only after ESXi-1 is healthy do you proceed to ESXi-2.

This is the VMware equivalent of a rolling node upgrade.

---

# 13. VMware HA vs VMware FT

This is a common interview follow-up.

## VMware HA

If an ESXi host fails:

~~~text
ESXi-2 fails
    |
    v
VMware HA detects host failure
    |
    v
Affected VMs are restarted on another healthy ESXi host
~~~

There is normally a VM restart event.

## VMware Fault Tolerance

Conceptually:

~~~text
Primary VM
   ||
Secondary VM
~~~

The secondary copy runs in lockstep with the primary for supported workloads/configurations.

If the primary host/VM fails, the secondary can continue with extremely small interruption compared with restarting the VM.

Simple interview difference:

> **HA = restart the VM elsewhere.**  
> **FT = another synchronized VM is already running.**

FT is more resource intensive and has product/version/workload limitations, so it is normally used only where the requirement justifies it.

---

# 14. VMware Application Upgrade

After infrastructure is stable, applications can be upgraded independently.

Example:

~~~text
                 Load Balancer
                      |
          +-----------+-----------+
          |           |           |
        VM-1        VM-2        VM-3
       App v1      App v1      App v1
~~~

Take VM-3 out of rotation:

~~~text
Drain VM-3
   |
upgrade App / Guest OS if planned
   |
health check
   |
return VM-3
~~~

Then repeat.

For a high-risk application change, use:

- blue-green
- canary
- parallel application stacks

---

# 15. VMware DR

Possible DR design:

~~~text
Primary vSphere Site
         |
VM/Data Replication
         |
         v
Secondary vSphere Site
~~~

DR may use:

- storage replication
- VM replication
- backup/restore
- VMware disaster-recovery tooling
- Site Recovery Manager / VMware Live Site Recovery depending on platform/version
- DNS/global traffic failover

DR should be tested, not merely documented.

---

# 16. Kubernetes Cluster Upgrade

High-level order:

~~~text
Control Plane
     |
     v
Cluster add-ons
     |
     v
Worker nodes
     |
     v
Applications
~~~

For managed Kubernetes such as EKS/AKS/GKE, the provider manages much of control-plane HA.

For self-managed Kubernetes, the control plane itself must be redundant.

Example:

~~~text
             API Load Balancer
                    |
        +-----------+-----------+
        |           |           |
       CP-1        CP-2        CP-3
~~~

---

# 17. Kubernetes Pre-Checks

Before upgrading:

- supported Kubernetes version path
- deprecated APIs
- CNI compatibility
- CSI compatibility
- CoreDNS
- kube-proxy if applicable
- ingress controllers
- admission webhooks
- CRDs/operators
- service mesh
- monitoring/logging
- storage drivers
- autoscaling components
- application API compatibility

Never assume only the API server matters.

---

# 18. Kubernetes Worker Upgrade

Prefer replacing worker capacity rather than performing uncontrolled in-place upgrades.

Example:

~~~text
Old Pool                    New Pool
Node-1                      Node-4
Node-2                      Node-5
Node-3                      Node-6
~~~

For each old node:

~~~bash
kubectl cordon node-1
kubectl drain node-1
~~~

Conceptual flow:

~~~text
Add new worker capacity
       |
       v
Validate new nodes
       |
       v
Cordon old node
       |
       v
Drain workloads
       |
       v
Pods reschedule
       |
       v
Validate service
       |
       v
Remove old node
~~~

---

# 19. Kubernetes PDB

A PodDisruptionBudget limits how much voluntary disruption Kubernetes should allow for a workload.

Example:

~~~yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: critical-app-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: critical-app
~~~

For three replicas:

~~~text
Pod-1
Pod-2
Pod-3

PDB minAvailable = 2
~~~

During a voluntary node drain, Kubernetes should not intentionally reduce the workload below the allowed availability budget.

Important nuance:

- PDB primarily protects against **voluntary disruptions**
- it does not magically protect against all unplanned failures
- the workload still needs enough replicas
- the cluster needs enough capacity to reschedule pods

---

# 20. Kubernetes Application Upgrade

For normal stateless workloads use rolling deployment.

Example:

~~~text
v1  v1  v1
     |
     v
v1  v1  v2
     |
     v
v1  v2  v2
     |
     v
v2  v2  v2
~~~

For strict availability:

~~~yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1
~~~

Combine with:

- readiness probe
- liveness probe
- startup probe when appropriate
- multiple replicas
- PDB
- anti-affinity or topology spread
- sufficient spare capacity

Traffic must only reach a new pod after it is ready.

---

# 21. Blue-Green Deployment

Useful for critical applications or where instant rollback is important.

~~~text
BLUE
App v1
100% traffic

GREEN
App v2
0% traffic
~~~

Validate GREEN.

Then:

~~~text
BLUE  -> 0%
GREEN -> 100%
~~~

If problems occur:

~~~text
Traffic -> BLUE
~~~

The same principle can be used for:

- application stacks
- EC2 fleets
- node pools
- entire clusters in some designs

---

# 22. Canary Deployment

Gradually increase traffic.

~~~text
95% -> v1
 5% -> v2
~~~

Observe:

- latency
- errors
- resource consumption
- logs
- business transactions

Then:

~~~text
5% -> 10% -> 25% -> 50% -> 100%
~~~

Rollback by directing traffic back to the old version.

---

# 23. Stateful Applications

Zero-downtime upgrade is harder for stateful systems.

Examples:

- databases
- Kafka
- Elasticsearch
- distributed caches
- clustered middleware

Check:

- replication
- quorum
- leader/follower behavior
- schema compatibility
- persistent storage backup
- supported rolling upgrade sequence
- application backward compatibility

Database changes should ideally follow an **expand-and-contract** model.

Example:

~~~text
Add backward-compatible schema
         |
         v
Deploy new application
         |
         v
Validate
         |
         v
Remove obsolete schema later
~~~

Avoid making an incompatible database schema change at the same instant as the application cutover.

---

# 24. Monitoring and Go/No-Go Gates

After every batch:

~~~text
Upgrade one component/batch
        |
        v
Observe
        |
        v
Validate
        |
   +----+----+
   |         |
Pass       Fail
 |           |
Next       Rollback
batch
~~~

Monitor four layers:

## Infrastructure

- CPU
- memory
- network
- storage
- host health

## Cluster

- quorum
- node health
- control-plane health
- scheduling/placement
- replication

## Application

- availability
- latency
- error rate
- restart count
- connection failures

## Business

- login success
- checkout/transaction success
- message processing
- critical user journey

---

# 25. Rollback Matrix

| Layer | Rollback |
|---|---|
| Linux EC2 application fleet | return traffic to old instances / old AMI or Launch Template |
| Linux HA cluster | return upgraded node to previous supported state or fail resources back to healthy members |
| VMware ESXi | keep old hosts/capacity until validation; follow supported rollback/recovery method |
| Guest OS | restore/rebuild using previous image/backup |
| Kubernetes worker | move workload back to previous node pool |
| Kubernetes application | previous image / Helm release / deployment revision |
| Database | failover/replica/backup depending on product |
| Persistent data | snapshot / backup |
| Full cluster/site failure | DR environment / rebuild via IaC |

Important:

> Do not assume every control plane or data-layer change supports an easy downgrade. That is why pre-upgrade compatibility testing and DR readiness matter.

---

# 26. Recommended End-to-End Upgrade Strategy

~~~text
1. Assess dependencies
2. Check compatibility
3. Verify HA/FT design
4. Verify backup and DR
5. Add temporary capacity if required
6. Upgrade management/control layer
7. Validate
8. Upgrade one server/node at a time
9. Validate after every node
10. Upgrade applications in controlled waves
11. Upgrade stateful applications with product-specific procedure
12. Validate business transactions
13. Keep old capacity for a soak period
14. Remove old capacity only after stability is proven
~~~

---

# 27. Interview-Ready 2–3 Minute Answer

> "I would first clarify what kind of cluster we are talking about because the implementation differs between Linux, VMware and Kubernetes. But I would use the same overall principle: no big-bang upgrade. I would separate the activity into core platform software, underlying servers or nodes, and applications, and I would upgrade one failure domain at a time.
>
> Before starting I would check compatibility, cluster health, spare capacity, backups, rollback and DR readiness. I would make sure the application is actually highly available and that the remaining nodes can carry production traffic while one node is unavailable. For a critical environment I would maintain N+1 or N+2 capacity.
>
> If these are Linux EC2 servers behind a load balancer, I would prefer immutable replacement using a new AMI and Launch Template, followed by a controlled Auto Scaling Group Instance Refresh. I would use minimum healthy capacity, health checks and connection draining before removing old instances. If it is Pacemaker/Corosync, I also have to preserve quorum and validate fencing.
>
> If it is VMware, I would normally upgrade the management layer according to VMware's supported compatibility sequence, then evacuate each ESXi host using vMotion, put it into maintenance mode, upgrade and validate it, and only then continue to the next host. VMware HA protects against host failures, while VMware FT can protect selected critical VMs with a synchronized secondary instance.
>
> If it is Kubernetes, I would upgrade the control plane first, validate add-on compatibility, then roll or replace worker nodes using cordon and drain, and finally upgrade applications using rolling, canary or blue-green deployments. PDBs, readiness probes, multiple replicas and spare capacity protect application availability.
>
> HA keeps the service online during planned maintenance, FT ensures I can tolerate another unexpected failure while the maintenance is happening, and DR is my last-resort protection if the cluster, site or region suffers a major failure. After every batch I would have a go/no-go checkpoint and a rollback plan before continuing."

---

# 28. Follow-Up Interview Q&A

## Q1. Why not upgrade core software, servers and applications together?

Because it increases the blast radius and makes troubleshooting extremely difficult.

If something fails, you will not know whether it was caused by:

- platform software
- OS/kernel
- server firmware
- network driver
- runtime
- application

Separating the layers gives clear rollback points.

---

## Q2. What if there is not enough capacity to take one node down?

Add temporary capacity first.

Do not begin the rolling upgrade if the remaining nodes cannot safely carry peak traffic.

Interview phrase:

> "Capacity validation is a prerequisite, not something I discover after draining the first node."

---

## Q3. Would the approach be the same if the Linux machines are EC2 instances?

Yes, the rolling principle remains the same, but AWS lets me automate it.

I would normally use:

~~~text
AMI
 ->
Launch Template
 ->
Auto Scaling Group
 ->
Instance Refresh
 ->
Target Group health check
 ->
Connection draining
~~~

For major OS upgrades I prefer replacement over in-place patching.

---

## Q4. What is the PDB equivalent for Linux servers on EC2?

There is no exact equivalent.

The closest practical combination is:

- ASG minimum healthy capacity / MinHealthyPercentage
- Instance Refresh batch control
- load-balancer health checks
- target deregistration delay
- graceful application shutdown
- quorum rules for a real Linux HA cluster

---

## Q5. Is MinHealthyPercentage exactly the same as a PDB?

No.

A PDB is a Kubernetes workload-level policy for voluntary disruptions.

ASG minimum healthy capacity is an infrastructure fleet-level control.

They solve a similar availability objective at different layers.

---

## Q6. What if these EC2 servers run Pacemaker/Corosync?

Then protecting instance count is not enough.

I must also protect:

- quorum
- fencing/STONITH
- resource ownership
- replicated state
- failover behavior

I would place one cluster member into maintenance/standby, move resources away, upgrade it, validate its rejoin, and then continue to the next node.

---

## Q7. Why is fencing important?

To avoid split brain.

A failed or network-isolated node must not continue modifying a resource while another node has taken ownership.

Fencing forcibly isolates the bad member.

---

## Q8. Can I use an ASG for a Pacemaker cluster exactly like stateless web servers?

Not blindly.

Stateless instances are easily replaceable.

Pacemaker members may have:

- stable identity requirements
- quorum membership
- fencing configuration
- persistent state
- resource placement rules

ASG automation must respect the cluster's membership and failover design.

---

## Q9. For VMware, why use vMotion before upgrading an ESXi host?

To evacuate running VMs from the host so the host can enter maintenance mode without application outage.

---

## Q10. What if vMotion is not possible?

Then I need another strategy depending on the reason:

- fix the networking/storage compatibility problem
- gracefully stop/fail over the application
- use application-level HA
- schedule downtime if no redundancy exists

Zero downtime cannot be guaranteed if neither infrastructure-level nor application-level mobility/redundancy exists.

---

## Q11. Difference between VMware HA and VMware FT?

**VMware HA:** restarts a VM on another host after failure.

**VMware FT:** maintains a synchronized secondary VM for supported workloads so failover can happen without waiting for a normal VM restart.

---

## Q12. Is VMware FT used for every VM?

Usually no.

It consumes additional resources and has configuration/product limitations.

Use it where the business requirement justifies the cost and complexity.

---

## Q13. Why upgrade vCenter before ESXi?

Because the management plane must support the ESXi versions it manages.

The exact supported ordering must always be verified against the VMware compatibility matrix for the source and target versions.

---

## Q14. Should I upgrade VMware Tools and virtual hardware immediately?

Not necessarily.

VMware Tools can be upgraded when required/appropriate, but virtual hardware upgrades should be deliberate because they can affect rollback and compatibility with older hosts.

---

## Q15. Kubernetes — why control plane before workers?

Workers must communicate with and be supported by the control plane.

The upgrade must follow Kubernetes/provider-supported version-skew rules and upgrade paths.

---

## Q16. What about Kubernetes add-ons?

They are part of the upgrade plan.

Check:

- CNI
- CSI
- CoreDNS
- kube-proxy
- ingress
- operators
- service mesh
- monitoring
- admission webhooks
- CRDs

A control-plane upgrade alone is not the full platform upgrade.

---

## Q17. What happens during kubectl drain?

Kubernetes attempts to evict evictable workloads from the node so they can run elsewhere, subject to policies such as PDBs and workload/controller behavior.

After drain, the node can be patched, upgraded or removed.

---

## Q18. Does a PDB guarantee zero downtime?

No.

You still need:

- multiple replicas
- enough cluster capacity
- working readiness checks
- correct service/load-balancer behavior
- compatible application design

A PDB alone cannot protect a single-replica application.

---

## Q19. What if the PDB blocks node drain?

Do not immediately delete the PDB.

First determine why the workload cannot maintain its required availability.

Possible causes:

- insufficient replicas
- insufficient worker capacity
- unhealthy pods
- placement constraints
- application issue

Fix the availability problem before forcing disruption unless there is an emergency and the business accepts the risk.

---

## Q20. How would you keep the application online while upgrading the application itself?

Use:

- rolling deployment
- blue-green
- canary
- active/active

The load balancer should only send traffic to healthy instances.

For strict rolling availability, bring the new instance up and validate it before taking the old instance down.

---

## Q21. What if the application has only one instance?

Then true zero downtime is generally not achievable during a disruptive upgrade.

I would first introduce redundancy or use a parallel blue-green stack before changing the existing instance.

---

## Q22. What about database upgrades?

Use the database vendor's supported HA/rolling-upgrade procedure.

Check:

- replication
- quorum
- primary/secondary roles
- schema compatibility
- backup
- restore
- failover

Do not treat a database like a stateless application server.

---

## Q23. What about schema changes?

Prefer backward-compatible changes.

Use expand-and-contract:

~~~text
Add compatible schema
     ->
deploy new app
     ->
validate
     ->
remove obsolete schema later
~~~

---

## Q24. What if the upgrade fails halfway through?

Stop the rollout.

Do not continue to the next batch.

Use the defined rollback path:

~~~text
Failure
  |
Stop rollout
  |
Isolate bad version
  |
Return traffic/workload to healthy version
  |
Validate
  |
Investigate
~~~

---

## Q25. When would you use DR during an upgrade?

DR is for major failure, for example:

- entire cluster unavailable
- storage corruption
- database corruption
- site failure
- region failure
- unrecoverable platform failure

A normal node failure during a rolling upgrade should usually be handled by HA/FT rather than immediately invoking DR.

---

## Q26. Would you perform the upgrade in all availability zones at once?

No.

For critical environments, preserve failure-domain separation.

A safer approach is controlled batches across AZs/racks/fault domains so a bad upgrade does not remove all redundant capacity simultaneously.

---

## Q27. How do you decide the batch size?

Based on:

- redundancy
- minimum healthy capacity
- peak utilization
- application replica count
- failure-domain distribution
- recovery time
- operational risk

For highly critical systems, begin with one node or a very small canary batch.

---

## Q28. What metrics decide whether you proceed?

Examples:

- application availability
- latency
- 4xx/5xx depending on expected behavior
- transaction success
- node/host health
- cluster health
- database replication lag
- resource utilization
- logs/events
- business KPI or synthetic test

---

## Q29. What is your rollback point?

Every layer has its own rollback point.

Do not wait until the entire upgrade is complete.

Example:

~~~text
Control layer -> validate -> checkpoint
Host batch    -> validate -> checkpoint
App batch     -> validate -> checkpoint
~~~

---

## Q30. What is the single most important principle?

> "Never consume all of your redundancy during the upgrade."

If planned maintenance already uses all spare capacity, one unexpected failure becomes an outage.

---

# 29. Fast Comparison Table

| Area | Linux on EC2 | VMware | Kubernetes |
|---|---|---|---|
| Management/core layer | cluster software / runtime / AMI dependencies | vCenter | control plane |
| Server/node | EC2 | ESXi host | worker node |
| Rolling mechanism | ASG Instance Refresh or one-node maintenance | vMotion + Maintenance Mode + host upgrade | cordon + drain / replacement node pool |
| Health gate | ALB/NLB/app check | vCenter/host/VM/app health | readiness + node/pod health |
| Availability control | min healthy capacity + LB drain | HA/DRS/capacity | replicas + PDB + rollout strategy |
| FT example | N+1/N+2, replicated service | VMware FT for supported VMs | redundant pods/nodes/control plane |
| Failure-domain design | Multi-AZ | hosts/racks/sites | nodes/AZs |
| DR | second region/site + backup/IaC | secondary vSphere site/replication | secondary cluster/region + backup/GitOps/IaC |
| Preferred major OS/node change | immutable replacement | rolling host remediation | replace node pool/workers |
| Critical extra concern | quorum/fencing for Linux HA | storage/vMotion compatibility | add-on/version compatibility |

---

# 30. One-Line Memory Aid

~~~text
PRECHECK
   ->
CAPACITY
   ->
CORE PLATFORM
   ->
SERVERS/NODES
   ->
APPLICATIONS
   ->
VALIDATE EACH BATCH
   ->
ROLLBACK IF REQUIRED
   ->
DR ONLY FOR MAJOR FAILURE
~~~

And remember:

> **HA keeps service available during planned maintenance. FT protects you from an additional unexpected failure. DR recovers you from a major cluster/site disaster.**
