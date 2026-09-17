# OpenShift Operators & OLM — Interview Guide

This guide is focused on what an OpenShift Lead / Architect should know for interviews and production troubleshooting.

## 1. What is an Operator?

An Operator is essentially a Kubernetes controller plus domain-specific operational knowledge.

A normal Kubernetes Deployment understands things such as:

- desired replica count
- pod template
- rolling deployment behavior

An Operator can additionally understand the lifecycle of a complex application, for example:

- initial deployment and configuration
- scaling
- failover
- backup and restore
- upgrades
- recovery
- application-specific health checks

Examples include database, Kafka, Elasticsearch, storage, observability, virtualization, and service-mesh Operators.

### Interview answer

> An Operator is a Kubernetes controller that watches custom resources and continuously reconciles actual state with desired state. The difference from a generic controller is that it contains application-specific operational knowledge such as installation, configuration, scaling, upgrades, backup and recovery.

---

## 2. Reconciliation Loop

The most important Operator concept is reconciliation.

Example Custom Resource:

```yaml
apiVersion: database.example.com/v1
kind: PostgreSQL
metadata:
  name: prod-db
spec:
  replicas: 3
  storage: 500Gi
```

The Operator watches the resource and continuously tries to make actual state match desired state.

```text
Desired state
    ↓
Custom Resource
    ↓
Operator watches CR
    ↓
Compare desired vs actual
    ↓
Create / update / delete resources
    ↓
Actual state matches desired
```

If somebody deletes a resource managed by the Operator, the Operator normally recreates or reconciles it because the desired state still exists in the Kubernetes API.

If the Operator pod itself restarts, it can normally resume reconciliation because desired state is stored in API objects such as CRs.

---

## 3. CRD vs CR

### CRD — CustomResourceDefinition

A CRD defines a new Kubernetes API/resource type.

Examples:

```text
PostgreSQL
Kafka
VirtualMachine
```

List CRDs:

```bash
oc get crd
```

Example:

```text
postgresqls.database.example.com
```

Think of a CRD as the schema/type definition.

### CR — Custom Resource

A CR is an instance of a CRD.

Example:

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: payment-server
```

Simple analogy:

```text
CRD = Class
CR  = Object
```

---

## 4. What does an Operator manage?

An Operator can watch a CR and manage many underlying resources:

```text
Custom Resource
      ↓
Operator Controller
      ↓
Deployment / StatefulSet
Service
ConfigMap
Secret
PVC
Route
Job
RBAC
etc.
```

Example:

```text
Kafka CR
   ↓
Kafka Operator
   ↓
StatefulSets
Services
PVCs
ConfigMaps
Secrets
```

The administrator declares intent in the CR instead of manually building every resource.

---

## 5. Operator vs Standard Kubernetes Controller

Kubernetes already includes controllers such as:

- Deployment Controller
- ReplicaSet Controller
- Node Controller
- Job Controller

An Operator follows the same controller pattern but adds domain-specific operational logic.

Example:

A Deployment controller understands:

```text
replicas: 3
```

A PostgreSQL Operator may additionally understand:

```text
primary
replicas
replication
backup
failover
database upgrade
```

---

## 6. Two Important Operator Categories in OpenShift

Do not mix these up.

### A. OpenShift Cluster Operators

These manage the OpenShift platform itself.

Examples:

```text
authentication
network
dns
ingress
machine-config
console
storage
etcd
kube-apiserver
```

Check status:

```bash
oc get clusteroperators
```

or:

```bash
oc get co
```

Typical output:

```text
NAME             AVAILABLE   PROGRESSING   DEGRADED
network          True        False         False
authentication   True        False         False
ingress          True        False         False
```

These are primarily part of the OpenShift platform lifecycle and are coordinated by the Cluster Version Operator (CVO).

### B. OLM-managed Add-on Operators

These are usually installed from catalogs/OperatorHub.

Examples can include:

- OpenShift Data Foundation
- AMQ Streams
- Advanced Cluster Management
- OpenShift Virtualization
- Service Mesh
- third-party Operators

These are managed using resources such as:

```text
OperatorHub
CatalogSource
Subscription
InstallPlan
ClusterServiceVersion (CSV)
OperatorGroup
```

---

# OLM — Operator Lifecycle Manager

## 7. What is OperatorHub?

OperatorHub is the discovery/catalog experience in the OpenShift console.

Typical flow:

```text
Operators
   ↓
