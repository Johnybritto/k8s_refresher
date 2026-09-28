# OpenShift on VMware vSphere — Production Upgrade Runbook

## Purpose

This document is a detailed production-grade runbook for upgrading an OpenShift cluster running on **VMware vSphere**. It covers:

- Upgrade planning and supported path validation
- Cluster and vSphere prechecks
- Third-party dependency handling
- Dynatrace compatibility and upgrade sequencing
- Backup and rollback preparation
- OpenShift upgrade execution
- CVO/MCO behavior during the upgrade
- Node rolling updates and reboots
- Monitoring and troubleshooting
- Post-upgrade validation

> Important: OpenShift does **not** support routine downgrade to an earlier cluster version. An etcd restore is a disaster-recovery procedure, not a normal rollback strategy. The real safeguards are compatibility validation, backups, rehearsal, and abort criteria before starting the production upgrade.

---

# 1. End-to-End Upgrade Flow

```text
Identify current + target OCP version
          ↓
Check supported upgrade path
          ↓
Read target release notes / known issues
          ↓
Inventory all third-party Operators/components
          ↓
Build compatibility matrix
          ↓
Upgrade incompatible dependencies FIRST
          ↓
Validate cluster health
          ↓
Validate vSphere compatibility
          ↓
Check PDBs / capacity / application resilience
          ↓
Take etcd + application/data backups
          ↓
Capture pre-upgrade baseline
          ↓
Pause MachineHealthChecks
          ↓
Freeze unrelated changes
          ↓
Trigger OpenShift upgrade
          ↓
CVO updates platform components
          ↓
MCO rolls node configuration / RHCOS
          ↓
Nodes cordon → drain → update → reboot → uncordon
          ↓
Monitor CO / MCP / Nodes / Applications
          ↓
Validate target OpenShift version
          ↓
Validate vSphere / storage / ingress / applications
          ↓
Validate Dynatrace and other dependencies
          ↓
Resume MachineHealthChecks
          ↓
Observation period
          ↓
Change complete
```

---

# 2. Identify the Current Cluster State

Check the current cluster version:

```bash
oc get clusterversion
oc version
```

Check nodes:

```bash
oc get nodes -o wide
```

Check Cluster Operators:

```bash
oc get co
```

Check MachineConfigPools:

```bash
oc get mcp
```

Check current update information:

```bash
oc adm upgrade
```

The goal is to establish:

```text
Current OpenShift version
Current Kubernetes version
Current RHCOS level
Current Operator health
Current node health
Current MCP state
```

Do not choose a target version simply because it exists. OpenShift upgrades must follow the supported update graph.

---

# 3. Check the Supported Upgrade Path

Run:

```bash
oc adm upgrade recommend
```

This is the preferred pre-upgrade assessment because it highlights:

- Available/recommended versions
- Cluster health issues
- Known risks
- Critical alerts
- PodDisruptionBudget risks
- Image pull issues
- Update preconditions

Also check:

```bash
oc adm upgrade
```

Verify:

```text
Current version
Current channel
Recommended target versions
```

Example supported path:

```text
4.20.x
   ↓
4.21.x
   ↓
4.22.x
```

Do not skip unsupported versions.

Avoid using:

```bash
oc adm upgrade --force
```

unless it is part of a specific Red Hat-supported recovery procedure.

---

# 4. Review Target Release Notes and Known Issues

Before the maintenance window review:

```text
Known issues
Removed APIs
Deprecated APIs
Administrator acknowledgements
RHCOS changes
Networking changes
Storage changes
Operator changes
vSphere requirements
```

Version-specific administrator acknowledgements can block a minor upgrade until accepted.

The key rule is:

> Check release-specific upgrade acknowledgements before every minor OpenShift upgrade.

---

# 5. Validate VMware vSphere Compatibility

Before upgrading OpenShift, verify the underlying VMware platform supports the target OpenShift release.

Check:

```text
vCenter version
ESXi version
VM virtual hardware version
vSphere CSI compatibility
Datastores
Storage policy
Networking
vCenter connectivity
```

Do not combine major vSphere/ESXi changes with the OpenShift upgrade unless explicitly planned and validated.

Bad maintenance pattern:

```text
OpenShift upgrade
+
ESXi patching
+
network migration
+
storage upgrade
```

Prefer one major failure domain at a time.

---

