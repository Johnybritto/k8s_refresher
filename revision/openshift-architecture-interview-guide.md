# OpenShift Architecture — Interview Guide

## 1. Elevator Pitch

Red Hat OpenShift is an enterprise Kubernetes platform that extends upstream Kubernetes with an opinionated operating model, integrated lifecycle management, platform Operators, security controls, ingress, authentication, monitoring integrations, developer workflows, and tight management of the node operating system through Red Hat Enterprise Linux CoreOS (RHCOS).

The simplest way to explain it in an interview is:

> Kubernetes provides the orchestration foundation. OpenShift adds an enterprise platform around it: lifecycle management, Operators, integrated security, networking, routing, authentication, upgrades, developer tooling, and management of the underlying RHCOS nodes.

Important interview nuance:

- Do not say vanilla Kubernetes does "only orchestration" and nothing else. Kubernetes already provides scheduling, service discovery, controllers, RBAC, networking APIs, storage APIs, etc.
- The stronger answer is that OpenShift packages these capabilities into a more opinionated, integrated, and supported enterprise platform.

---

# 2. High-Level Architecture

```text
                            USERS / DEVELOPERS
                                   |
                     +-------------+-------------+
                     |                           |
                     v                           v
                 Cluster API                Application Routes
                     |                           |
                     v                           v
              API Load Balancer           Ingress Load Balancer
                     |                           |
                     v                           v
       +---------------------------------------------------+
       |                 CONTROL PLANE                     |
       |                                                   |
       |  kube-apiserver   scheduler   controllers         |
       |  etcd             OpenShift API / OAuth           |
       |  Cluster Operators / CVO                          |
       |  RHCOS                                            |
       +---------------------------------------------------+
                           |
                           | desired state / scheduling
                           v
       +---------------------------------------------------+
       |                 WORKER NODES                      |
       |                                                   |
       |  kubelet        CRI-O                             |
       |  application pods                                 |
       |  OVN-Kubernetes components                        |
       |  CSI node plugins                                 |
       |  RHCOS                                            |
       +---------------------------------------------------+
                           |
                 +---------+---------+
                 |                   |
                 v                   v
              Storage            External Systems
             via CSI         DB / APIs / Messaging
```

Memory line:

```text
Control plane decides.
Worker nodes execute.
Operators reconcile.
RHCOS provides the managed node OS.
```

---

# 3. The Foundation — RHCOS

OpenShift 4 tightly integrates the operating system into the cluster lifecycle.

RHCOS is the container-optimized, immutable-style operating system used by OpenShift control-plane nodes and commonly by worker nodes.

RHCOS includes:

- RHEL kernel
- SELinux enabled by default
- kubelet
- CRI-O
- Ignition for first-boot provisioning
- rpm-ostree / bootable image based OS lifecycle mechanisms

## Why RHCOS matters

The important OpenShift idea is that the operating system is treated as part of the cluster lifecycle rather than something administrators normally patch manually with traditional package-management workflows.

### Ignition

Ignition is primarily used during first boot to provision a machine into the required initial state.

```text
install-config
     |
     v
Ignition configuration
     |
     v
Node first boot
     |
     v
RHCOS configured
```

### Machine Config Operator

After installation, node-level configuration changes are managed through the Machine Config Operator (MCO).

The MCO can manage items such as:

- kubelet configuration
- CRI-O configuration
- kernel parameters
- systemd units
- NetworkManager configuration
- files and certificates
- OS updates

Typical flow:

```text
MachineConfig
     |
     v
Machine Config Operator
     |
     v
MachineConfigPool
     |
 +---+---+---+
 |       |   |
node-1 node-2 node-3
```

Important distinction:

```text
MachineSet / ComputeMachineSet = manages machines / VMs
MachineConfigPool              = manages node OS configuration
```

---

# 4. Control Plane

The control plane is the brain of the OpenShift cluster.

A standard production OpenShift deployment normally uses three control-plane nodes to provide high availability and etcd quorum.