OperatorHub
   ↓
Search Operator
   ↓
Choose Operator
   ↓
Choose channel / scope
   ↓
Install
```

OperatorHub is not itself the lifecycle engine.

**OLM performs the lifecycle management.**

---

## 8. What is OLM?

OLM stands for **Operator Lifecycle Manager**.

OLM manages the lifecycle of add-on Operators, including:

- installation
- dependencies
- permissions
- versions
- upgrade paths
- scope
- updates

A useful mental model:

```text
OperatorHub = discovery/UI
OLM         = lifecycle engine
```

OLM Classic uses resources such as:

```text
CatalogSource
Subscription
InstallPlan
ClusterServiceVersion (CSV)
OperatorGroup
```

OLM Classic consists primarily of an OLM Operator and Catalog Operator. The OLM Operator handles CSV deployment when requirements are satisfied, while the Catalog Operator monitors catalogs, resolves packages/dependencies and produces InstallPlans.

---

## 9. OLM Architecture / Core Flow

Memorize this:

```text
Operator Catalog
      ↓
CatalogSource
      ↓
Package / Channel
      ↓
Subscription
      ↓
InstallPlan
      ↓
CSV
      ↓
Operator Deployment
      ↓
CRD
      ↓
CR
      ↓
Operator reconciles managed workload
```

A more complete install flow:

```text
1. Operator exists in an Operator catalog
              ↓
2. CatalogSource exposes metadata
              ↓
3. Administrator selects Operator/channel
              ↓
4. Subscription is created
              ↓
5. OLM resolves version and dependencies
              ↓
6. InstallPlan is created
              ↓
7. InstallPlan is approved
   automatically or manually
              ↓
8. CSV is created
              ↓
9. OLM validates requirements
   - CRDs
   - RBAC
   - APIs
   - dependencies
   - OperatorGroup
              ↓
10. Operator Deployment is created
              ↓
11. CSV becomes Succeeded
              ↓
12. Operator watches CRs
```

---

## 10. CatalogSource

A CatalogSource tells OpenShift where Operator package/version metadata comes from.

List catalogs:

```bash
oc get catalogsource -n openshift-marketplace
```

or:

```bash
oc get catsrc -n openshift-marketplace
```

Common catalog names may include:

```text
redhat-operators
certified-operators
community-operators
```

Troubleshoot:

```bash
oc describe catsrc <catalog-name> -n openshift-marketplace
```

Main question:

> Can OLM obtain the required Operator metadata from the configured catalog?

---

## 11. Operator Package

An Operator package contains available versions of an Operator.

Conceptually:

```text
Package
 ├── v1.0
 ├── v1.1
 ├── v1.2
 └── v2.0
```

Example package:

```text
elasticsearch-operator
```

---

## 12. Channel

A channel represents an update stream.

Examples:

```text
stable
fast
candidate
stable-4.17
stable-4.18
```

Conceptually:

```text
stable

v1.0
 ↓
v1.1
 ↓
v1.2
 ↓
v1.3
```

The Subscription follows a selected channel.

The channel defines the supported update stream rather than allowing arbitrary jumps between all visible versions.

---

## 13. Subscription

A Subscription expresses intent:

> Install and track this Operator package from this catalog and channel.

Example:

```yaml
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: my-operator
  namespace: operators
spec:
  channel: stable
  name: my-operator
  source: redhat-operators
  sourceNamespace: openshift-marketplace
  installPlanApproval: Automatic
```

Important fields:

```text
name
source
sourceNamespace
channel
installPlanApproval
```

Commands:

```bash
oc get subscription -A
```

```bash
oc get sub -n <namespace>
```

```bash
oc describe sub <operator> -n <namespace>
```

Important status fields include:

```text
currentCSV
installedCSV
installPlanRef
conditions
```

---

## 14. Automatic vs Manual InstallPlan Approval

### Automatic

```yaml
installPlanApproval: Automatic
```

Flow:

```text
new compatible version
      ↓
