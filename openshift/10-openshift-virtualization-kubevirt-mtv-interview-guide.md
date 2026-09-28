# OpenShift Virtualization, KubeVirt & MTV — Interview Guide

This guide consolidates the key terminology, architecture, scheduling model, storage concepts, and migration basics needed for OpenShift Lead / Architect interviews.

---

# 1. Big Picture

Keep these three technologies separate:

```text
OpenShift Virtualization
        ↓
Red Hat virtualization capability on OpenShift

KubeVirt
        ↓
Underlying open-source technology that lets Kubernetes run VMs

MTV
        ↓
Migration Toolkit for Virtualization
moves existing VMs into OpenShift Virtualization
```

The simplest interview summary is:

> OpenShift Virtualization is Red Hat's supported virtualization layer on OpenShift, KubeVirt is the underlying Kubernetes-native VM technology, and MTV is the migration tool used to move VMs from platforms such as VMware into OpenShift Virtualization.

---

# 2. OpenShift Virtualization

OpenShift Virtualization allows both containers and virtual machines to run on the same OpenShift platform.

```text
OpenShift Cluster

Worker Node
 ├── Pod
 ├── Pod
 └── Virtual Machine

Worker Node
 ├── Pod
 └── Virtual Machine
```

Important:

> A VM is not converted into a container.

The VM still runs using virtualization technologies such as:

```text
KVM
QEMU
libvirt
```

Kubernetes/OpenShift provides the orchestration, scheduling, networking and lifecycle around that VM.

---

# 3. KubeVirt

KubeVirt extends Kubernetes so that it understands virtual-machine resources.

Normally Kubernetes understands resources such as:

```text
Pod
Deployment
StatefulSet
Service
```

KubeVirt introduces resources such as:

```text
VirtualMachine
VirtualMachineInstance
```

Think of KubeVirt as:

> Kubernetes APIs and controllers that allow Kubernetes to create, start, stop, schedule and manage virtual machines.

---

# 4. VM vs VMI

This distinction is important.

## VirtualMachine (VM)

The VM object represents the desired virtual-machine configuration and lifecycle.

Example conceptually:

```text
VirtualMachine: payment-vm
CPU: 4
Memory: 8Gi
Disk: payment-disk
Running: false
```

If the VM is defined but stopped:

```text
VirtualMachine exists
        ↓
No running VMI
        ↓
No virt-launcher pod
```

## VirtualMachineInstance (VMI)

A VMI represents the currently running instance of the VM.

When the VM starts:

```text
VirtualMachine
      ↓
VMI created
      ↓
virt-launcher pod created
      ↓
VM runs
```

Useful analogy:

```text
VirtualMachine
≈ desired VM definition

VirtualMachineInstance
≈ currently running VM instance
```

---

# 5. virt-launcher

This is one of the most important KubeVirt concepts.

Think of `virt-launcher` as:

> The Kubernetes pod wrapper/runtime environment for one running VM.

When a VM starts:

```text
VirtualMachine
      ↓
VirtualMachineInstance
      ↓
virt-launcher Pod
      ↓
QEMU/KVM
      ↓
Guest OS
```

Example on a worker node:

```text
Worker Node

├── nginx pod
├── app pod
└── virt-launcher-payment-vm
       ↓
      QEMU/KVM
       ↓
      Linux/Windows VM
```

The VM itself is not a container. The `virt-launcher` pod gives Kubernetes a schedulable object around the VM runtime.

### If the VM is stopped

```text
VM object exists
but VM is stopped
      ↓
No VMI
      ↓
No virt-launcher
```

### If the VM is started but cannot be scheduled

```text
VM start requested
      ↓
VMI created
      ↓
virt-launcher pod created
      ↓
Pod remains Pending
```

So:

```text
VM stopped
= no virt-launcher

VM started but unschedulable
= virt-launcher may be Pending

VM running
= virt-launcher Running on a worker
```

Interview sentence:

> Each running OpenShift Virtualization VM is associated with a virt-launcher pod, which provides the runtime environment in which QEMU/KVM runs that VM on the selected OpenShift worker node.

---

# 6. Does the VM run on a worker node?

Yes.

In OpenShift Virtualization, a VM is normally a workload running ON an OpenShift worker node.

```text
OpenShift Worker Node
       ↓
virt-launcher pod
       ↓
VM
```

The VM does not normally become an OpenShift worker node itself.

Keep these separate:

```text
OpenShift Virtualization VM
= workload running on a worker

VM used as Kubernetes/OpenShift node
= infrastructure VM separately joined to a cluster
```

The OpenShift scheduler chooses which worker will host the VM based on factors such as:

```text
CPU
Memory
Node labels
Affinity
Taints/tolerations
Device requirements
Topology
Storage/network constraints
```

---