# 6. Inventory Every OpenShift Dependency

Do not only check Red Hat Cluster Operators.

List installed Operators:

```bash
oc get csv -A
oc get subscriptions -A
oc get crd
```

Inventory components such as:

```text
Dynatrace
OpenShift Data Foundation / ODF
Portworx
NetApp Trident
OADP / Velero
Red Hat Advanced Cluster Security
Red Hat Advanced Cluster Management
OpenShift GitOps / Argo CD
OpenShift Service Mesh
OpenShift Logging / Splunk
Vault
cert-manager
Kafka / AMQ Operators
Database Operators
Third-party CSI drivers
Security agents
Backup agents
```

Why this matters:

```text
Current Operator supports OCP 4.21 ✅
Current Operator supports OCP 4.22 ❌

Upgrade OCP first
        ↓
Operator becomes unsupported/broken
        ↓
Application/platform failure
```

---

# 7. Build a Compatibility Matrix

Create a matrix before production change approval.

| Component | Current Version | Supports Current OCP | Supports Target OCP | Required Action |
|---|---:|---:|---:|---|
| OpenShift | current | Yes | target | Upgrade |
| Dynatrace Operator | current | Check | Check | Upgrade first if needed |
| OneAgent | current | Check | Check | Upgrade if needed |
| ActiveGate | current | Check | Check | Upgrade if needed |
| ODF | current | Check | Check | Upgrade if needed |
| GitOps | current | Check | Check | Upgrade if needed |
| ACS | current | Check | Check | Upgrade if needed |
| Other Operators | current | Check | Check | Upgrade if needed |

The ideal state is:

```text
Dependency version X

supports:
Current OCP ✅
Target OCP  ✅
```

Then:

```text
Upgrade dependency first
        ↓
Validate it on current OCP
        ↓
Upgrade OpenShift
        ↓
Validate dependency again
```

This overlap strategy is much safer than upgrading OpenShift and add-ons simultaneously.

---

# 8. Dynatrace Dependency Workflow

Dynatrace should be treated as a cluster dependency, not as an afterthought.

First determine how it is installed.

```bash
oc get pods -n dynatrace
oc get dynakube -A
oc get csv -A | grep -i dynatrace
oc get subscriptions -A | grep -i dynatrace
helm list -n dynatrace
```

Identify whether the deployment uses:

```text
OLM / OperatorHub
Helm
Raw manifests
```

Also identify the components:

```text
Dynatrace Operator
OneAgent
ActiveGate
CSI driver
DynaKube CRs
Other Dynatrace components
```

---

# 9. Check Dynatrace CRD/API Compatibility

Check DynaKube API versions:

```bash
oc get dynakubes -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,API:.apiVersion'
```

Check the stored CRD versions:

```bash
oc get crd dynakubes.dynatrace.com \
  -o jsonpath='{.status.storedVersions}'
```

Older DynaKube stored versions can require an intermediate Dynatrace Operator upgrade for CRD migration.

The principle is:

```text
Old CRD format
      ↓
Intermediate supported Operator
      ↓
CRD migration
      ↓
New Operator
      ↓
Target OpenShift
```

Never skip a documented CRD migration step.

---

# 10. Upgrade Dynatrace Before OpenShift if Needed

If the current Dynatrace version does not support the target OpenShift release:

```text
Dynatrace upgrade FIRST
        ↓
Validate
        ↓
OpenShift upgrade
```

## OLM installation

Check Subscription:

```bash
oc get subscription -n <namespace>
```

Check InstallPlans:

```bash
oc get installplan -n <namespace>
```

If approval is manual, review and approve the required InstallPlan only after verifying compatibility.

## Helm installation

Use the vendor-approved compatible chart/operator version and current values file.

Conceptually:

```bash
helm upgrade dynatrace-operator <chart> \
  --version <approved-compatible-version> \
  --namespace dynatrace \
  -f values.yaml
```

Avoid blindly reusing old chart values across major Operator changes without checking the new chart schema.

## Manifest installation

Apply the vendor-supported manifests for the approved target Operator version.

After the Dynatrace upgrade, validate:

```bash
oc get pods -n dynatrace
oc get ds -n dynatrace
oc get deploy -n dynatrace
oc get dynakube -A
```

And confirm telemetry in the Dynatrace UI before continuing with OpenShift.

---

# 11. Baseline OpenShift Health