InstallPlan created
      ↓
automatically approved
      ↓
upgrade proceeds
```

### Manual

```yaml
installPlanApproval: Manual
```

Flow:

```text
new compatible version
      ↓
InstallPlan created
      ↓
WAIT
      ↓
administrator approval
      ↓
upgrade proceeds
```

For production/regulated environments, manual approval may be preferable because it allows controlled change windows, compatibility checks and rollback preparation.

---

## 15. InstallPlan

The Subscription says **what you want**.

The InstallPlan represents **how OLM plans to install/upgrade it**.

Conceptually:

```text
Subscription = desired Operator/channel
InstallPlan  = calculated installation/upgrade actions
```

An InstallPlan may include resources such as:

- CSVs
- CRDs
- RBAC objects
- ServiceAccounts
- Deployments

Commands:

```bash
oc get installplan -n <namespace>
```

or:

```bash
oc get ip -n <namespace>
```

Detailed view:

```bash
oc describe ip <installplan> -n <namespace>
```

Manual approval example:

```bash
oc patch installplan <name> \
  -n <namespace> \
  --type merge \
  -p '{"spec":{"approved":true}}'
```

---

## 16. CSV — ClusterServiceVersion

CSV stands for **ClusterServiceVersion**.

Do not confuse it with the OpenShift cluster version.

Think of CSV as:

> The metadata and installation definition for a specific Operator version.

Example:

```text
elasticsearch-operator.v5.8.0
```

A CSV can define:

- Operator version
- deployment/install strategy
- required permissions
- RBAC
- CRDs it owns
- CRDs/APIs it requires
- dependencies
- metadata

Commands:

```bash
oc get csv -A
```

```bash
oc get csv -n <namespace>
```

```bash
oc describe csv <csv-name> -n <namespace>
```

Typical CSV phases include:

```text
Pending
InstallReady
Installing
Succeeded
Failed
Replacing
Deleting
```

Normally the target state is:

```text
Succeeded
```

---

## 17. OperatorGroup

OperatorGroup defines the namespace scope in which Operators installed in a namespace are allowed to manage resources.

Commands:

```bash
oc get operatorgroup -A
```

or:

```bash
oc get og -A
```

Detailed view:

```bash
oc describe og <name> -n <namespace>
```

Typical installation modes include:

```text
OwnNamespace
SingleNamespace
MultiNamespace
AllNamespaces
```

### Meaning

**OwnNamespace** — Operator manages the namespace where it runs.

**SingleNamespace** — Operator runs in one namespace and watches one target namespace.

**MultiNamespace** — Operator watches multiple specified namespaces.

**AllNamespaces** — cluster-wide scope.

Whether an Operator supports each mode is declared in its CSV.

This is especially important in multi-tenant cluster design.

---

## 18. Core OLM Components

Useful namespaces:

```bash
oc get pods -n openshift-operator-lifecycle-manager
```

Typical components include:

```text
olm-operator
catalog-operator
packageserver
```

Marketplace components/catalogs are commonly checked under:

```bash
oc get pods -n openshift-marketplace
```

Conceptually:

- **OLM Operator** — handles CSV/operator installation logic when requirements are satisfied
- **Catalog Operator** — works with catalogs, dependency/version resolution and InstallPlans

---

## 19. Operator Upgrade Flow

Example:

```text
Installed version = 1.5
stable channel now offers supported update = 1.6
```

Flow:

```text
CatalogSource exposes 1.6
        ↓
Subscription follows stable
        ↓
OLM detects update
        ↓
Dependency resolution
        ↓
InstallPlan created
        ↓
Approval
        ↓
New CSV created
        ↓
Old CSV becomes Replacing
        ↓
Operator Deployment upgraded
        ↓