# 7. Can a VM have more resources than a worker?

A running VM still resides on one worker at a time, but virtualization can use overcommitment.

## CPU

A VM can have more virtual CPUs than the number of dedicated physical CPUs allocated to it because vCPUs can be time-sliced.

```text
Worker:
32 physical CPU cores

VM:
64 vCPU
```

This can be possible with CPU overcommit.

But if the VM requires:

```text
64 dedicated physical CPUs
```

the worker must actually have sufficient capacity.

## Memory

Memory can also be overcommitted depending on configuration and workload, but overcommit increases the risk of memory pressure.

If the VM needs guaranteed memory, the worker must provide that memory.

## Storage

VM disk size is different.

A VM can have:

```text
2 TB virtual disk
```

even if the worker has much less local disk, because the VM disk can reside on shared CSI-backed storage.

Key interview sentence:

> A VM is scheduled onto one worker node, but virtual CPU and memory can be overcommitted. If dedicated/guaranteed resources are required, the worker must physically have enough capacity. VM disks can reside on shared persistent storage rather than local worker storage.

---

# 8. Core KubeVirt Components

For interviews, remember these four components:

```text
virt-api
virt-controller
virt-handler
virt-launcher
```

## virt-api

Handles virtualization-related API requests.

Think:

> API entry point for KubeVirt operations.

## virt-controller

Cluster-level controller.

It watches VM/VMI objects and coordinates the lifecycle of VMs.

Think:

> Cluster-level VM orchestration.

## virt-handler

Runs on virtualization-capable nodes, typically as a DaemonSet.

```text
Node1 → virt-handler
Node2 → virt-handler
Node3 → virt-handler
```

It handles node-local VM operations.

Think:

> Node-level VM management.

## virt-launcher

One per running VM.

```text
VMI
 ↓
virt-launcher
 ↓
QEMU/KVM
 ↓
Guest VM
```

Good distinction:

```text
virt-controller
= cluster-level coordination

virt-handler
= node-level coordination

virt-launcher
= per-VM runtime pod
```

---

# 9. End-to-End VM Runtime Flow

Memorize this:

```text
VirtualMachine
      ↓
Start requested
      ↓
VirtualMachineInstance
      ↓
Kubernetes scheduler selects worker
      ↓
virt-launcher pod scheduled
      ↓
virt-handler manages node-local VM lifecycle
      ↓
QEMU/KVM starts guest
      ↓
Guest VM runs
```

---

# 10. VM Storage

VM disks are usually backed by persistent storage.

Typical flow:

```text
StorageClass
    ↓
PVC
    ↓
VM disk
```

The VM disk does not need to live permanently on the local worker.

This is important for:

```text
VM rescheduling
VM migration
node maintenance
high availability
```

---

# 11. CDI — Containerized Data Importer

CDI helps manage VM disk data.

It can handle:

```text
Import VM image
Upload VM image
Clone disk
Populate VM storage
```

Think:

> KubeVirt runs the VM; CDI helps get the VM disk/data into persistent storage.

---

# 12. DataVolume

A DataVolume is a CDI custom resource that helps create/populate storage for a VM.

Typical flow:

```text
DataVolume
     ↓
CDI
     ↓
PVC created/populated
     ↓
VM uses disk
```

Useful mental model:

```text
DataVolume
= request to prepare VM disk data

PVC
= actual persistent storage claim
```

---

# 13. OpenShift Virtualization Storage Considerations

For VM workloads, validate:

```text
StorageClass compatibility
RWO/RWX behavior
Performance / latency
IOPS
Snapshot support
CSI capabilities
Shared storage availability
Capacity
Backup strategy
```

Large VM environments should be designed around storage performance and failure domains, not just CPU/memory.

---

# 14. MTV — Migration Toolkit for Virtualization

MTV is used to migrate VMs from existing virtualization platforms into OpenShift Virtualization.

Typical example:

```text
VMware vSphere
      ↓
     MTV
      ↓
OpenShift Virtualization
```

KubeVirt and MTV have different jobs:

```text
KubeVirt
= runs VMs

MTV
= moves existing VMs into the platform
```

---

# 15. MTV Terminology

Remember these terms:

```text
Provider
Network Mapping
Storage Mapping
Migration Plan
Migration
```

## Provider

Represents the source or target environment.

Example:

```text
Source Provider
= VMware vCenter

Target
= OpenShift Virtualization
```

## Network Mapping

Maps source VM networking to target OpenShift networking.

Example:

```text
VMware VLAN-100
      ↓
OpenShift Multus network: production-network
```

## Storage Mapping

Maps source storage to target StorageClass.

Example:

```text
VMware Datastore01
      ↓
OpenShift StorageClass
```

## Migration Plan

Defines:

```text
Which VMs
Source provider
Target
Network mapping
Storage mapping
Migration settings
```