Before starting the upgrade:

```bash
oc get co
```

Target state:

```text
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

Check nodes:

```bash
oc get nodes
```

All nodes should be `Ready`.

Check MCPs:

```bash
oc get mcp
```

Target state:

```text
UPDATED=True
UPDATING=False
DEGRADED=False
```

Check pending CSRs:

```bash
oc get csr
```

Investigate unexplained pending CSRs before proceeding.

---

# 12. Validate Application Resilience and PDBs

The upgrade can reboot worker nodes, so applications must tolerate a node disappearing temporarily.

Check PodDisruptionBudgets:

```bash
oc get pdb -A
```

Risk example:

```text
Application replicas = 1
PDB minAvailable = 1
        ↓
Node drain attempts pod eviction
        ↓
Eviction blocked
        ↓
Node update stalls
```

Check Deployments and StatefulSets:

```bash
oc get deploy -A
oc get sts -A
```

Work with application owners where necessary to:

```text
Increase replicas
Correct PDB settings
Move workloads
Adjust disruption policies
```

Do not delete PDBs blindly.

---

# 13. Validate Cluster Capacity

During a rolling worker update, remaining nodes must absorb workloads from the node being drained.

Check:

```bash
oc adm top nodes
oc adm top pods -A
```

Example risk:

```text
Worker-1 drains
      ↓
Pods reschedule
      ↓
Worker-2 / Worker-3 lack CPU or memory
      ↓
Pods remain Pending
```

Make sure sufficient spare compute capacity exists before the upgrade.

---

# 14. Take an etcd Backup

Create a fresh etcd backup immediately before the maintenance operation.

Find masters:

```bash
oc get nodes -l node-role.kubernetes.io/master
```

Debug a healthy master:

```bash
oc debug --as-root node/<master-node>
```

Then:

```bash
chroot /host
/usr/local/bin/cluster-backup.sh /home/core/assets/backup
```

The backup contains:

```text
etcd snapshot
+
static Kubernetes resources
```

Important:

> etcd backup is not a full application-data backup.

Also back up where applicable:

```text
Databases
PVC data / storage snapshots
External databases
Application-specific state
Critical configuration
Git-managed configuration
```

---

# 15. Understand the Rollback Limitation

OpenShift does not support routine downgrade to an earlier version.

Do not plan:

```text
Upgrade fails
      ↓
Simply downgrade OpenShift
```

Instead plan:

```text
Strong prechecks
Compatibility validation
Lower-environment rehearsal
Fresh backups
Clear go/no-go criteria
Early abort criteria
Vendor support escalation paths
```

An etcd restore is a disaster-recovery mechanism, not a normal version rollback procedure.

---

# 16. Capture Pre-Upgrade Baseline

Save the current state:

```bash
oc get clusterversion -o yaml > clusterversion-before.yaml
oc get co > clusteroperators-before.txt
oc get nodes -o wide > nodes-before.txt
oc get mcp > mcp-before.txt
oc get csv -A > operators-before.txt
oc get storageclass > storage-before.txt
```

Also record:

```text
Application smoke-test results
Dynatrace dashboards
Current alerts
Node utilization
Ingress health
Storage health
API latency
```

This makes post-upgrade comparison easier.

---

# 17. Pause MachineHealthChecks

During the upgrade, nodes may intentionally reboot and temporarily become `NotReady`.

A MachineHealthCheck can mistake this for a real failure and attempt remediation.

Conceptually:

```text
Node reboot for upgrade
       ↓
Node temporarily NotReady
       ↓
MachineHealthCheck detects failure
       ↓
Unwanted remediation/replacement
```

Pause applicable MachineHealthChecks according to the approved procedure and record their original state so they can be restored afterward.

---

# 18. Apply a Maintenance Freeze

Before starting:

```text
Stop unrelated MachineConfig changes
Stop cluster configuration changes
Stop Operator upgrades unrelated to this change
Stop manual node reboots
Stop vSphere maintenance on OpenShift hosts
Stop storage/network changes
Inform application teams and monitoring/NOC teams
```

The goal is to make every observed change attributable to the OpenShift upgrade.

---

# 19. Update the oc Client

Use an `oc` version appropriate for the target OpenShift release.

Check:

```bash
oc version
```

---

# 20. Set the Approved Upgrade Channel

Examples:

```text
stable-4.x
fast-4.x
eus-4.x
```

For example:

```bash
oc adm upgrade channel stable-4.22
```

Then re-run:

```bash
oc adm upgrade recommend
```

Do not automatically choose `latest` without change-control approval.

---

# 21. Trigger the OpenShift Upgrade

Upgrade to the approved recommended version:

```bash
oc adm upgrade --to=<approved-version>
```

Example:

```bash
oc adm upgrade --to=4.22.x
```

Avoid `--force` unless explicitly directed by Red Hat support for a recovery situation.

---

# 22. What Happens Internally — CVO

The OpenShift upgrade is orchestrated by the **Cluster Version Operator (CVO)**.

```text
oc adm upgrade
      ↓