```text
Control Plane

master-1
master-2
master-3
    |
    +---- API servers
    +---- schedulers
    +---- controller managers
    +---- etcd members
    +---- OpenShift control-plane services
```

## kube-apiserver

The API server is the main entry point for Kubernetes API operations.

Requests from:

- `oc`
- `kubectl`
- OpenShift Console
- controllers
- Operators
- automation tools

ultimately interact with the API server.

Example:

```text
oc apply -f deployment.yaml
            |
            v
       API Server
            |
            v
          etcd
```

The API server handles API validation, authentication, authorization and communication with the cluster state.

Memory line:

```text
API server = front door of the cluster API.
```

---

# 5. etcd

etcd is the distributed key-value store that holds Kubernetes/OpenShift cluster state.

It stores resources such as:

- Deployments
- Services
- Routes
- ConfigMaps
- Secrets
- Nodes
- Operators
- cluster configuration

It does NOT store your application's business database.

```text
Application database != etcd
```

## Quorum

With three etcd members:

```text
etcd-1
etcd-2
etcd-3
```

quorum requires two members.

```text
3-node etcd cluster -> can tolerate 1 member failure
```

If two of three etcd members are unavailable, the control plane cannot continue normal writes because quorum is lost.

---

# 6. Scheduler

The scheduler decides which node should run an unscheduled pod.

It evaluates factors such as:

- CPU and memory
- node selectors
- affinity / anti-affinity
- taints and tolerations
- topology constraints
- resource requests

```text
New Pod
   |
   v
Scheduler
   |
Checks suitable nodes
   |
   v
Chooses Worker-2
```

Memory line:

```text
Scheduler decides WHERE a pod runs.
```

---

# 7. Controller Manager

Controllers continuously compare desired state with actual state.

Example:

```text
Desired replicas = 3
Actual replicas  = 2
        |
        v
Controller detects mismatch
        |
        v
Creates another pod
```

Memory line:

```text
Controllers keep actual state matching desired state.
```

---

# 8. OpenShift API and OAuth

OpenShift adds platform-specific APIs and integrated authentication components around Kubernetes.

Typical authentication flow:

```text
Developer
   |
   v
OpenShift OAuth
   |
   +--> LDAP / AD
   +--> OIDC
   +--> GitHub / other IdP
   |
   v
Authenticated identity
   |
   v
RBAC authorization
```

Important distinction:

```text
Authentication = Who are you?
Authorization  = What are you allowed to do?
```

---

# 9. Worker Nodes

Worker nodes run application workloads.

Typical worker components:

```text
Worker Node
-------------------------
kubelet
CRI-O
OVN-Kubernetes components
application pods
CSI node plugins
RHCOS
-------------------------
```

## kubelet

The kubelet is the node agent.

It watches the desired pod state assigned to its node and ensures the requested containers are running.

```text
Scheduler chooses node
        |
        v
      kubelet
        |
        v
       CRI-O
        |
        v
    Containers run
```

Memory line:

```text
Scheduler chooses the node.
kubelet makes it happen on that node.
```

## CRI-O

OpenShift uses CRI-O as its Kubernetes container runtime.

CRI-O is responsible for operations such as:

- pulling images
- creating containers
- starting containers
- stopping containers
- maintaining container runtime lifecycle

Memory line:

```text
kubelet = node agent
CRI-O   = container runtime
```

---

# 10. What Happens When an Application Is Deployed?

Suppose the developer runs:

```bash
oc apply -f deployment.yaml
```

A simplified sequence is:

```text
Developer
   |
   v
API Server
   |
   v
etcd stores desired state
   |
   v
Deployment Controller
   |
   v
ReplicaSet
   |
   v
Pods created
   |
   v
Scheduler chooses workers
   |
   v
kubelet sees assigned Pod
   |
   v
CRI-O pulls image and starts containers
```

This is an excellent interview flow to remember.

---

# 11. Operators — A Major OpenShift Differentiator

Operators are one of the most important ideas in OpenShift.

An Operator is essentially a Kubernetes controller with domain-specific operational knowledge.