Think:

> Plan = what/how we intend to migrate.

## Migration

Executes the migration plan.

Think:

```text
Plan
= definition

Migration
= execution
```

---

# 16. VMware to OpenShift Migration Flow

Keep this concise:

```text
VMware vCenter
      ↓
Create source Provider
      ↓
Discover/select VMs
      ↓
Readiness/compatibility checks
      ↓
Network Mapping
      ↓
Storage Mapping
      ↓
Create Migration Plan
      ↓
Choose Cold or Warm migration
      ↓
Execute Migration
      ↓
Copy/convert disks
      ↓
Create OpenShift VM resources
      ↓
Start VM
      ↓
Validate application
```

---

# 17. Cold vs Warm Migration

## Cold migration

```text
Shutdown source VM
      ↓
Copy/convert disks
      ↓
Create target VM
      ↓
Start VM in OpenShift
```

Advantages:

```text
Simpler
Predictable
Lower complexity
```

Disadvantage:

```text
Longer downtime
```

## Warm migration

Most data is copied while the source VM continues running.

```text
VM running
   ↓
Initial copy
   ↓
Changed blocks tracked
   ↓
Incremental copies
   ↓
Cutover
   ↓
Shutdown source VM
   ↓
Final delta copy
   ↓
Start target VM
```

Important:

> Warm migration is not zero-downtime.

There is still a cutover period.

For VMware warm migration, Changed Block Tracking (CBT) is commonly required.

---

# 18. Warm vs Live Migration

Do not mix these terms.

```text
Cold
= source VM stopped for migration

Warm
= source VM runs during most of the copy
  but is stopped at final cutover

Live
= running VM execution moves with minimal interruption
```

For VMware to OpenShift migration, focus on cold/warm migration concepts rather than assuming it is a true live migration.

---

# 19. Migration Readiness Checks

Before migrating, validate:

```text
VM compatibility
Guest OS support
CPU/memory requirements
Disk size and format
Storage capacity
StorageClass mapping
Network/VLAN mapping
IP/DNS dependencies
Firewall rules
Snapshots
CBT for warm migration
Application dependencies
Database dependencies
Maintenance window
Rollback plan
Post-migration validation
```

Architect-level point:

> Migration success is not "the VM booted"; the application and its dependencies must also work after cutover.

---

# 20. Post-Migration Validation

Validate:

```text
VM boots
 ↓
Guest OS healthy
 ↓
Network reachable
 ↓
IP/DNS correct
 ↓
Storage mounted
 ↓
Application services running
 ↓
Load balancer / routes updated
 ↓
Monitoring working
 ↓
Business validation
 ↓
Operational acceptance
```

---

# 21. High-Probability Interview Questions

Be able to answer:

1. What is OpenShift Virtualization?
2. What is KubeVirt?
3. What is the difference between VM and VMI?
4. What is virt-launcher?
5. Does a VM always need to run on an OpenShift worker?
6. What happens if a VM cannot be scheduled?
7. What are virt-controller and virt-handler?
8. How is VM storage handled?
9. What are CDI and DataVolume?
10. Can a VM have more vCPU/memory than a worker physically provides?
11. What is MTV?
12. Explain VMware to OpenShift migration.
13. What is the difference between cold and warm migration?
14. What are network and storage mappings?
15. What would you validate before and after migration?

---

# 22. Final Memory Map

```text
OpenShift Virtualization
        ↓
uses KubeVirt
        ↓
VirtualMachine
        ↓
VMI
        ↓
virt-launcher
        ↓
QEMU/KVM
        ↓
Guest VM


virt-controller
= cluster-level VM lifecycle

virt-handler
= node-level VM management

virt-launcher
= one runtime pod per running VM


Storage
DataVolume
   ↓
CDI
   ↓
PVC
   ↓
VM disk


Migration

VMware
   ↓
MTV Provider
   ↓
Network Mapping
+
Storage Mapping
   ↓
Migration Plan
   ↓
Cold / Warm Migration
   ↓
OpenShift Virtualization VM
   ↓
Application validation
```

---

# 23. Interview Summary Answer

> OpenShift Virtualization allows traditional VMs and containers to run on the same OpenShift platform. It uses KubeVirt, which introduces VM-related Kubernetes resources such as VirtualMachine and VirtualMachineInstance. When a VM starts, a VMI is created and a virt-launcher pod is scheduled onto an OpenShift worker, where QEMU/KVM runs the actual guest VM. virt-controller handles cluster-level VM lifecycle and virt-handler manages node-level VM operations. VM disks are backed by persistent storage, with CDI and DataVolume helping import or populate those disks. For migration, MTV can move workloads such as VMware VMs into OpenShift using providers, network/storage mappings, migration plans and cold or warm migration workflows.
