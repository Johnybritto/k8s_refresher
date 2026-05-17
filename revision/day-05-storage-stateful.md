# Day 5 – Storage & Stateful Systems

Status: Covered in chat. These notes capture Day 5 learning and the PVC Pending exercise review.

---

## 1. Core Idea

Pods are temporary. They can be deleted, recreated, evicted, or rescheduled. If an application writes important data only inside the container filesystem, that data can be lost when the Pod is recreated.

Kubernetes storage separates application storage requests from actual storage implementation.

```text
Pod -> PVC -> PV -> Actual storage backend
```

Senior interview point:

```text
Stateless workloads can be freely replaced. Stateful workloads need stable storage, stable identity, backup, restore, and failure testing.
```

---

## 2. Stateless vs Stateful Workload

### Stateless workload

Examples:

- Frontend
- REST API
- Auth service
- Worker service
- Payment API

A stateless workload stores state outside the Pod, such as in a database, queue, cache, or object store.

### Stateful workload

Examples:

- PostgreSQL
- MySQL
- Kafka
- Zookeeper
- Elasticsearch
- Redis with persistence

A stateful workload usually needs:

- Stable storage
- Stable network identity
- Ordered startup/shutdown
- Backup and restore
- Data consistency
- Disaster recovery planning

---

## 3. Volumes and emptyDir

A volume is storage mounted into a Pod.

`emptyDir` is created when a Pod starts and deleted when the Pod is removed.

Important:

```text
emptyDir survives container restart inside the same Pod, but it does not survive Pod deletion or rescheduling.
```

Good for:

- Temporary cache
- Scratch space
- Temporary processing
- File sharing between containers in the same Pod

Not good for:

- Database data
- Important uploads
- Persistent application state

---

## 4. PersistentVolume – PV

A PersistentVolume is the actual storage resource represented in Kubernetes.

It may represent:

- AWS EBS
- Azure Disk
- GCP Persistent Disk
- NFS
- vSphere volume
- Ceph
- SAN-backed storage

Interview line:

```text
PV is the actual storage resource available to the cluster.
```

---

## 5. PersistentVolumeClaim – PVC

A PersistentVolumeClaim is a request for storage by an application.

The application says:

```text
I need 100Gi storage with this access mode and StorageClass.
```

Kubernetes then binds the PVC to a suitable PV or dynamically provisions one through a StorageClass.

---

## 6. PV vs PVC

```text
PV  = actual storage resource
PVC = user's request for storage
Pod = workload using the PVC
```

Strong answer:

```text
A PersistentVolume is the actual storage resource available in the cluster. A PersistentVolumeClaim is a request for storage by a workload. The Pod references the PVC, and the PVC binds to a suitable PV either statically or dynamically through a StorageClass.
```

---

## 7. StorageClass

StorageClass defines how storage is dynamically provisioned.

It includes:

- Provisioner / CSI driver
- Disk type or storage tier
- Reclaim policy
- Volume binding mode
- Provider-specific parameters

Example concepts:

```text
fast-ssd -> AWS EBS gp3 / premium disk
standard -> slower disk
nfs-shared -> shared filesystem
```

Strong answer:

```text
StorageClass defines how Kubernetes should dynamically provision storage, including the CSI provisioner, disk type, reclaim policy, volume binding mode, and provider-specific parameters.
```

---

## 8. Dynamic Provisioning

Without dynamic provisioning:

```text
Admin creates PV manually.
Developer creates PVC.
PVC binds to existing PV.
```

With dynamic provisioning:

```text
Developer creates PVC.
StorageClass calls CSI driver.
Storage backend creates disk.
PV is created automatically.
PVC binds to PV.
Pod mounts the PVC.
```

---

## 9. CSI Driver

CSI means Container Storage Interface.

Kubernetes uses CSI drivers to integrate with external storage systems.

CSI operations include:

- Create volume
- Delete volume
- Attach volume to node
- Detach volume from node
- Mount volume into Pod
- Expand volume
- Snapshot volume

Examples:

- AWS EBS CSI Driver
- Azure Disk CSI Driver
- Azure Files CSI Driver
- vSphere CSI Driver
- Ceph CSI Driver
- NFS CSI Driver