```text
Desired configuration
       |
       v
    Operator
       |
       v
Checks actual state
       |
 +-----+------+------+
 |            |      |
Healthy     Broken  Changed
 |            |      |
Nothing     Repair  Reconcile
```

## Cluster Operators

Cluster Operators manage core OpenShift platform components.

Examples include:

- Ingress Operator
- Network Operator
- Authentication Operator
- etcd Operator
- DNS Operator
- Machine Config Operator
- storage-related Operators

## OLM-managed Operators

Operator Lifecycle Manager (OLM) is used for many add-on and application Operators.

Examples:

- databases
- AMQ
- OpenShift Data Foundation
- third-party products

Important nuance:

Core OpenShift Cluster Operators are not simply ordinary OLM-installed application Operators.

Memory line:

```text
Cluster Operators run OpenShift itself.
OLM manages many optional/application Operators.
```

---

# 12. Cluster Version Operator — CVO

The Cluster Version Operator coordinates OpenShift platform updates.

Simplified view:

```text
Requested OpenShift version
         |
         v
Cluster Version Operator
         |
         v
Cluster Operators reconcile
         |
         v
Platform components update
```

The Machine Config Operator coordinates required node OS/configuration updates.

Important correction for interviews:

Do NOT say:

> OpenShift upgrades guarantee zero application downtime.

Better answer:

> OpenShift provides orchestrated rolling platform and node upgrades. Application availability during an upgrade depends on application design, replica count, PodDisruptionBudgets, topology placement, storage behavior and other HA controls.

---

# 13. Networking — OVN-Kubernetes

OVN-Kubernetes is the default OpenShift networking implementation in current OpenShift releases.

It provides the cluster network fabric for pod and service communication.

Simplified cross-node flow:

```text
Pod A
Worker-1
   |
OVN / OVS
   |
Overlay / routed cluster network
   |
OVN / OVS
Worker-2
   |
Pod B
```

It supports areas such as:

- pod-to-pod connectivity
- Service connectivity
- NetworkPolicy enforcement
- north-south traffic integration
- east-west communication

### Multus

OpenShift can also use Multus where pods need multiple network interfaces.

Example use cases:

- telecom/NFV
- storage traffic
- separate management/data networks

Do not describe Multus as replacing OVN-Kubernetes. It is normally used to attach additional networks to pods.

---

# 14. Service Networking

Pods are ephemeral, so applications normally communicate through Kubernetes Services.

```text
Frontend Pod
     |
     v
backend-service
     |
 +---+---+
 |       |
pod-1  pod-2
```

Memory line:

```text
Pod IPs change.
Services provide a stable endpoint.
```

---

# 15. External Traffic — Routes and Ingress

For application traffic:

```text
Internet / Corporate User
          |
          v
External Load Balancer
          |
          v
OpenShift Ingress Router
      (commonly HAProxy)
          |
          v
Route / Ingress
          |
          v
Service
          |
          v
Pods
```

OpenShift Routes provide native OpenShift ingress capabilities, including TLS termination modes such as:

- edge
- passthrough
- re-encrypt

Standard Kubernetes Ingress resources are also supported on OpenShift.

Memory line:

```text
Route = OpenShift-native HTTP(S) exposure
Ingress = Kubernetes-standard HTTP(S) exposure
```

---

# 16. API Traffic vs Application Traffic

This is an important architecture distinction.

## Cluster/API traffic

```text
oc / kubectl / Console
         |
         v
api.cluster.example.com
         |
         v
API Load Balancer
         |
         v
Control Plane API Servers
```

## Application traffic

```text
User
 |
 v
*.apps.cluster.example.com
 |
 v
Ingress Load Balancer
 |
 v
Router
 |
 v
Route
 |
 v
Service -> Pods
```

Memory line:

```text
api.*  = manage the cluster
*.apps = access applications
```

---

# 17. Storage Architecture

OpenShift uses CSI for modern storage integration.

Typical flow:

```text
Pod
 |
 v
PVC
 |
 v
StorageClass
 |
 v
CSI Driver
 |
 v
Storage Backend
```

Examples of storage backends:

- AWS EBS
- Azure Disk
- vSphere datastore
- Ceph / OpenShift Data Foundation
- enterprise SAN/NAS depending on CSI provider

Memory line:

```text
PVC          = app requests storage
StorageClass = defines how storage is provided
CSI          = integration with storage backend
PV           = allocated persistent storage
```

---

# 18. Security Architecture

A simple OpenShift security flow is:

```text
User
 |
 v
OAuth / Identity Provider
 |
 v
RBAC
 |
 v
Project / Namespace
 |
 v
Service Account
 |
 v
SCC
 |
 v
Pod
 |
 v
NetworkPolicy
```

## RBAC

Controls API authorization.

```text
Who can perform which API actions?
```

## SecurityContextConstraints — SCC

SCC governs security-sensitive pod/container settings such as:

- privileged mode
- host networking
- host PID/IPC
- hostPath-style permissions
- user/group execution constraints
- filesystem/security settings

Current OpenShift releases use restrictive defaults for normal workloads, such as `restricted-v2` for newly created pods.

Important wording correction:

Do not say SCC is "stricter than RBAC." They solve different problems.

Better:

```text
RBAC = who can call which APIs
SCC  = under which security conditions a pod may run
```

## NetworkPolicy

Controls allowed network communication between workloads.

Memory line:

```text
RBAC          = API permissions
SCC           = pod security permissions
NetworkPolicy = network communication permissions
```

---

# 19. Integrated Image Registry

OpenShift provides an integrated image registry capability managed by the Image Registry Operator.

It can be used for cluster-local application images and integrates with OpenShift image resources and build/deployment workflows.

Important nuance:

The registry is integrated into the platform, but its storage and exposure must still be correctly configured for the environment.

Do not describe it as eliminating all external registry requirements. Enterprises frequently also use external registries such as Quay, Artifactory, Harbor, ECR or ACR.

---

# 20. Source-to-Image — S2I

S2I is an OpenShift build workflow that can turn source code into a runnable container image by combining application source with a builder image.

Simplified flow:

```text
Git source code
      |
      v
S2I builder image
      |
      v
Build
      |
      v
Container image
      |
      v
Deployment
```

Interview nuance:

S2I is a supported and useful OpenShift developer workflow, but it is not the only way applications are built or deployed.

Modern environments may also use:

- Dockerfile/Containerfile builds
- Tekton / OpenShift Pipelines
- GitLab CI
- Jenkins
- GitHub Actions
- external build systems
- GitOps deployment using Argo CD

So say:

> OpenShift supports S2I for developer-friendly source-to-image builds, but production organizations often integrate external CI/CD and GitOps workflows as well.

---

# 21. High Availability

A standard production control plane commonly looks like:

```text
             API Load Balancer
                    |
        +-----------+-----------+
        |           |           |
     master-1    master-2    master-3
        |           |           |
        +------ etcd quorum -----+
```

Worker layer:

```text
Ingress LB
    |
 +--+-------------------+
 |                      |
router-1             router-2
 |                      |
 +----------+-----------+
            |
         Services
            |
   +--------+--------+
   |        |        |
 worker-1 worker-2 worker-3
```

Application HA should also include:

- multiple pod replicas
- anti-affinity / topology spread
- PodDisruptionBudgets
- readiness/liveness/startup probes
- redundant ingress/router pods
- redundant storage where required
- multiple worker failure domains where platform supports them

---

# 22. What Happens If a Control-Plane Node Fails?

In a healthy three-control-plane cluster, one control-plane node can fail while etcd still maintains quorum.

Existing workloads on worker nodes can continue running.

However, the control plane is responsible for functions such as:

- scheduling new pods
- controllers reconciling state
- API operations
- configuration changes
- scaling decisions that depend on cluster APIs

So the senior-level answer is:

> Existing application containers may continue running even during some control-plane disruption, but once control-plane quorum or API functionality is lost, normal cluster management, scheduling and reconciliation are affected.

---

# 23. Infrastructure Nodes

Larger OpenShift environments commonly separate infrastructure workloads from business application workloads.

