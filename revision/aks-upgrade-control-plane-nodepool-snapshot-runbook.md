# AKS Upgrade Runbook — Control Plane, Node Pool, and Snapshot-Based Node Image

## Goal

Upgrade an AKS cluster in a controlled production sequence:

```text
Pre-checks
   ↓
Upgrade Control Plane
   ↓
Validate
   ↓
Upgrade Node Pools using a validated Node Pool Snapshot
   ↓
Kubernetes version + pinned node image move together
   ↓
Validate workloads and cluster health
```

The snapshot-based node-pool step is useful when we want a **known, validated Kubernetes + node-image combination** instead of allowing the node pool to automatically take the latest available compatible node image.

---

## 1. Pre-upgrade checks

Check the current AKS control-plane version:

```bash
az aks show \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --query kubernetesVersion -o tsv
```

Check available upgrades:

```bash
az aks get-upgrades \
  --resource-group <resource-group> \
  --name <cluster-name> \
  -o table
```

Check node pools and versions:

```bash
az aks nodepool list \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  -o table
```

Kubernetes checks:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp
```

Before production upgrade, validate:

- Deprecated Kubernetes APIs
- PodDisruptionBudgets
- Application replica count
- CNI/IP capacity for surge nodes
- Azure compute quota
- CSI/storage compatibility
- Ingress/load-balancer compatibility
- Monitoring/logging agents
- Dynatrace or other DaemonSets
- Workload smoke tests
- Target Kubernetes version support

---

## 2. Upgrade the AKS control plane first

Example:

```bash
az aks upgrade \
  --resource-group <resource-group> \
  --name <cluster-name> \
  --kubernetes-version <target-version> \
  --control-plane-only
```

Example state:

```text
Before:
Control Plane : 1.34.x
Node Pools    : 1.34.x

After control-plane-only upgrade:
Control Plane : 1.35.x
Node Pools    : 1.34.x
```

Validate:

```bash
kubectl get --raw='/readyz?verbose'
kubectl get nodes
kubectl get pods -A
```

The node pools must not be upgraded to a Kubernetes version greater than the control plane.

---

## 3. Prepare a validated node-pool snapshot

A node-pool snapshot contains configuration from its source pool, including:

```text
Kubernetes version
Node image version
OS type
OS SKU
VM size / node-pool configuration information
```

### Important design point

A snapshot does **not** magically create a desired future image.

A practical production pattern is:

```text
Canary / staging node pool
        ↓
Upgrade to target Kubernetes version
        ↓
Receive/test target node image
        ↓
Application + platform validation
        ↓
Create node-pool snapshot
        ↓
Use that snapshot as the production node-pool upgrade source
```

Get the source node-pool resource ID:

```bash
NODEPOOL_ID=$(az aks nodepool show \
  --resource-group <resource-group> \
  --cluster-name <source-cluster> \
  --name <source-nodepool> \
  --query id -o tsv)
```

Create the snapshot:

```bash
az aks nodepool snapshot create \
  --resource-group <resource-group> \
  --name <snapshot-name> \
  --nodepool-id "$NODEPOOL_ID"
```

Inspect it:

```bash
az aks nodepool snapshot show \
  --resource-group <resource-group> \
  --name <snapshot-name> \
  -o json
```

Verify the snapshot's Kubernetes version and node-image version before production use.

---

## 4. Upgrade the production node pool using the snapshot

Get the snapshot resource ID:

```bash
SNAPSHOT_ID=$(az aks nodepool snapshot show \
  --name <snapshot-name> \
  --resource-group <resource-group> \
  --query id -o tsv)
```

Upgrade the node pool:

```bash
az aks nodepool upgrade \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --snapshot-id "$SNAPSHOT_ID"
```

### Why this command is useful

The snapshot already contains the target Kubernetes version and node image.

Conceptually:

```text
Current production node pool
Kubernetes : 1.34.x
Node image : Image-A

Validated snapshot
Kubernetes : 1.35.x
Node image : Image-B
       ↓

az aks nodepool upgrade --snapshot-id ...

       ↓

Production node pool
Kubernetes : 1.35.x
Node image : Image-B
```

This lets the Kubernetes version and the **validated/pinned node image from the snapshot** move together in one node-pool upgrade operation.

Normally, there is no need to add a separate `--kubernetes-version` when the snapshot is being used as the target configuration.

---

## 5. What happens to the nodes during the upgrade?

AKS performs the node-pool upgrade as a rolling operation.

Conceptually:

```text
Create surge capacity
       ↓
Cordon old node
       ↓
Drain workloads
       ↓
