# OpenShift on VMware vSphere — UPI Cluster Build Procedure

## Purpose

This document is an interview/refresher guide for building an OpenShift cluster on **VMware vSphere using UPI (User-Provisioned Infrastructure)**.

The key idea is:

> **With UPI, you provision the infrastructure and machines; OpenShift installs and configures OpenShift on those machines.**

---

# 1. End-to-End Flow

```text
Prepare vSphere infrastructure
        ↓
Plan IPs / hostnames
        ↓
Configure DNS
        ↓
Configure API + Ingress Load Balancers
        ↓
Download openshift-install + oc
        ↓
Create install-config.yaml
        ↓
Generate manifests
        ↓
Generate Ignition files
        ↓
Download/import RHCOS OVA
        ↓
Create Bootstrap VM manually
        ↓
Create 3 Master VMs manually
        ↓
Create Worker VMs manually
        ↓
Attach correct Ignition to each VM
        ↓
Power on machines
        ↓
Bootstrap creates temporary control plane
        ↓
Masters form permanent etcd/control plane
        ↓
Wait for bootstrap-complete
        ↓
Remove Bootstrap from Load Balancer
        ↓
Delete Bootstrap VM
        ↓
Approve Worker CSRs if required
        ↓
Workers become Ready
        ↓
Operators initialize
        ↓
wait-for install-complete
        ↓
Cluster Ready
```

---

# 2. Prepare vSphere Infrastructure

With UPI, the underlying VMware environment must already exist.

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
Port Group / VLAN
```

Example:

```text
vCenter:      vcsa.company.com
Datacenter:   DC01
Cluster:      Prod-Compute
Resource Pool: OCP-Prod
Datastore:    vsanDatastore
Network:      OCP-Prod-VLAN
```

### What you do

Prepare:

- vCenter
- ESXi cluster
- Resource pool
- Datastore
- VLAN / Port Group
- VM folder
- Capacity
- Permissions

### What OpenShift does

Nothing yet.

Key difference:

```text
IPI:
openshift-install → vCenter → creates VMs

UPI:
YOU → vCenter → create VMs
```

---

# 3. Plan Node IPs and Hostnames

Example:

```text
Network:
10.20.30.0/24

Load Balancers:
API LB      10.20.30.10
Ingress LB  10.20.30.11

Bootstrap:
bootstrap   10.20.30.20

Masters:
master-0    10.20.30.21
master-1    10.20.30.22
master-2    10.20.30.23

Workers:
worker-0    10.20.30.31
worker-1    10.20.30.32
worker-2    10.20.30.33
```

You can use DHCP or static addressing. For DHCP, keep persistent mappings/reservations for the nodes.

---

# 4. Configure DNS Manually

Example:

```text
Cluster name: ocp-prod
Base domain:  company.com

Cluster domain:
ocp-prod.company.com
```

Create records:

```text
api.ocp-prod.company.com
        ↓
10.20.30.10

api-int.ocp-prod.company.com
        ↓
10.20.30.10

*.apps.ocp-prod.company.com
        ↓
10.20.30.11
```

Also create node records:

```text
bootstrap.ocp-prod.company.com
master-0.ocp-prod.company.com
master-1.ocp-prod.company.com
master-2.ocp-prod.company.com
worker-0.ocp-prod.company.com
worker-1.ocp-prod.company.com
worker-2.ocp-prod.company.com
```

Reverse DNS should also be planned where required.

### API path

```text
oc
 ↓
api.ocp-prod.company.com
 ↓
API Load Balancer
 ↓
Masters
```

### Application path

```text
User
 ↓
myapp.apps.ocp-prod.company.com
 ↓
Ingress Load Balancer
 ↓
OpenShift Router
 ↓
Service
 ↓
Pod
```

---

# 5. Configure Load Balancers Manually

UPI requires external load balancing to be prepared manually.

Possible platforms:

- F5
- HAProxy
- NSX Advanced Load Balancer / AVI
- Citrix ADC
- Other enterprise Layer-4 load balancer

## API Load Balancer

Initially:

```text
                 API LB
                /      \
             :6443    :22623
               |         |
       -------------------------
       |       |       |       |
   Bootstrap Master0 Master1 Master2
```

Ports:

```text
6443  = Kubernetes / OpenShift API
22623 = Machine Config Server
```

During bootstrap, bootstrap and masters are included as backends.

### API path

```text
api.ocp-prod.company.com
         ↓
       API LB
         ↓
     TCP 6443
         ↓
master-0/master-1/master-2
```

### Machine Config Server path

```text
api-int.ocp-prod.company.com
         ↓
       API LB
         ↓
     TCP 22623
         ↓