New CSV becomes Succeeded
```

Important point:

> OLM follows the Operator's defined update graph/channel. It does not simply assume every visible version can be upgraded to from every other version.

---

## 20. Do Not Manually Treat OLM-managed Deployments as Independent

If OLM owns the Operator Deployment, avoid manually managing its version as if it were an unrelated application.

Instead control lifecycle through supported OLM resources such as:

- Subscription
- channel
- InstallPlan approval
- Operator configuration

This respects the reconciliation/lifecycle model.

---

## 21. Operator Dependencies

An Operator may require:

- a particular CRD
- another Operator/API
- RBAC permissions
- a specific Kubernetes/OpenShift API
- an APIService
- supported platform versions

OLM evaluates requirements and dependency resolution during install/upgrade.

A missing or conflicting requirement may cause CSV/InstallPlan failures.

---

## 22. Operator Bundle and Catalog/Index Content

Modern OLM packaging uses Operator bundles.

A bundle typically contains things such as:

```text
CSV
CRDs
metadata
```

A catalog/index can contain multiple packages and versions.

Conceptually:

```text
Catalog / Index
   │
   ├── Kafka Operator
   │      ├── 1.0
   │      └── 1.1
   │
   └── Database Operator
          ├── 2.0
          └── 2.1
```

For most interviews, knowing this concept is enough unless the interviewer explicitly asks about `opm`, bundle building or custom catalogs.

---

## 23. Disconnected / Air-gapped Clusters

In disconnected environments, the cluster cannot pull Operator catalog content and images directly from public registries.

Typical model:

```text
mirror Operator/catalog content
            +
mirror referenced container images
            ↓
internal registry
            ↓
cluster catalog/mirroring configuration
            ↓
CatalogSource / OLM
            ↓
install Operator
```

Key interview point:

> Both Operator metadata/catalog content and the referenced container images must be available from registries reachable by the disconnected cluster.

---

# Troubleshooting Operators / OLM

## 24. Structured Troubleshooting Chain

Memorize this order:

```text
CATALOGSOURCE
      ↓
SUBSCRIPTION
      ↓
INSTALLPLAN
      ↓
CSV
      ↓
OPERATORGROUP
      ↓
OPERATOR POD
      ↓
CRD
      ↓
CR
      ↓
MANAGED WORKLOAD
```

This prevents random troubleshooting.

---

## 25. Step 1 — Check Overall Cluster / OLM Health

```bash
oc get co
```

Check OLM components:

```bash
oc get pods -n openshift-operator-lifecycle-manager
```

Check marketplace/catalog components:

```bash
oc get pods -n openshift-marketplace
```

---

## 26. Step 2 — Check CatalogSource

```bash
oc get catsrc -n openshift-marketplace
```

```bash
oc describe catsrc <catalog-name> -n openshift-marketplace
```

Questions to answer:

- Is catalog content available?
- Is catalog registry/pod reachable?
- Is the required package/version visible?

---

## 27. Step 3 — Check Subscription

```bash
oc get sub -n <operator-namespace>
```

```bash
oc describe sub <operator> -n <operator-namespace>
```

Inspect:

```text
channel
currentCSV
installedCSV
conditions
installPlanRef
```

This tells you what version OLM wants, what is currently installed, and whether an InstallPlan exists.

---

## 28. Step 4 — Check InstallPlan

```bash
oc get ip -n <namespace>
```

```bash
oc describe ip <installplan> -n <namespace>
```

Very common issue:

```text
Approval = Manual
Approved = false
```

In that case the Operator may simply be waiting for change approval rather than being broken.

---

## 29. Step 5 — Check CSV

```bash
oc get csv -n <namespace>
```

```bash
oc describe csv <csv> -n <namespace>
```

Check:

```text
Phase
Reason
Message
Conditions
Events
```

If the CSV is not `Succeeded`, the reason/conditions usually point to the next layer.

---

## 30. Step 6 — Check OperatorGroup / Scope

```bash
oc get og -n <namespace>
```

```bash
oc describe og <name> -n <namespace>
```

A mismatch between Operator-supported install modes and OperatorGroup scope can block installation.

Example:

```text
Operator supports SingleNamespace only
OperatorGroup targets AllNamespaces
```

This can fail.

---

## 31. Step 7 — Check Operator Deployment / Pods

```bash
oc get pods -n <namespace>
```

```bash
oc describe pod <pod> -n <namespace>
```

```bash
oc logs <operator-pod> -n <namespace>
```

For a multi-container pod:

```bash
oc logs <pod> -c <container> -n <namespace>
```

Common failures:

```text
ImagePullBackOff
CrashLoopBackOff
RBAC denied
certificate errors
webhook failures
API unavailable
configuration errors
```

---

## 32. Step 8 — Check Events

```bash
oc get events -n <namespace> --sort-by=.lastTimestamp
```

Events may show:

```text
FailedMount
FailedScheduling
FailedCreate
ImagePull errors
RBAC errors
webhook failures
quota problems
```

---

## 33. Step 9 — Check CRDs

```bash
oc get crd | grep <operator>
```

```bash
oc describe crd <crd-name>
```

Potential issues:

```text
missing CRD
wrong version
ownership conflict
API version incompatibility
```

---

## 34. Step 10 — Check RBAC / ServiceAccount

If logs contain `forbidden`, inspect permissions.

```bash
oc get sa -n <namespace>
```

```bash
oc get role,rolebinding -n <namespace>
```

```bash
oc get clusterrole,clusterrolebinding
```

Useful validation:

```bash
oc auth can-i list pods \
  --as=system:serviceaccount:<namespace>:<serviceaccount>