Example:

```text
Control Plane
-------------
master-1
master-2
master-3

Infra Nodes
-----------
routers
registry
monitoring/logging components where appropriate

Application Workers
-------------------
business application pods
```

The exact placement depends on platform design and subscription/licensing considerations.

---

# 24. vSphere-Oriented Architecture

For a vSphere-based OpenShift environment:

```text
                         Corporate Users
                               |
                    +----------+----------+
                    |                     |
                  API LB                Apps LB
                    |                     |
                    v                     v
           api.cluster.example       *.apps.example
                    |                     |
                    v                     v
             Control Plane         HAProxy Routers
          +------+------+                |
          |      |      |                v
       Master1 Master2 Master3         Services
          \      |      /                |
             etcd quorum                 v
                                     Application Pods
                                         |
                        +----------------+----------------+
                        |                                 |
                    Worker Nodes                     Infra Nodes
                 kubelet + CRI-O                 Router/Monitoring
                        |
                        v
                   OVN-Kubernetes
                        |
                        v
                  vSphere Network
```

Persistent storage:

```text
Pod
 |
PVC
 |
StorageClass
 |
vSphere CSI
 |
vSphere Datastore
```

Machine lifecycle on supported vSphere deployments:

```text
Machine API / ComputeMachineSet
          |
          v
       vCenter
          |
          v
          VM
          |
          v
    OpenShift Node
```

---

# 25. Interview Question — What Makes OpenShift Different from Kubernetes?

A strong answer:

> OpenShift is built on Kubernetes, so the core orchestration model is the same: API server, etcd, scheduler, controllers, kubelet, Services and pods. OpenShift adds a highly integrated enterprise operating model around Kubernetes. It manages the RHCOS node operating system, uses Cluster Operators to manage platform components, provides coordinated upgrades through the Cluster Version Operator and Machine Config Operator, integrates OAuth, Routes, SCCs, monitoring, ingress and an internal registry capability, and provides a more opinionated lifecycle and security model. This reduces the amount of platform integration that an enterprise team has to assemble and maintain itself.

---

# 26. Interview Question — How Does an App Get Deployed?

A strong generic answer:

> The developer or pipeline submits the desired resource definition to the OpenShift API. The API server validates and stores the desired state in etcd. Controllers create the required ReplicaSet and Pod objects. The scheduler selects suitable worker nodes. The kubelet on those workers sees the assignments and asks CRI-O to pull and start the containers. Services provide stable access to the pods, and Routes or Ingress can expose HTTP or HTTPS applications externally.

Then, if asked about developer builds:

> OpenShift can also use S2I to turn source code into an image using builder images, although many enterprises integrate GitLab CI, Jenkins, Tekton or other CI systems and deploy through GitOps.

---

# 27. Interview Question — How Does OpenShift Achieve HA?

A strong answer:

> At the control-plane layer, I normally deploy three control-plane nodes so etcd maintains quorum and API/control-plane components have redundancy. One control-plane node can fail without losing quorum. At the ingress layer I run multiple router replicas behind a redundant load balancer. Worker capacity is distributed across multiple nodes and failure domains where possible. Applications should run multiple replicas with topology spread or anti-affinity, readiness probes and PodDisruptionBudgets. Persistent data and external dependencies must also be designed for HA. OpenShift provides the platform mechanisms, but application HA still depends on correct workload architecture.

---

# 28. Common Interview Traps

## Trap 1

Wrong:

```text
OpenShift is completely different from Kubernetes.
```

Correct:

```text
OpenShift is built on Kubernetes and adds an integrated enterprise platform around it.
```

## Trap 2

Wrong:

```text
SCC is stricter RBAC.
```

Correct:

```text
RBAC controls API authorization.
SCC controls security conditions under which pods may run.
```

## Trap 3

Wrong:

```text
OpenShift upgrades guarantee zero downtime.
```

Correct:

```text
OpenShift performs coordinated rolling platform/node updates, but application availability depends on the workload's HA design.
```

## Trap 4

Wrong:

```text
S2I is how every OpenShift application is deployed.
```

Correct:

```text
S2I is one supported build workflow. OpenShift also integrates with normal CI/CD and GitOps approaches.
```

## Trap 5

Wrong:

```text
If the control plane goes down, all running pods immediately stop.
```

Correct:

```text
Existing workloads can continue running for some time, but scheduling, reconciliation and management operations are impacted.
```

---

# 29. 3-Minute Interview Answer

> OpenShift is an enterprise Kubernetes platform. The underlying architecture still has Kubernetes control-plane and worker-node concepts, but OpenShift adds integrated lifecycle management, Operators, RHCOS, authentication, routing, security and developer/platform tooling.
>
> In production I normally expect three control-plane nodes. They run the API server, scheduler, controller managers, etcd and OpenShift-specific control-plane services. The API server handles cluster API requests, etcd stores the cluster state, the scheduler decides where new pods run, and controllers continuously reconcile actual state with desired state.
>
> Worker nodes run RHCOS, kubelet, CRI-O, OVN-Kubernetes components and the actual application pods. Once a pod is scheduled to a worker, kubelet ensures the requested containers are running and CRI-O provides the container runtime.
>
> A major OpenShift differentiator is the Operator model. Core platform capabilities such as networking, ingress, authentication and etcd are managed through Cluster Operators. The Cluster Version Operator coordinates platform upgrades, while the Machine Config Operator manages RHCOS and node-level configuration such as kubelet, CRI-O, systemd and kernel settings.
>
> Networking normally uses OVN-Kubernetes. External application traffic flows through a load balancer to the OpenShift ingress router, then through a Route or Ingress to a Service and finally the application pods. API traffic is separate and goes through the API load balancer to the control-plane API servers.
>
> Persistent storage is integrated through CSI using PVCs and StorageClasses. For example on vSphere, a PVC can be dynamically provisioned through the vSphere CSI driver.
>
> From a security perspective, OAuth handles authentication, RBAC handles API authorization, SCC controls the security conditions under which pods can run, and NetworkPolicy controls workload communication.
>
> For HA, I use three control-plane nodes to maintain etcd quorum, redundant ingress/router instances, multiple worker nodes and application replicas, topology spread, PodDisruptionBudgets and highly available storage and external dependencies.

---

# 30. Final Memory Diagram

```text
                        OPENSHIFT

                     CONTROL PLANE

API Server -> etcd -> Scheduler -> Controllers
     |
     +--> OpenShift APIs / OAuth
     +--> Cluster Operators
     +--> CVO
     +--> MCO

                           |
                           v

                       WORKERS

                kubelet -> CRI-O -> Pods
                           |
          +----------------+----------------+
          |                                 |
       Network                           Storage
    OVN-Kubernetes                        CSI
          |                                 |
Route/Ingress -> Service -> Pod         PVC -> PV

Security:
OAuth -> RBAC -> SCC -> NetworkPolicy

Lifecycle:
CVO -> Platform Operators
MCO -> RHCOS / Node configuration
```

## Final Memory Lines

```text
Control plane decides.
Workers execute.
Operators reconcile.
etcd stores cluster state.
RHCOS is managed as part of the platform lifecycle.
OVN-Kubernetes provides cluster networking.
Routes/Ingress expose applications.
CSI connects workloads to persistent storage.
RBAC controls API access; SCC controls pod security.
```

---

# Verification Notes

This guide was corrected against current Red Hat OpenShift Container Platform 4.20 documentation before being committed.

Important verified corrections include:

- RHCOS is integrated into OpenShift lifecycle management and includes kubelet, CRI-O and Ignition.
- MCO manages OS/runtime configuration such as systemd, CRI-O, kubelet, kernel and NetworkManager settings.
- OpenShift platform updates are coordinated by CVO, MCO and Cluster Operators.
- `restricted-v2` is a default SCC applied to newly created pods in current OpenShift releases.
- OpenShift upgrade orchestration does not by itself guarantee application zero downtime.
- S2I is a supported build workflow, not the only OpenShift application deployment model.