Bootstrap/Masters
```

Use Layer-4 / raw TCP load balancing and avoid session persistence for the API.

---

# 6. Configure Ingress Load Balancer

Ingress handles applications and the OpenShift console.

```text
*.apps.ocp-prod.company.com
             ↓
         Ingress LB
          /       \
       :80        :443
        |           |
        +-----------+
             ↓
          Workers
             ↓
      OpenShift Routers
```

Ports:

```text
80  = HTTP
443 = HTTPS
```

Backends are normally the nodes where router pods run, typically workers.

---

# 7. Download OpenShift Tools

Required binaries:

```text
openshift-install
oc
```

In UPI, the installer is mainly used to generate:

```text
install-config
manifests
Ignition configs
kubeconfig
metadata
```

The actual infrastructure and VMs are provisioned by you.

---

# 8. Prepare Pull Secret and SSH Key

Get the Red Hat pull secret.

Generate an SSH key if required:

```bash
ssh-keygen -t ed25519
```

Provide the pull secret and SSH public key in the installation configuration.

---

# 9. Create install-config.yaml

Create a working directory:

```bash
mkdir ocp-prod
```

Generate configuration:

```bash
openshift-install create install-config \
  --dir=ocp-prod
```

For UPI, a common design is:

```yaml
compute:
- name: worker
  replicas: 0

controlPlane:
  name: master
  replicas: 3
```

Why worker replicas = 0?

Because you are creating worker VMs yourself.

The control-plane replica count should match the number of control-plane VMs you deploy.

Back up `install-config.yaml` before continuing because later installer commands consume generated assets.

---

# 10. Generate Manifests

Run:

```bash
openshift-install create manifests \
  --dir=ocp-prod
```

This generates Kubernetes/OpenShift manifests describing the desired cluster configuration.

Conceptually:

```text
install-config.yaml
       ↓
OpenShift manifests
       ↓
desired cluster configuration
```

These include configuration for areas such as:

- Cluster networking
- Scheduler
- Infrastructure
- Operators
- Machine configuration

---

# 11. Verify Masters Are Not Schedulable

Check:

```text
manifests/cluster-scheduler-02-config.yml
```

Typically:

```yaml
mastersSchedulable: false
```

Meaning:

```text
Masters → control-plane workloads
Workers → application workloads
```

---

# 12. Generate Ignition Configurations

Run:

```bash
openshift-install create ignition-configs \
  --dir=ocp-prod
```

This creates:

```text
ocp-prod/
 |
 +-- bootstrap.ign
 +-- master.ign
 +-- worker.ign
 +-- metadata.json
 +-- auth/
      +-- kubeconfig
      +-- kubeadmin-password
```

---

# 13. Understand the Ignition Files

## bootstrap.ign

Used by:

```text
Bootstrap VM
```

Purpose:

- Configure bootstrap RHCOS
- Start temporary etcd
- Start temporary Kubernetes/OpenShift control plane

## master.ign

Used by:

```text
master-0
master-1
master-2
```

All three masters can use the same master Ignition configuration.

## worker.ign

Used by:

```text
worker-0
worker-1
worker-2
worker-N
```

Summary:

```text
bootstrap.ign → Bootstrap
master.ign    → Masters
worker.ign    → Workers
```

---

# 14. Host bootstrap.ign on an HTTP Server

For vSphere UPI, `bootstrap.ign` is typically hosted on an HTTP server reachable by the bootstrap VM.

Example:

```text
HTTP server:
10.20.30.50

URL:
http://10.20.30.50/bootstrap.ign
```

Create a smaller merge Ignition file that points to the real bootstrap Ignition file.

Conceptually:

```text
Bootstrap VM
     ↓
merge-bootstrap.ign
     ↓
HTTP server
     ↓
bootstrap.ign
     ↓
Bootstrap configuration
```

---

# 15. Base64 Encode Ignition Configurations

Examples:

```bash
base64 -w0 master.ign > master.64
base64 -w0 worker.ign > worker.64
base64 -w0 merge-bootstrap.ign > merge-bootstrap.64
```

These values are injected into VMware VM guestinfo properties.

---

# 16. Download and Import RHCOS OVA

Download the RHCOS VMware OVA compatible with the OpenShift release.

Example conceptually:

```text
rhcos-vmware.x86_64.ova
```

Import into vSphere:

```text
RHCOS OVA
    ↓
vSphere
    ↓
RHCOS Template / source VM
```

All cluster VMs can be cloned from this common base.

---

# 17. Create Bootstrap VM Manually

Clone the RHCOS template.

Example:

```text
bootstrap.ocp-prod.company.com
10.20.30.20
```

Configure VM guestinfo properties:

```text
guestinfo.ignition.config.data
      =
