# OpenShift on VMware vSphere — IPI Cluster Build Procedure

## Purpose

This document is an interview/refresher guide for building an OpenShift cluster on **VMware vSphere using IPI (Installer-Provisioned Infrastructure)**.

The focus is not only on the steps, but also on **what happens internally at each stage**.

---

# 1. End-to-End Flow

```text
Prepare vSphere + DNS + VIPs
        ↓
Download openshift-install + oc
        ↓
Prepare pull secret + SSH key
        ↓
Create install-config.yaml
        ↓
openshift-install create cluster
        ↓
Installer connects to vCenter
        ↓
Creates Bootstrap VM + Control Plane VMs
        ↓
Bootstrap starts temporary control plane
        ↓
3 Masters form permanent etcd/control plane
        ↓
Bootstrap VM removed
        ↓
Machine API creates Worker VMs
        ↓
Workers join the cluster
        ↓
Cluster Operators initialize
        ↓
Networking + Ingress + CSI become ready
        ↓
Validate cluster
        ↓
Cluster Ready
```

---

# 2. Prepare the vSphere Infrastructure

Before OpenShift installation starts, the VMware platform must already exist.

Typical hierarchy:

```text
vCenter
   |
Datacenter
   |
vSphere Cluster
   |
ESXi Hosts
   |
Resource Pool
   |
Datastore
   |
Port Group / Network
```

Example:

```text
vCenter:       vcsa.company.com
Datacenter:    DC01
Cluster:       Compute-Cluster-01
Datastore:     vsanDatastore
Network:       OCP-Prod-Network
```

## What we do

Prepare or identify:

- vCenter
- Datacenter
- vSphere compute cluster
- ESXi hosts
- Datastore
- Port Group / VLAN
- Resource pool/folder if required
- vCenter service account with required permissions

## What happens later

`openshift-install` uses the vCenter API to create and manage OpenShift virtual machines.

```text
openshift-install
        |
      HTTPS 443
        |
        v
     vCenter
        |
        v
     ESXi Hosts
```

---

# 3. Plan the Network and VIPs

Example cluster subnet:

```text
10.20.30.0/24
```

Plan IP capacity for:

- Bootstrap VM
- 3 Control Plane VMs
- Worker VMs
- API VIP
- Ingress VIP

Example:

```text
API VIP:      10.20.30.10
Ingress VIP:  10.20.30.11
```

## API VIP

Used for the OpenShift/Kubernetes API.

```text
api.ocp-prod.company.com
          ↓
      API VIP
          ↓
 Control Plane API Servers
          ↓
        6443
```

## Ingress VIP

Used for application and console routes.

```text
*.apps.ocp-prod.company.com
            ↓
       Ingress VIP
            ↓
     OpenShift Router
            ↓
         Service
            ↓
           Pod
```

---

# 4. Configure DNS

Assume:

```text
Base domain:  company.com
Cluster name: ocp-prod
```

Cluster domain:

```text
ocp-prod.company.com
```

Important DNS records:

```text
api.ocp-prod.company.com
        ↓
API VIP

api-int.ocp-prod.company.com
        ↓
API VIP

*.apps.ocp-prod.company.com
        ↓
Ingress VIP
```

## Why they matter

### `api.<cluster>.<domain>`
External/admin API access.

### `api-int.<cluster>.<domain>`
Internal cluster API access.

### `*.apps.<cluster>.<domain>`
Wildcard DNS for OpenShift Routes and console traffic.

---

# 5. Prepare vCenter Credentials

Example:

```text
vCenter:  vcsa.company.com
Username: openshift-installer@vsphere.local
Password: ********
```

## What happens

The installer authenticates to vCenter and uses the VMware API to:

- create VMs
- configure disks
- attach networks
- power VMs on/off
- remove bootstrap resources
- allow Machine API to create workers later

---

# 6. Download OpenShift Tools

Required:

```text
openshift-install
oc
```

Difference:

```text
openshift-install
      =
create / destroy OpenShift cluster
```