```

---

## 35. Step 11 — Check the Managed CR

Do not stop when the Operator pod is Running and the CSV is Succeeded.

Inspect the actual Custom Resource:

```bash
oc describe <custom-resource> <name> -n <namespace>
```

Many Operators expose status like:

```yaml
status:
  conditions:
```

Example:

```text
Ready=False
Reason=StorageFailure
```

---

## 36. Operator Healthy vs Workload Healthy

These are two separate layers.

```text
Layer 1 — Operator lifecycle
CSV
Subscription
Operator Deployment

Layer 2 — Managed application
CR
Pods
PVCs
Services
Routes
application-specific status
```

An Operator can be healthy while the managed workload is unhealthy.

This is a strong interview point.

---

## 37. Scenario — Operator Degraded After Upgrade

A good troubleshooting sequence:

```text
1. Check Subscription
2. Compare installedCSV and currentCSV
3. Check InstallPlan
4. Check CSV phase/conditions
5. Check Operator Deployment/pods
6. Check logs/events
7. Validate OperatorGroup/scope
8. Validate CRDs and dependent APIs
9. Validate RBAC/service account
10. Check release notes and supported upgrade path
11. Check managed CR status
12. Validate the application after Operator recovery
```

This is the type of structured answer expected from a Lead/Architect.

---

# Architect-level Considerations

## 38. Why Operators are Useful in Enterprise Platforms

Operators provide:

- repeatability
- declarative operations
- standardization
- self-healing/reconciliation
- automated lifecycle management
- less dependence on manual runbooks

Instead of many administrators executing a large operational runbook manually, an Operator can encode part of that operational logic into software.

---

## 39. Operator Risks / Governance

Operators are powerful but they also introduce risk.

Potential risks:

```text
CRD/API compatibility
Operator/OpenShift compatibility
automatic upgrades
privileged RBAC
cluster-wide permissions
webhooks
CRD changes
dependency conflicts
vendor quality
unsupported upgrade paths
```

Enterprise governance should consider:

```text
Which Operators are approved
Which catalogs are allowed
Which channels are allowed
Manual vs automatic upgrades
RBAC and SCC impact
Operator scope
Compatibility testing
Change windows
Backup / rollback / recovery procedures
```

---

## 40. Manual vs Automatic Upgrades — Interview Position

Do not say "always manual" or "always automatic".

Better answer:

> It depends on environment and Operator criticality. In development, automatic upgrades may be acceptable. In production or regulated environments, I would usually prefer controlled upgrade channels with manual InstallPlan approval after validating compatibility, release notes, CRD/API changes, application dependencies, backup and rollback options.

---

## 41. Operator Upgrade vs OpenShift Upgrade

Before an OpenShift upgrade, verify critical add-on Operator compatibility with the target OCP release.

Questions to check:

- Does the installed Operator version support the target OCP release?
- Is a newer Operator version required first?
- Are CRD/API versions changing?
- Is there a supported upgrade path?
- Are application dependencies affected?

Do not assume an OpenShift cluster upgrade and third-party Operator upgrade are independent.

---

## 42. OLM v1 Awareness

Modern OpenShift releases also introduce newer OLM v1 concepts, while many enterprise environments still use OLM Classic resources.

For most interviews, prioritize OLM Classic knowledge:

```text
CatalogSource
Subscription
InstallPlan
CSV
OperatorGroup
```

Be aware that newer OLM capabilities exist, but do not overcomplicate the answer unless asked.

---

# Interview Answers to Memorize

## 43. 60-second OLM Answer

> OLM manages the lifecycle of add-on Operators in OpenShift. Operator metadata and versions are exposed through a CatalogSource. When we subscribe to an Operator package and channel, OLM resolves the required version and dependencies and generates an InstallPlan. Once that plan is approved, OLM creates the appropriate CSV and required CRDs, RBAC and Operator deployment. The CSV represents that specific Operator version and its installation requirements. OperatorGroup controls the namespaces the Operator can watch. For upgrades, the Subscription follows the selected channel and OLM generates a new InstallPlan based on the supported upgrade path. If an Operator fails, I normally troubleshoot from CatalogSource → Subscription → InstallPlan → CSV → Operator pod/logs → managed CR.

---

## 44. 30-second Operator Answer

> An Operator is a Kubernetes controller with application-specific operational knowledge. It watches custom resources and continuously reconciles actual state to desired state. That lets it automate tasks such as deployment, configuration, scaling, upgrades, recovery or backup depending on the Operator.

---

## 45. 30-second CRD vs CR Answer

> A CRD defines a new Kubernetes API type, while a CR is an instance of that type. For example, KubeVirt can define the VirtualMachine CRD, and a specific `payment-vm` object is a VirtualMachine CR. The Operator watches those CRs and reconciles the underlying infrastructure or workloads.

---

# Must-know Commands

```bash
# Cluster Operators
oc get co