merge-bootstrap.64

guestinfo.ignition.config.data.encoding
      =
base64
```

Also configure:

```text
disk.EnableUUID = TRUE
```

Where required, static network parameters can also be passed through guestinfo/afterburn configuration.

---

# 18. Create Three Master VMs

Clone:

```text
master-0
master-1
master-2
```

Inject:

```text
guestinfo.ignition.config.data = master.64
```

Conceptually:

```text
RHCOS template
        |
        +---- master-0 + master.ign
        +---- master-1 + master.ign
        +---- master-2 + master.ign
```

Also configure:

```text
disk.EnableUUID = TRUE
```

---

# 19. Create Worker VMs

Clone:

```text
worker-0
worker-1
worker-2
```

Inject:

```text
guestinfo.ignition.config.data = worker.64
```

Flow:

```text
RHCOS Template
      ↓
Worker VM
      ↓
worker.ign
```

Again configure:

```text
disk.EnableUUID = TRUE
```

---

# 20. Power On Bootstrap and Masters

Now the actual OpenShift installation begins.

Bootstrap flow:

```text
Bootstrap VM
     ↓
RHCOS
     ↓
Ignition
     ↓
Temporary etcd
     ↓
Temporary Kubernetes control plane
```

Masters:

```text
master-0
master-1
master-2
      ↓
RHCOS
      ↓
master.ign
```

They begin obtaining the configuration necessary to form the permanent control plane.

---

# 21. Bootstrap Builds the Permanent Control Plane

```text
                   Bootstrap
                      |
              Temporary etcd
              Temporary API
              Temporary controllers
                      |
              ------------------
              |        |       |
           Master0  Master1 Master2
```

Bootstrap helps bring up:

- etcd
- kube-apiserver
- kube-controller-manager
- kube-scheduler
- OpenShift APIs
- Cluster Version Operator
- Cluster Operators

---

# 22. Permanent etcd Forms

Eventually:

```text
Master0
Master1
Master2
   |
   +----------+
   |  etcd    |
   | cluster  |
   +----------+
```

For a 3-member etcd cluster:

```text
Quorum = 2 of 3
```

Once permanent etcd and the control plane are healthy, bootstrap is no longer required.

---

# 23. Permanent API Becomes Available

Traffic:

```text
api.ocp-prod.company.com
        ↓
API Load Balancer
        ↓
Master0
Master1
Master2
        ↓
kube-apiserver
```

Test:

```bash
curl -k https://api.ocp-prod.company.com:6443/version
```

---

# 24. Wait for Bootstrap Completion

Run:

```bash
openshift-install \
  --dir=ocp-prod \
  wait-for bootstrap-complete \
  --log-level=info
```

Successful completion means the permanent control plane can operate independently.

Typical message:

```text
It is now safe to remove the bootstrap resources
```

---

# 25. Remove Bootstrap from the API Load Balancer

This step is manual in UPI.

Before:

```text
API LB
 |
 +-- bootstrap
 +-- master0
 +-- master1
 +-- master2
```

After:

```text
API LB
 |
 +-- master0
 +-- master1
 +-- master2
```

Remove bootstrap from both:

```text
6443 backend
22623 backend
```

---

# 26. Delete the Bootstrap VM

After bootstrap completion and removal from the load balancer:

```text
Bootstrap VM → DELETE
```

Permanent cluster:

```text
Master0
Master1
Master2

Worker0
Worker1
Worker2
```

---

# 27. Worker Join Process

Worker lifecycle:

```text
Worker VM
    ↓
RHCOS
    ↓
worker.ign
    ↓
Machine Config Server
    ↓
api-int:22623
    ↓
kubelet starts
    ↓
api-int:6443
    ↓
requests to join cluster
```

---

# 28. Approve CSRs Where Required

Set kubeconfig:

```bash
export KUBECONFIG=ocp-prod/auth/kubeconfig
```

Check pending CSRs:

```bash
oc get csr
```

Validate each CSR belongs to an expected node before approving.

Example:

```bash
oc adm certificate approve <csr-name>
```

You may see two stages:

```text
Client CSR
    ↓
approve
    ↓
Node begins joining
    ↓
Serving CSR
    ↓
approve
```

Do not blindly approve unknown CSRs.

---

# 29. Workers Become Ready

Check:

```bash
oc get nodes
```

Expected:

```text
master-0   Ready   control-plane,master
master-1   Ready   control-plane,master
master-2   Ready   control-plane,master
worker-0   Ready   worker
worker-1   Ready   worker
worker-2   Ready   worker
```

---

# 30. Ingress Becomes Functional

```text
Ingress Operator
      ↓