Interview line:

```text
CSI is the plugin interface that lets Kubernetes talk to external storage providers.
```

---

## 10. Access Modes

Common access modes:

- ReadWriteOnce – RWO
- ReadOnlyMany – ROX
- ReadWriteMany – RWX
- ReadWriteOncePod – RWOP

### ReadWriteOnce

Volume can be mounted read-write by a single node.

Common for:

- AWS EBS
- Azure Disk
- GCP Persistent Disk

Important:

```text
RWO means single node, not always strictly single Pod.
```

### ReadWriteMany

Volume can be mounted read-write by multiple nodes.

Common for:

- NFS
- EFS
- Azure Files
- CephFS

Useful for shared filesystem workloads.

---

## 11. Reclaim Policy

PersistentVolume reclaim policies are:

```text
Retain
Delete
Recycle
```

In modern Kubernetes, focus mainly on **Retain** and **Delete**. **Recycle is deprecated** and rarely used.

### Delete

When PVC is deleted, the PV and underlying storage may also be deleted.

Good for:

- Dev/test
- Temporary environments
- Non-critical dynamic storage

Risk:

```text
Accidental PVC deletion can delete the underlying disk.
```

### Retain

When PVC is deleted, the PV and underlying disk remain for manual recovery.

Good for:

- Production databases
- Critical data
- Manual recovery situations

### Recycle

Old/deprecated behavior where Kubernetes tried to scrub the volume and make it available again. Usually avoid in modern Kubernetes.

Strong answer:

```text
Kubernetes reclaim policies are Retain, Delete, and Recycle. Delete removes the PV and usually the backing storage when the PVC is deleted. Retain keeps the backing storage for manual recovery. Recycle is deprecated and rarely used now. For production data, Retain is usually safer unless backups and deletion controls are mature.
```

---

## 12. volumeBindingMode

Two common modes:

```text
Immediate
WaitForFirstConsumer
```

### Immediate

PVC binds or provisions storage immediately when the PVC is created.

Risk:

```text
Storage may be created in zone-a, but the Pod may be scheduled to zone-b. Zonal disks usually cannot attach across zones.
```

### WaitForFirstConsumer

Storage is provisioned only after Kubernetes knows where the consuming Pod will run.

This avoids zone mismatch for zonal disks such as EBS or Azure Disk.

Strong answer:

```text
WaitForFirstConsumer delays provisioning until the scheduler knows the Pod placement. This prevents zone mismatch between the volume and the node.
```

---

## 13. StatefulSet Storage

StatefulSet uses `volumeClaimTemplates` to create a separate PVC per Pod.

Example:

```text
mysql-0 -> data-mysql-0 -> PV/Disk 0
mysql-1 -> data-mysql-1 -> PV/Disk 1
mysql-2 -> data-mysql-2 -> PV/Disk 2
```

If `mysql-0` is recreated, it reuses `data-mysql-0`.

Interview line:

```text
StatefulSet gives stable Pod identity and stable PVC. Headless Service gives stable DNS identity.
```

---

## 14. Headless Service for StatefulSet

A headless Service has:

```yaml
clusterIP: None
```

It gives stable DNS names for StatefulSet Pods.

Example:

```text
mysql-0.mysql.default.svc.cluster.local
mysql-1.mysql.default.svc.cluster.local
mysql-2.mysql.default.svc.cluster.local
```

This is useful for clustered systems where members need stable identity.

---

## 15. Scenario: Pod Restart Causes Data Loss

If data disappears after Pod restart/recreation, check where the app wrote the data.

Ask:

- Is data written to the container filesystem?
- Is the path backed by emptyDir?
- Is the path backed by PVC?
- Was the Pod deleted or rescheduled?
- Is the PVC still Bound?
- What is the reclaim policy?

Strong answer:

```text
If data is lost after Pod restart or recreation, I check whether the application writes to the container filesystem, emptyDir, or a PVC-backed mount. Container filesystem and emptyDir are not durable across Pod deletion. For persistent data, I would mount a PVC at the application data path and verify reclaim policy and backup strategy.
```

---

## 16. Scenario: PVC Stuck Pending

PVC Pending means the claim exists but has not bound to a PV.