# OLM components
oc get pods -n openshift-operator-lifecycle-manager
oc get pods -n openshift-marketplace

# Catalogs
oc get catsrc -n openshift-marketplace
oc describe catsrc <name> -n openshift-marketplace

# Subscriptions
oc get sub -A
oc describe sub <name> -n <namespace>

# InstallPlans
oc get ip -A
oc describe ip <name> -n <namespace>

# CSVs
oc get csv -A
oc describe csv <name> -n <namespace>

# OperatorGroups
oc get og -A
oc describe og <name> -n <namespace>

# CRDs
oc get crd
oc describe crd <name>

# Operator/workload pods
oc get pods -n <namespace>
oc describe pod <pod> -n <namespace>
oc logs <pod> -n <namespace>

# Events
oc get events -n <namespace> --sort-by=.lastTimestamp

# RBAC verification
oc auth can-i list pods --as=system:serviceaccount:<namespace>:<serviceaccount>
```

---

# High-probability Interview Questions

You should be able to answer all of these confidently:

1. What is an Operator and how is it different from a normal Kubernetes controller?
2. Explain CRD vs CR and the reconciliation loop.
3. Explain OperatorHub → CatalogSource → Subscription → InstallPlan → CSV.
4. What exactly is a CSV?
5. What is OperatorGroup and why is it required?
6. How does an Operator upgrade happen through OLM?
7. An Operator is stuck/degraded after an upgrade — how do you troubleshoot it?
8. How would you control Operator upgrades in a regulated production environment?
9. What happens if someone deletes a resource managed by an Operator?
10. Can an Operator be healthy while the application is unhealthy?
11. How do Operators work in disconnected clusters?
12. What should you validate before an OpenShift upgrade when third-party Operators are installed?

---

# Final Memory Map

```text
OperatorHub
   ↓
CatalogSource
   ↓
Package / Channel
   ↓
Subscription
   ↓
InstallPlan
   ↓
CSV
   ↓
OperatorGroup / Scope
   ↓
Operator Deployment
   ↓
CRD
   ↓
CR
   ↓
Reconciliation
   ↓
Managed Workload
```

If troubleshooting, walk the chain from top to bottom instead of jumping randomly into pod logs.

---

## References

- Red Hat OpenShift Operators documentation: https://docs.redhat.com/en/documentation/openshift_container_platform/
- Operator Lifecycle Manager documentation: https://olm.operatorframework.io/