ClusterVersion desired version changes
      ↓
Cluster Version Operator
      ↓
Target release payload
      ↓
Release manifests / dependency ordering
      ↓
Cluster Operators update
```

CVO manages:

```text
Platform release version
Cluster Operators
Release manifests
Dependency-aware update sequencing
```

A useful mental model is:

```text
Foundational components
       ↓
Dependent control-plane components
       ↓
Higher-level Operators
```

Do not memorize a fixed operator-by-operator order because the dependency graph is encoded in the release payload.

---

# 23. What Happens Internally — MCO

The **Machine Config Operator (MCO)** manages node-level configuration and RHCOS changes.

It can update:

```text
RHCOS
kubelet
CRI-O
kernel
systemd configuration
NetworkManager configuration
MachineConfig-managed files/settings
```

For a disruptive node update, the high-level sequence is:

```text
Node
 ↓
CORDON
 ↓
DRAIN
 ↓
Apply new machine configuration / OS
 ↓
REBOOT
 ↓
Node returns Ready
 ↓
UNCORDON
```

Remember:

```text
CVO = OpenShift platform version / Operators
MCO = Node operating system / machine configuration
```

---

# 24. Control-Plane Rolling Upgrade

With three masters:

```text
master-0
master-1
master-2
```

OpenShift must preserve:

```text
etcd quorum
API availability
```

Conceptually:

```text
Master-0 update/reboot
       ↓
Master-0 Ready
       ↓
Master-1 update/reboot
       ↓
Master-1 Ready
       ↓
Master-2 update/reboot
       ↓
Master-2 Ready
```

For a three-member etcd cluster:

```text
3 members
   ↓
minimum 2 available for quorum
```

Never intentionally reboot or remove multiple control-plane nodes simultaneously during a normal upgrade.

---

# 25. Worker Rolling Upgrade

Workers follow the MCO update process:

```text
Worker
  ↓
cordon
  ↓
drain
  ↓
pods reschedule elsewhere
  ↓
RHCOS / MachineConfig update
  ↓
reboot
  ↓
kubelet starts
  ↓
Ready
  ↓
uncordon
```

MachineConfigPool settings such as `maxUnavailable` control how many nodes can be unavailable during a pool rollout.

---

# 26. What Happens to Dynatrace During Worker Reboots

If OneAgent runs as a DaemonSet:

```text
Worker-1
   ↓
OneAgent pod
```

During upgrade:

```text
Worker-1 drains
      ↓
OneAgent stops on that node
      ↓
Worker-1 reboots
      ↓
kubelet returns
      ↓
DaemonSet starts OneAgent again
      ↓
monitoring resumes
```

A short per-node monitoring gap can be expected during reboot.

If ActiveGate runs with multiple replicas:

```text
ActiveGate pod on draining worker
        ↓
rescheduled to another healthy worker
```

provided capacity and scheduling constraints allow it.

---

# 27. Dynatrace CSI Considerations

If Dynatrace uses CSI-based injection, also monitor the CSI components during worker updates.

Check:

```bash
oc get pods -n dynatrace -o wide
oc get ds -n dynatrace
```

Expected sequence:

```text
Worker reboot
     ↓
CSI node component stops
     ↓
RHCOS returns
     ↓
kubelet starts
     ↓
CSI DaemonSet returns
     ↓
OneAgent/code module functionality restored
```

Do not intentionally upgrade the Dynatrace Operator and OpenShift at the same time.

Preferred sequence:

```text
Upgrade Dynatrace if required
       ↓
Validate Dynatrace
       ↓
Upgrade OpenShift
       ↓