Common causes:

- No default StorageClass
- Wrong StorageClass name
- StorageClass does not exist
- CSI provisioner missing or unhealthy
- External storage backend issue
- Unsupported access mode
- Quota exceeded
- Cloud provider permission issue
- Zone/region mismatch
- volumeBindingMode is WaitForFirstConsumer and no Pod exists yet
- Pod scheduling issue when using WaitForFirstConsumer

Useful commands:

```bash
kubectl get pvc -n <namespace>
kubectl describe pvc <pvc> -n <namespace>
kubectl get storageclass
kubectl describe storageclass <storageclass>
kubectl get csidriver
kubectl get pods -A | grep -i csi
kubectl get events -n <namespace> --sort-by=.lastTimestamp
kubectl get resourcequota -n <namespace>
kubectl describe pod <pod> -n <namespace>
```

Strong answer:

```text
For a PVC stuck Pending, I first describe the PVC and read the events. Then I check whether the StorageClass exists and whether it is the default if no storageClassName was specified. I verify the CSI provisioner, access mode, volumeBindingMode, reclaim policy, and provider-specific parameters. Then I check CSI driver health, external storage backend health, namespace quota, cloud permissions, and zone constraints. If the StorageClass uses WaitForFirstConsumer, I also check whether a consuming Pod exists and whether the Pod can be scheduled.
```

---

## 17. Running Databases on Kubernetes

Running databases on Kubernetes is possible, but it requires operational maturity.

Challenges:

- Storage latency and IOPS
- Volume attach/detach delays
- Backup and restore
- Split-brain risk
- Data corruption risk
- Network latency between replicas
- Pod rescheduling behavior
- Upgrade complexity
- Encryption and security
- Disaster recovery

When it may be acceptable:

- Mature CSI storage
- Tested backup/restore
- Clear RPO/RTO
- Stable operator
- Anti-affinity and PDBs
- Monitoring and alerting
- Failure testing

Strong answer:

```text
Running databases on Kubernetes is possible, but I would not treat them like stateless workloads. I would evaluate storage performance, CSI maturity, backup/restore, failover behavior, anti-affinity, PodDisruptionBudgets, monitoring, encryption, and operator maturity. For many critical workloads, managed databases may still be preferred unless the platform team has strong operational maturity.
```

---

## 18. Day 5 Exercise Review

### Exercise

A PVC is stuck in Pending state and the application Pod cannot start. How will you troubleshoot it step by step?

### User Answer

```text
describe the pvc 
check the csi provisioner 
check heath of the external storage 
check access modes 
check quota
```

### Review

This is a good answer. You covered several important checks:

- Describe PVC
- CSI provisioner
- External storage health
- Access modes
- Quota

To make it senior-level, add:

- StorageClass existence/default status
- PVC events
- volumeBindingMode
- WaitForFirstConsumer behavior
- Pod scheduling
- Zone constraints
- Cloud permissions
- CSI controller/node pod health
- Reclaim policy and parameters

### Strong Interview Version

```text
For a PVC stuck in Pending, I would first describe the PVC and read the Events section because it usually gives the exact reason. Then I would check whether the correct StorageClass exists and whether a default StorageClass is configured if the PVC does not specify one.

Next, I would verify the CSI provisioner and CSI driver pods, including controller and node components. I would check whether the external storage backend is healthy and whether the cluster has permission to create volumes.

Then I would validate access mode, requested size, quota, StorageClass parameters, reclaim policy, and volumeBindingMode. If the StorageClass uses WaitForFirstConsumer, I would check whether a consuming Pod exists and whether the Pod can be scheduled to a suitable node/zone. I would also check for zone mismatch, cloud provider events, and namespace resource quota.
```

---

## 19. Day 5 Final Summary

```text
Pod uses PVC.
PVC binds to PV.
PV represents actual storage.
StorageClass dynamically creates PV.
CSI driver talks to the storage backend.
```

```text
Reclaim policy: Delete removes storage, Retain keeps it, Recycle is deprecated.
```

```text
PVC Pending troubleshooting: PVC events -> StorageClass -> CSI driver -> external storage -> access mode -> quota -> binding mode -> zone/scheduling -> cloud permissions.
```