Scheduler places workloads elsewhere
       ↓
Replace/reimage node using target configuration
       ↓
Node rejoins cluster
       ↓
Repeat for remaining nodes
       ↓
Remove temporary surge capacity
```

Configure surge before the upgrade if required:

```bash
az aks nodepool update \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --max-surge 33%
```

Make sure subnet IP capacity and regional VM quota can support the temporary surge nodes.

---

## 6. Snapshot rule that must be remembered

AKS permits upgrading an existing node pool to a snapshot configuration only when:

- The snapshot Kubernetes version is newer than the current node-pool Kubernetes version.
- The snapshot node-image version is newer than the current node-pool image.
- The snapshot node image is within the supported snapshot/image age window.

Therefore:

```text
Snapshot = deterministic / pinned target configuration
Snapshot ≠ arbitrary downgrade mechanism
```

Do **not** describe node-pool snapshots as a general way to downgrade to any old N-1 image.

---

## 7. Normal node-image-only upgrade vs snapshot-based upgrade

### Normal node-image-only upgrade

```bash
az aks nodepool upgrade \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --node-image-only
```

Meaning:

```text
Kubernetes version → unchanged
Node image         → latest available compatible node image
```

Use this for routine OS/runtime/security image updates while staying on the same Kubernetes version.

### Snapshot-based node-pool upgrade

```bash
az aks nodepool upgrade \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --snapshot-id "$SNAPSHOT_ID"
```

Meaning:

```text
Kubernetes version → version stored in snapshot
Node image         → exact image stored in snapshot
```

Use this when production must move to a **pre-tested known combination**.

---

## 8. Upgrade multiple node pools

Preferred controlled sequence:

```text
Control Plane
     ↓
System / Canary Pool
     ↓
Validate cluster services
     ↓
User Pool 1
     ↓
Validate applications
     ↓
User Pool 2
     ↓
Validate
     ↓
Continue pool by pool
```

For each pool:

```bash
az aks nodepool upgrade \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --snapshot-id "$SNAPSHOT_ID"
```

Use a snapshot that is compatible with that node pool's OS/VM configuration.

---

## 9. Final validation

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get pdb -A
kubectl get events -A --sort-by=.lastTimestamp
```

Check the node image:

```bash
az aks nodepool show \
  --resource-group <resource-group> \
  --cluster-name <cluster-name> \
  --name <nodepool-name> \
  --query nodeImageVersion -o tsv
```

Validate:

- Nodes are Ready
- All system pods are healthy
- CoreDNS works
- CNI/network connectivity works
- Ingress and load balancers work
- CSI volumes mount correctly
- Monitoring/logging agents are healthy
- DaemonSets are healthy
- Application smoke tests pass
- No abnormal events
- Node pool Kubernetes version matches target
- Node image matches the validated snapshot

---

## 10. Interview-ready explanation

> In production I prefer to separate the AKS control-plane upgrade from the node-pool upgrade. I first upgrade and validate the managed control plane. For node pools, if we want deterministic image control, we validate the target Kubernetes and node-image combination on a canary or staging pool and create a node-pool snapshot from it. The production node pool can then be upgraded with `az aks nodepool upgrade --snapshot-id`. The snapshot supplies both the Kubernetes version and the node-image version, so the pool moves to the tested combination in one rolling operation. AKS uses surge capacity, cordons and drains nodes, replaces or reimages them, and rejoins them to the cluster. A normal `--node-image-only` operation instead moves the pool to the latest compatible image while keeping the Kubernetes version unchanged.

---

## Quick memory flow

```text
PRE-CHECK
   ↓
CONTROL PLANE UPGRADE
   ↓
VALIDATE
   ↓
VALIDATED CANARY/STAGING NODE POOL
   ↓
CREATE NODE-POOL SNAPSHOT
   ↓
SNAPSHOT =
TARGET K8S VERSION + PINNED NODE IMAGE
   ↓
PRODUCTION NODEPOOL UPGRADE
--snapshot-id
   ↓
SURGE → CORDON → DRAIN → REIMAGE/REPLACE → REJOIN
   ↓
VALIDATE
```

## Microsoft references

- AKS node-pool snapshots:
  https://learn.microsoft.com/en-us/azure/aks/node-pool-snapshot
- Upgrade AKS node pools:
  https://learn.microsoft.com/en-us/azure/aks/upgrade-node-pools
- Upgrade AKS node images:
  https://learn.microsoft.com/en-us/azure/aks/upgrade-node-image
- Azure CLI — az aks nodepool:
  https://learn.microsoft.com/en-us/cli/azure/aks/nodepool