Router Pods
      ↓
Workers
```

Application traffic:

```text
myapp.apps.ocp-prod.company.com
              ↓
DNS
              ↓
Ingress LB
              ↓
80 / 443
              ↓
Worker nodes
              ↓
Router
              ↓
Service
              ↓
Pod
```

---

# 31. Cluster Operators Converge

Check:

```bash
oc get co
```

Target state:

```text
AVAILABLE   PROGRESSING   DEGRADED
True        False         False
```

Also verify:

```bash
oc get clusterversion
```

---

# 32. Wait for Installation Completion

Run:

```bash
openshift-install \
  --dir=ocp-prod \
  wait-for install-complete
```

Eventually:

```text
Install complete!
```

---

# 33. IPI vs UPI — Key Difference

| Task | IPI | UPI |
|---|---|---|
| Create Bootstrap VM | Installer | You |
| Create Masters | Installer | You |
| Create Workers | Installer / Machine API | You |
| Configure Load Balancer | Automated/platform managed | You |
| Configure DNS | Mostly automated/platform managed | You |
| VM IP configuration | Automated | You |
| RHCOS deployment | Automated | You |
| Ignition generation | Installer | Installer |
| Bootstrap process | OpenShift | OpenShift |
| etcd creation | OpenShift | OpenShift |
| Control plane setup | OpenShift | OpenShift |
| Operators | OpenShift | OpenShift |
| Remove bootstrap from LB | Automated | You |
| Remove Bootstrap VM | Automated | You |
| Worker CSR handling | Usually automated | May require you |

Important:

> UPI does **not** mean manually installing kube-apiserver, etcd, scheduler, OVN or OpenShift Operators.

Those are still installed and managed by OpenShift.

UPI means you provision and manage the underlying infrastructure.

---

# 34. Installation Architecture

```text
                    DNS
                     |
             api.ocp-prod.company.com
                     |
                  API LB
             6443 / 22623
                     |
        +------------+-------------+
        |            |             |
    Bootstrap     Master0        Master1       Master2
        |            |             |             |
        |            +------ etcd cluster -------+
        |
 Temporary Kubernetes
 Temporary etcd
        |
        +---------- bootstrap permanent control plane


              *.apps.ocp-prod.company.com
                         |
                     Ingress LB
                      80 / 443
                         |
               +---------+---------+
               |         |         |
             Worker0   Worker1   Worker2
               |         |         |
                  Router Pods
```

After bootstrap:

```text
                    API LB
                       |
          +------------+------------+
          |            |            |
       Master0       Master1      Master2
          +------------+------------+
                       |
                      etcd

BOOTSTRAP = REMOVED
```

---

# 35. Short Memory Flow

```text
UPI:

Infra
 ↓
DNS + LB
 ↓
install-config
 ↓
manifests
 ↓
Ignition
 ↓
RHCOS template
 ↓
Create Bootstrap + Masters + Workers
 ↓
Power ON
 ↓
Bootstrap
 ↓
Permanent Masters + etcd
 ↓
Remove Bootstrap
 ↓
Approve Workers
 ↓
Operators
 ↓
READY
```

---

# 36. Interview Answer

If asked: **“Explain how you build an OpenShift cluster using UPI on VMware vSphere.”**

A strong answer is:

> In a vSphere UPI installation, we provision the underlying infrastructure ourselves. First I prepare the vSphere datacenter, ESXi cluster, resource pools, datastore, VLAN or port group, IP addressing, DNS and external API and Ingress load balancers.
>
> I create the OpenShift `install-config.yaml`, normally setting compute replicas to zero because workers are manually provisioned. I then use `openshift-install` to generate the Kubernetes manifests and the `bootstrap.ign`, `master.ign` and `worker.ign` Ignition configurations.
>
> I import the appropriate RHCOS OVA into vSphere and use it to manually provision one bootstrap VM, three control-plane VMs and the required worker VMs. Each VM receives the appropriate Ignition configuration through the vSphere guestinfo properties.
>
> Initially the API load balancer sends ports 6443 and 22623 to bootstrap and the three masters. The bootstrap node runs the temporary control plane and helps the masters establish the permanent etcd and OpenShift control plane.
>
> Once `openshift-install wait-for bootstrap-complete` succeeds, I remove bootstrap from the load balancer and delete the bootstrap VM.
>
> The worker VMs then join through the API, I validate and approve their CSRs where required, and finally verify that the nodes and Cluster Operators are healthy before running `wait-for install-complete`.