Validate Dynatrace again
```

---

# 28. Monitor the Upgrade

Use:

```bash
watch oc adm upgrade status
```

Also monitor:

```bash
watch oc get co
watch oc get mcp
watch oc get nodes
```

Check events:

```bash
oc get events -A --sort-by='.lastTimestamp'
```

Useful interpretation:

```text
PROGRESSING=True
```

can be normal during an upgrade.

But:

```text
DEGRADED=True
```

requires investigation.

For MCPs:

```text
UPDATED=False
UPDATING=True
DEGRADED=False
```

is often expected while node updates are in progress.

---

# 29. If the Upgrade Gets Stuck

Do not immediately force the upgrade.

First identify the failing layer:

```bash
oc adm upgrade status
oc get co
oc get mcp
oc get nodes
```

If an Operator is degraded:

```bash
oc describe co <operator-name>
```

If the MCO/MCP is degraded:

```bash
oc describe co machine-config
oc get mcp
oc get pods -n openshift-machine-config-operator -o wide
```

Follow the dependency chain instead of randomly restarting components.

---

# 30. Common Failure — Drain Blocked by PDB

Symptom:

```text
Node cordoned
Drain starts
Pod eviction fails
Upgrade stops progressing
```

Check:

```bash
oc get pdb -A
oc describe pdb <name> -n <namespace>
```

Possible actions after application-owner review:

```text
Increase replicas
Correct PDB
Temporarily adjust disruption policy
Move workload
```

Do not delete PDBs blindly.

---

# 31. Common Failure — Node Does Not Return

If a node remains `NotReady` after reboot:

Check vSphere first:

```text
VM powered on?
Correct network attached?
Disk healthy?
ESXi host healthy?
Datastore accessible?
```

Then OpenShift:

```bash
oc describe node <node>
```

If node access is available:

```bash
journalctl -b -u kubelet
journalctl -b -u crio
```

Also inspect MachineConfigDaemon status and logs.

---

# 32. Common Failure — Cluster Operator Degraded

Example:

```text
network
AVAILABLE=True
PROGRESSING=True
DEGRADED=True
```

Check:

```bash
oc describe co network
oc get pods -n openshift-network-operator
```

Then inspect the relevant Operator/controller logs.

Do not randomly reboot nodes because a Cluster Operator is degraded.

---

# 33. Confirm Upgrade Completion

Check:

```bash
oc adm upgrade
oc get clusterversion
```

Confirm the desired and current versions match the approved target.

Check nodes:

```bash
oc get nodes -o wide
```

Verify all nodes have returned and are running the expected Kubernetes/RHCOS level.

---

# 34. Validate MachineConfigPools

Run:

```bash
oc get mcp
```

Final state should be:

```text
UPDATED=True
UPDATING=False
DEGRADED=False
```

for all required pools.

---

# 35. Validate Cluster Operators

Run:

```bash
oc get co
```

Target:

```text
AVAILABLE=True
PROGRESSING=False
DEGRADED=False
```

Pay particular attention to:

```text
etcd
kube-apiserver
authentication
network
machine-config
storage
ingress
image-registry
monitoring
console
```

---

# 36. Validate OpenShift Functionality

Validate at minimum:

```text
API
Console
Authentication
DNS
Routes
Ingress
Service networking
Pod-to-pod networking
Registry
Storage/PVC provisioning
Pod scheduling
Node networking
```

Run application smoke tests before closing the change.

---

# 37. Validate vSphere Integration

Check:

```text
VM health
vCenter connectivity
Datastore health
vSphere CSI
StorageClasses
Machine API where applicable
```

For IPI clusters:

```bash
oc get machines -n openshift-machine-api
oc get machinesets -n openshift-machine-api
```

Confirm vSphere provisioning and CSI remain healthy after the upgrade.

---

# 38. Validate Dynatrace After OpenShift Upgrade

Run:

```bash
oc get pods -n dynatrace
oc get ds -n dynatrace
oc get deploy -n dynatrace
oc get dynakube -A
```

Validate:

```text
Dynatrace Operator healthy
OneAgent present on expected nodes
ActiveGate healthy
CSI components healthy if used
No webhook failures
No DynaKube reconciliation errors
```

Then validate the Dynatrace UI:

```text
All OpenShift nodes reporting
Kubernetes metrics reporting
Application traces arriving
Logs arriving
Infrastructure metrics arriving
No continuing telemetry gaps
```

A short monitoring interruption while a node reboots may be normal. A continuing telemetry gap after the node is healthy is not.

---

# 39. Re-enable MachineHealthChecks

After confirming:

```text
Cluster stable
Nodes Ready
Cluster Operators healthy
MCPs healthy
Applications healthy
Dynatrace healthy
```

restore the MachineHealthChecks that were paused.

Verify:

```bash
oc get machinehealthcheck -A
```

---

# 40. Observation Period

Do not close the change immediately after `oc get nodes` becomes green.

Continue monitoring:

```text
Cluster alerts
API latency
etcd health
Node CPU/memory
Ingress
Storage
Application errors
Application transaction rate
Dynatrace health
```

Compare results against the pre-upgrade baseline.

---

# Dependency Upgrade Rule

Use this pattern for any cluster-integrated component:

```text
                  Supports
               OLD OCP   NEW OCP