```text
oc
      =
administer the running OpenShift cluster
```

---

# 7. Get the Red Hat Pull Secret

The pull secret allows the installer and cluster to pull OpenShift release images and platform images.

Conceptually:

```text
OpenShift installer / cluster
            ↓
       Pull Secret
            ↓
registry.redhat.io / quay.io
            ↓
OpenShift release images
```

---

# 8. Create an SSH Key

Example:

```bash
ssh-keygen -t ed25519
```

The public key is added to the installation configuration.

SSH is mainly useful for:

- installation troubleshooting
- node debugging
- recovery situations

Normal cluster administration should be performed with `oc`.

---

# 9. Create the Installation Directory

```bash
mkdir ocp-prod
```

Generate the configuration:

```bash
openshift-install create install-config --dir=ocp-prod
```

The installer asks for information such as:

```text
SSH public key
        ↓
Platform: VMware vSphere
        ↓
vCenter
        ↓
vCenter username/password
        ↓
Datacenter
        ↓
Datastore
        ↓
vSphere Cluster
        ↓
Network / Port Group
        ↓
API VIP
        ↓
Ingress VIP
        ↓
Base Domain
        ↓
Cluster Name
        ↓
Pull Secret
```

The result is:

```text
ocp-prod/
└── install-config.yaml
```

---

# 10. Understand `install-config.yaml`

Simplified example:

```yaml
apiVersion: v1

baseDomain: company.com

metadata:
  name: ocp-prod

controlPlane:
  name: master
  replicas: 3

compute:
- name: worker
  replicas: 3

platform:
  vsphere:
    vcenters:
    - server: vcsa.company.com
      user: openshift-installer@vsphere.local
      password: xxxxx

    failureDomains:
    - name: primary
      region: region1
      zone: zone1
      server: vcsa.company.com
      topology:
        datacenter: DC01
        computeCluster: /DC01/host/Compute-Cluster-01
        datastore: /DC01/datastore/vsanDatastore
        networks:
        - OCP-Prod-Network

    apiVIPs:
    - 10.20.30.10

    ingressVIPs:
    - 10.20.30.11

networking:
  networkType: OVNKubernetes

pullSecret: '...'
sshKey: 'ssh-ed25519 ...'
```

## What this file means

It describes the desired cluster:

```text
Which vCenter?
Which Datacenter?
Which vSphere Cluster?
Which Datastore?
Which Network?
Which API VIP?
Which Ingress VIP?
How many Masters?
How many Workers?
Which domain?
```

Back it up before installation because the installer consumes installation assets from its working directory.

Example:

```bash
cp ocp-prod/install-config.yaml install-config-backup.yaml
```

---

# 11. Start the Cluster Build

Run:

```bash
openshift-install create cluster \
  --dir=ocp-prod \
  --log-level=info
```

This is where IPI begins provisioning infrastructure.

## What happens

```text
Installation Host
      |
openshift-install
      |
      | HTTPS 443
      v
    vCenter
      |
      +--> Create Bootstrap VM
      +--> Create Control Plane VMs
      +--> Configure VM networking
      +--> Configure disks/datastore
      +--> Apply OpenShift installation assets
```

---

# 12. Installer Generates OpenShift Assets

Internally the installer generates artifacts such as:

- manifests
- Ignition configuration
- certificates
- kubeconfig
- machine configuration
- cluster/operator configuration

Conceptually:

```text
install-config.yaml
        ↓
openshift-install
        ↓
Manifests
Ignition configs
Certificates
Kubeconfig
Machine definitions
```

With normal IPI you do not manually build every one of these assets.

---

# 13. Bootstrap VM Is Created

vCenter temporarily contains something similar to:

```text
ocp-prod-bootstrap
ocp-prod-master-0
ocp-prod-master-1
ocp-prod-master-2
```

The bootstrap VM runs **RHCOS — Red Hat CoreOS**.

## Why bootstrap exists

At the beginning there is no permanent Kubernetes/OpenShift control plane yet.

This creates a chicken-and-egg problem:

```text
Who configures Kubernetes
if Kubernetes is not running yet?
```

Answer:

```text
Bootstrap VM
```

The bootstrap node temporarily provides enough control-plane functionality to bring up the permanent masters.

---

# 14. Bootstrap VM Boots Using Ignition

```text
Bootstrap VM starts
        ↓
RHCOS boots
        ↓
Ignition runs
        ↓
Bootstrap configuration applied
        ↓
Temporary etcd starts
        ↓
Temporary Kubernetes control plane starts
```

Ignition configures the machine during first boot.

It can provide/configure things such as:

- files
- systemd units
- certificates
- users
- machine configuration

---

# 15. Three Control Plane VMs Start

The installer creates:

```text
Master-0
Master-1
Master-2
```

Each one is:

```text
VMware VM
   +
RHCOS
```

They also consume their first-boot configuration.

During bootstrap:

```text
                     vCenter
                        |
       +----------------+----------------+
       |                |                |
   Master-0         Master-1         Master-2
       \                |                /
        \               |               /
             Bootstrap VM
```

---

# 16. Bootstrap Brings Up the Permanent Control Plane

Bootstrap temporarily runs enough Kubernetes/OpenShift services to configure the permanent masters.

Conceptually:

```text
BOOTSTRAP

Temporary:
- etcd
- Kubernetes API/control plane
- controllers

          ↓

MASTER-0
MASTER-1
MASTER-2

Permanent:
- kube-apiserver
- etcd
- kube-controller-manager
- kube-scheduler
- OpenShift APIs
- Cluster Operators
```

---

# 17. Permanent etcd Cluster Forms

Initially:

```text
Bootstrap
   ↓
Temporary etcd
```

Then the three control-plane nodes establish the production etcd cluster:

```text
Master-0 ─┐
Master-1 ─┼── Production etcd cluster
Master-2 ─┘
```

etcd becomes the permanent database for Kubernetes/OpenShift cluster state.

---

# 18. Permanent Control Plane Becomes Operational

The control-plane VMs host services including:

```text
kube-apiserver
kube-controller-manager
kube-scheduler
etcd
OpenShift API
Cluster Version Operator
Cluster Operators
```

At this point the permanent control plane can manage the cluster independently.

---

# 19. Cluster Version Operator and Operators Start

The **Cluster Version Operator (CVO)** manages the OpenShift release payload and ensures required OpenShift components/operators are installed at the desired version.

Conceptually:

```text
OpenShift Release Payload
          ↓
         CVO
          ↓
--------------------------------
|      |        |       |      |
etcd  API    network  ingress  etc.
```

OpenShift relies heavily on Operators to continuously reconcile platform components.

---

# 20. Bootstrap Completes

You can monitor bootstrap completion with:

```bash
openshift-install wait-for bootstrap-complete --dir=ocp-prod
```

Bootstrap complete means:

> The permanent control plane is now capable of operating without the bootstrap VM.

---

# 21. Bootstrap VM Is Deleted

Before:

```text
Bootstrap
Master-0
Master-1
Master-2
```

After:

```text
Master-0
Master-1
Master-2
```

In IPI the installer removes the temporary bootstrap VM automatically.

## Interview point

**Where is the bootstrap node after installation?**

It no longer exists. It is only required to bootstrap the permanent control plane.

---

# 22. Machine API Creates Worker VMs

OpenShift uses resources such as:

```text
MachineSet
Machine
Machine API Operator
```

Flow:

```text
MachineSet
    ↓
Machine objects
    ↓
Machine API Operator
    ↓
vCenter API
    ↓
Create VMware Worker VM
```

Example:

```text
MachineSet
worker-zone1
      ↓
Machine
worker-abc123
      ↓
vCenter
      ↓
VM
ocp-prod-worker-abc123
```

This is an important IPI capability: **OpenShift can manage the lifecycle of worker infrastructure through the Machine API.**