Old dependency    ✅        ❌
       |
       | upgrade dependency first
       ↓
New dependency    ✅        ✅
       |
       | upgrade OpenShift
       ↓
New OCP + New dependency
                 ✅
```

This applies to:

```text
Dynatrace
ODF
ACM
ACS
GitOps
Logging
Service Mesh
Portworx
Trident
Splunk
Vault
Kafka Operators
Database Operators
Backup Operators
Third-party CSI drivers
```

---

# CVO vs MCO — Interview Memory Aid

```text
CVO
Cluster Version Operator
        ↓
OpenShift release
Cluster Operators
Release manifests
Dependency ordering
```

```text
MCO
Machine Config Operator
        ↓
RHCOS
kubelet
CRI-O
MachineConfig
Node reboot / rollout
```

Combined flow:

```text
oc adm upgrade
      ↓
CVO
      ↓
OpenShift platform components
      ↓
MCO
      ↓
Masters + Workers
      ↓
cordon
      ↓
drain
      ↓
update
      ↓
reboot
      ↓
uncordon
```

---

# Production Interview Answer

If asked **“How do you upgrade a production OpenShift cluster with third-party dependencies?”**, answer:

> I first identify the current and target OpenShift versions and use `oc adm upgrade recommend` to confirm the supported update path and cluster-specific risks. I review release notes, API removals, vSphere prerequisites and administrator acknowledgements.
>
> Before upgrading OpenShift, I inventory all layered products and third-party Operators such as Dynatrace, storage, backup, security and GitOps components. I build a compatibility matrix and, wherever possible, move each dependency to a version that supports both the current and target OpenShift versions.
>
> For Dynatrace, I validate the Operator, OneAgent, ActiveGate, DynaKube CRD/API version and CSI compatibility. If the current version does not support the target OpenShift release, I upgrade Dynatrace first and validate monitoring before starting the OpenShift upgrade.
>
> I then ensure Cluster Operators, nodes and MachineConfigPools are healthy, review PDBs and cluster capacity, take a fresh etcd backup plus application-specific backups, capture a baseline, pause MachineHealthChecks and freeze unrelated changes.
>
> I initiate the approved update using `oc adm upgrade`. The CVO orchestrates the platform update using dependency-aware release manifests, while the MCO rolls node configuration and RHCOS changes through nodes using cordon, drain, update, reboot and uncordon operations.
>
> During the upgrade I monitor `oc adm upgrade status`, Cluster Operators, MachineConfigPools, nodes, applications and Dynatrace. After completion I validate OpenShift, vSphere integration, networking, ingress, storage, application health and monitoring before re-enabling MachineHealthChecks and closing the change.

---

# Short Memory Flow

```text
VERSION/PATH
   ↓
DEPENDENCIES
   ↓
COMPATIBILITY MATRIX
   ↓
UPGRADE ADD-ONS FIRST IF REQUIRED
   ↓
HEALTH CHECK
   ↓
PDB/CAPACITY
   ↓
BACKUP
   ↓
FREEZE + PAUSE MHC
   ↓
OC ADM UPGRADE
   ↓
CVO
   ↓
MCO
   ↓
ROLLING NODE REBOOTS
   ↓
MONITOR
   ↓
VALIDATE OCP
   ↓
VALIDATE vSPHERE + APPS + DYNATRACE
   ↓
RESUME MHC
   ↓
OBSERVE
   ↓
DONE
```