---

# 23. Worker VM Boot and Join Process

```text
Machine object
      ↓
Machine API
      ↓
vCenter
      ↓
Create VM
      ↓
RHCOS boots
      ↓
Machine configuration applied
      ↓
kubelet starts
      ↓
Connects to OpenShift API
      ↓
Registers as Node
```

Eventually:

```bash
oc get nodes
```

Example:

```text
master-0   Ready   control-plane
master-1   Ready   control-plane
master-2   Ready   control-plane
worker-0   Ready   worker
worker-1   Ready   worker
worker-2   Ready   worker
```

---

# 24. API Traffic Flow

Administrative/client traffic:

```text
Administrator
      |
     oc
      |
      v
api.ocp-prod.company.com
      |
    API VIP
      |
      v
Control Plane API Servers
      |
     6443
```

Port to remember:

```text
6443 = Kubernetes/OpenShift API
```

---

# 25. Ingress Traffic Flow

Application traffic:

```text
User
 |
 v
myapp.apps.ocp-prod.company.com
 |
DNS
 |
Ingress VIP
 |
OpenShift Router
 |
Service
 |
Application Pod
```

The wildcard record:

```text
*.apps.ocp-prod.company.com
```

allows OpenShift Routes such as:

```text
console-openshift-console.apps.ocp-prod.company.com
myapp.apps.ocp-prod.company.com
```

---

# 26. OVN-Kubernetes Networking Initializes

OpenShift normally uses **OVN-Kubernetes** networking.

It provides cluster networking functionality such as:

- Pod-to-Pod networking
- Service networking
- NetworkPolicy
- Node-to-node overlay networking
- cluster east-west traffic

---

# 27. vSphere CSI Storage Initializes

OpenShift integrates with VMware storage using the vSphere CSI driver.

Storage path:

```text
PVC
 ↓
StorageClass
 ↓
vSphere CSI
 ↓
vCenter
 ↓
Datastore
 ↓
Virtual Disk
 ↓
Attached to Worker VM
 ↓
Mounted into Pod
```

This allows Kubernetes/OpenShift workloads to dynamically request persistent storage from vSphere-backed datastores.

---

# 28. Cluster Operators Become Healthy

Check:

```bash
oc get clusteroperators
```

or:

```bash
oc get co
```

Healthy state should generally be:

```text
AVAILABLE   PROGRESSING   DEGRADED
True        False         False
```

Important operators include:

```text
authentication
console
dns
etcd
image-registry
ingress
kube-apiserver
machine-api
machine-config
monitoring
network
storage
operator-lifecycle-manager
```

---

# 29. Validate the Cluster

## Nodes

```bash
oc get nodes
```

## Cluster Operators

```bash
oc get co
```

## Cluster Version

```bash
oc get clusterversion
```

## Machine objects

```bash
oc get machines -n openshift-machine-api
```

## MachineSets

```bash
oc get machinesets -n openshift-machine-api
```

## Pods

```bash
oc get pods -A
```

## Storage classes

```bash
oc get storageclass
```

## Ingress Controller

```bash
oc get ingresscontroller -n openshift-ingress-operator
```

---

# 30. Final vSphere IPI Architecture

```text
                       VMware vCenter
                              |
             +----------------+----------------+
             |                |                |
         ESXi Host        ESXi Host        ESXi Host
             |                |                |
             +----------------+----------------+
                              |
                       vSphere Cluster
                              |
                        Port Group/VLAN
                              |
       +----------------------+----------------------+
       |                      |                      |
   Master-0               Master-1               Master-2
       |                      |                      |
       +----------- Production etcd ----------------+
                              |
                       OpenShift API
                              |
                           API VIP
                              |
                 api.ocp-prod.company.com


                 Worker-0   Worker-1   Worker-2
                    |          |          |
                    +----------+----------+
                               |
                       OpenShift Router
                               |
                         Ingress VIP
                               |
                  *.apps.ocp-prod.company.com
```

During installation only:

```text
Bootstrap VM
     |
Temporary Kubernetes/OpenShift control plane
Temporary etcd
     |
     v
Builds / enables permanent control plane
```

After bootstrap completion:

```text
Bootstrap VM → DELETED
```

---

# 31. What We Prepare vs What IPI Creates

| Component | We prepare | IPI/OpenShift creates |
|---|---:|---:|
| vCenter | ✅ | |
| ESXi cluster | ✅ | |
| Datacenter | ✅ | |
| Datastore | ✅ | |
| VLAN / Port Group | ✅ | |
| DNS | ✅ | |
| API / Ingress IP planning | ✅ | |
| vCenter credentials | ✅ | |
| Pull secret | ✅ | |
| SSH key | ✅ | |
| `install-config.yaml` | ✅ | |
| Bootstrap VM | | ✅ |
| Control Plane VMs | | ✅ |
| Worker VMs | | ✅ |
| RHCOS initial configuration | | ✅ |
| etcd | | ✅ |
| Kubernetes/OpenShift control plane | | ✅ |
| Machine API | | ✅ |
| Cluster Operators | | ✅ |
| Bootstrap deletion | | ✅ |

---

# 32. Sequence to Memorize for Interviews

```text
1. Prepare vCenter / Datacenter / Cluster / Datastore / Network
                         ↓
2. Configure DNS + API VIP + Ingress VIP
                         ↓
3. Prepare vCenter credentials + pull secret + SSH key
                         ↓
4. Create install-config.yaml
                         ↓
5. Run openshift-install create cluster
                         ↓
6. Installer connects to vCenter
                         ↓
7. Bootstrap VM + 3 Control Plane VMs are created
                         ↓
8. Bootstrap starts temporary control plane / temporary etcd
                         ↓
9. Masters form permanent etcd + control plane
                         ↓
10. CVO / Cluster Operators start
                         ↓
11. Bootstrap VM is deleted
                         ↓
12. Machine API creates Worker VMs
                         ↓
13. Workers boot RHCOS and join cluster
                         ↓
14. OVN-Kubernetes + Ingress + CSI + remaining Operators initialize
                         ↓
15. Validate with oc get nodes / oc get co
                         ↓
                      READY
```

---

# 33. Interview Answer

If asked **"Explain how you install OpenShift using IPI on vSphere"**, use this structure:

> First I prepare the VMware prerequisites: vCenter, datacenter, compute cluster, datastore, port group/network, DNS, API and Ingress VIPs, and a vCenter account with the required privileges. I also obtain the Red Hat pull secret and prepare an SSH key.
>
> I use `openshift-install create install-config` to create `install-config.yaml`, where I define the vSphere placement details, control-plane and worker counts, network, VIPs, base domain and cluster name.
>
> I then run `openshift-install create cluster`. Because this is IPI, the installer connects to vCenter and creates the required OpenShift virtual machines automatically. It first brings up a temporary bootstrap VM and the three control-plane VMs.
>
> The bootstrap machine runs a temporary etcd and Kubernetes/OpenShift control plane that helps initialize the permanent masters. The masters then form the production etcd cluster and permanent OpenShift control plane. Once they are self-sufficient, bootstrap completes and the installer deletes the bootstrap VM.
>
> The Machine API then manages the worker VM lifecycle through vCenter. Workers boot RHCOS, receive their machine configuration, start kubelet and register with the API server. OpenShift Operators then bring up networking, ingress, authentication, monitoring, storage and the remaining platform services.
>
> Finally I validate the installation using `oc get nodes`, `oc get co`, `oc get clusterversion`, and Machine/MachineSet resources.

---

# Quick Memory Line

```text
install-config.yaml
      ↓
openshift-install
      ↓
vCenter
      ↓
Bootstrap
      ↓
3 Masters + permanent etcd
      ↓
Bootstrap removed
      ↓
Machine API
      ↓
Workers
      ↓
Operators
      ↓
Ready OpenShift Cluster
```
