# AKS Architecture & Networking — Consolidated Interview Notes

These notes consolidate the AKS architecture discussion and all follow-up questions into one interview-ready document.

---

## 1. AKS Architecture — Big Picture

```text
                         AZURE-MANAGED CONTROL PLANE
                  +-----------------------------------+
                  | kube-apiserver                    |
                  | etcd                              |
                  | scheduler                         |
                  | controller manager                |
                  | cloud controller manager          |
                  +----------------+------------------+
                                   |
                            Kubernetes API
                                   |
          +------------------------+------------------------+
          |                                                 |
          v                                                 v
 +----------------------+                         +----------------------+
 | SYSTEM NODE POOL     |                         | USER NODE POOL       |
 | Azure VM / VMSS      |                         | Azure VM / VMSS      |
 |                      |                         |                      |
 | kubelet              |                         | kubelet              |
 | containerd           |                         | containerd           |
 | Azure CNI            |                         | Azure CNI            |
 | kube-proxy/Cilium    |                         | kube-proxy/Cilium    |
 | CoreDNS/add-ons      |                         | application pods     |
 +----------+-----------+                         +----------+-----------+
            |                                                |
            +-------------------- Azure VNet ----------------+
                                   |
                    +--------------+--------------+
                    |              |              |
                   ACR         Key Vault     Azure Monitor
```

### Interview summary

AKS separates the cluster into:

1. **Azure-managed control plane** — API server, etcd, scheduler and controllers.
2. **Customer-managed worker/node pools** — VMs that run kubelet, containerd, networking and workloads.
3. **Azure integrations** — VNet, Load Balancer/Application Gateway, ACR, Key Vault, identities and monitoring.

The control plane decides **what should happen**.  
The node components make it **actually happen**.

---

# 2. Managed Control Plane

In AKS, Microsoft manages the Kubernetes control-plane infrastructure.

You do not normally create or operate the control-plane VMs yourself.

```text
Azure-managed
+--------------------------------+
| API Server                     |
| etcd                           |
| Scheduler                      |
| Controller Manager             |
| Cloud Controller Manager       |
+--------------------------------+

Customer-managed
+--------------------------------+
| Worker node pools              |
| kubelet                        |
| containerd                     |
| CNI                            |
| workloads                      |
+--------------------------------+
```

### Interview answer

> In AKS, Microsoft operates the Kubernetes control plane. We manage the node pools, workloads, networking design, security configuration, identities, policy, ingress and Azure integrations.

---


# 2A. AKS Automatic vs AKS Standard

AKS now has two cluster operating modes:

```text
                    Azure Kubernetes Service
                              |
                  +-----------+-----------+
                  |                       |
                  v                       v
          AKS Automatic              AKS Standard
          opinionated /              configurable /
          highly managed             operator controlled
```

Both are still AKS and both use the same core Kubernetes concepts. The main difference is **how much platform configuration and day-2 operation Azure performs for you**.

## AKS Automatic

AKS Automatic is the more opinionated, production-ready operating model.

Azure preconfigures and operates many platform capabilities so the application/platform team does not have to assemble the baseline themselves.

Typical Automatic behavior includes:

- Azure-managed node provisioning and node auto-provisioning
- managed system node pools
- automatic scaling based on workload demand
- automatic cluster upgrades through the stable channel
- NodeImage OS upgrade channel
- Azure RBAC for Kubernetes authorization
- OIDC issuer and Workload Identity enabled
- deployment safeguards/security controls enabled
- monitoring defaults such as Managed Prometheus and Container Insights
- HPA, VPA and KEDA capabilities available/preconfigured
- production networking defaults
- uptime SLA included
- qualifying pod-readiness SLA
- managed ingress/application-routing defaults

For networking, the important interview point is:

```text
AKS Automatic
      |
      v
Managed VNet / supported custom VNet
      |
      v
Azure CNI Overlay
      +
Cilium data plane
```

So many of the networking decisions discussed elsewhere in this document are already selected for you in the Automatic baseline.

Conceptually:

```text
Developer submits workload
          |
          v
       Scheduler
          |
          v
Node Auto Provisioning
          |
     selects/provisions
     appropriate compute
          |
          v
        Pod runs
```

The operator focuses more on **workloads and policies** than manually designing every node-pool lifecycle decision.

## AKS Standard

AKS Standard provides the traditional, highly configurable AKS operating model.

You make explicit platform decisions such as:

- node-pool design
- VM SKU and pool topology
- autoscaling configuration
- system vs user pools
- networking model
- CNI/data-plane selection
- ingress architecture
- maintenance windows
- cluster upgrade strategy
- node-image upgrade strategy
- monitoring integrations
- security features and policy
- public/private API exposure
- API Server VNet Integration
- egress architecture

Conceptually:

```text
Platform Team
     |
     +--> design node pools
     +--> choose networking
     +--> configure autoscaling
     +--> configure monitoring
     +--> define upgrade strategy
     +--> configure security
     +--> manage lifecycle
     |
     v
AKS Standard
```

## Side-by-Side Comparison

| Area | AKS Automatic | AKS Standard |
|---|---|---|
| Operating model | Opinionated / highly managed | Granular operator control |
| Best fit | Most new production workloads, application teams, fast onboarding | Platform teams, custom infrastructure requirements, existing automation |
| Node pools | Azure manages system nodes and workload-driven node provisioning | You create/manage node pools |
| Node scaling | Node auto-provisioning + workload scaling defaults | Cluster autoscaler/manual designs configured by operator |
| VM selection | More constrained by supported Automatic capabilities | Greater control over VM SKUs/topology |
| Networking | Opinionated production default, Azure CNI Overlay powered by Cilium | Broad choice of supported networking/data-plane models |
| Security | Several hardened controls preconfigured | Features selected/configured by operator |
| Monitoring | Important observability components enabled/defaulted | Explicitly enabled/configured |
| Cluster upgrades | Automatic stable-channel behavior | Manual by default; automatic channels optional |
| Node image upgrades | Automated channel | Operator controls strategy/channel |
| Ingress | Managed application-routing defaults available/preconfigured | Bring your own or enable managed options |
| Windows node pools | Not the primary Automatic model / use Standard for requirements needing Windows pools | Supported where AKS supports Windows |
| Uptime SLA | Included by default | Depends on selected pricing tier/configuration |
| Pod readiness SLA | Included for qualifying Automatic workloads | Not a Standard-mode feature |
| Day-2 operations | Lower operational burden | Higher control and higher operational responsibility |

## The Most Important Mental Model

Do not think:

```text
Automatic = different Kubernetes
Standard  = normal Kubernetes
```

Both are Kubernetes on AKS.

Think:

```text
AKS Automatic
    =
Azure chooses and operates more
of the platform defaults

AKS Standard
    =
Platform team chooses and operates
more of the platform configuration
```

## Relationship to Topics Already Covered

### Node pools

Standard:

```text
Platform team
   |
   +--> System Pool
   +--> General Pool
   +--> GPU Pool
   +--> Memory Pool
```

Automatic:

```text
Workload requests
      |
      v
Node Auto Provisioning
      |
      v
AKS dynamically provisions
appropriate compute
```

### Networking

Standard can involve deliberate choices such as:

```text
Azure CNI Overlay
Azure CNI Pod Subnet
Cilium
Calico
custom VNet
private cluster
custom ingress
custom egress
```

Automatic starts with a stronger opinionated baseline:

```text
Azure CNI Overlay
       +
Cilium
       +
managed networking defaults
```

### Upgrades

Standard:

```text
Platform Team
   |
   +--> choose Kubernetes version
   +--> control-plane upgrade
   +--> node-pool upgrade
   +--> node-image strategy
   +--> maintenance window
```

Automatic:

```text
Azure-managed upgrade channels
       |
       +--> Kubernetes stable channel
       +--> NodeImage channel
       |
       v
less manual lifecycle management
```

## When Would I Choose Each?

Choose **AKS Automatic** when:

- starting a new production platform
- you want strong defaults and lower operational overhead
- application teams should not manage node infrastructure
- standard Linux workloads fit the supported Automatic model
- you want Azure to handle more scaling, security, monitoring and upgrades

Choose **AKS Standard** when:

- you need precise node-pool/topology control
- you require custom networking or unusual routing
- you need Windows node pools
- you need VM SKUs or infrastructure patterns outside Automatic capabilities
- you already have mature AKS automation
- your organization requires explicit change control for upgrades
- you need granular control over maintenance and lifecycle operations

### Interview answer

> AKS Automatic and AKS Standard use the same underlying Kubernetes concepts, but they differ in operational ownership. Automatic is an opinionated production-ready mode where Azure preconfigures and manages node provisioning, scaling, security, networking, monitoring and upgrades. Standard gives the platform team direct control over node pools, networking, scaling, upgrade strategy and cluster lifecycle. I would use Automatic for workloads that fit the standard production model and Standard when infrastructure, networking or operational requirements demand granular control.

---

# 3. kube-apiserver

The API server is the **front door of Kubernetes**.

Almost all Kubernetes components communicate through it.

```text
kubectl / CI-CD / Argo CD
            |
            v
      kube-apiserver
            |
     +------+------+
     |             |
     v             v
    etcd      controllers
                   |
                scheduler
```

When you run:

```bash
kubectl apply -f deployment.yaml
```

the simplified flow is:

```text
kubectl
  |
  v
API Server
  |
  +--> authentication
  +--> authorization
  +--> admission
  |
  v
etcd
  |
  v
controllers + scheduler react
```

The API server does **not directly create the container**.

---

# 4. etcd Responsibility

etcd stores Kubernetes cluster state.

Examples:

- Deployments
- Pods
- Services
- ConfigMaps
- Secrets
- Nodes
- desired replica counts
- object metadata

```text
Desired state
replicas = 3
      |
      v
     etcd
      |
      v
controllers compare
desired vs actual
```

Memory line:

> **etcd remembers; controllers reconcile.**

---

# 5. Scheduler

The scheduler decides:

> Which node should this pod run on?

It evaluates conditions such as:

- CPU and memory requests
- taints and tolerations
- nodeSelector
- node affinity
- pod affinity / anti-affinity
- topology spread constraints
- available nodes

```text
New Pod
   |
   v
Scheduler
   |
   +--> Node1: insufficient CPU
   +--> Node2: suitable
   +--> Node3: taint not tolerated
   |
   v
Bind Pod -> Node2
```

Important:

> **Scheduler chooses the node. kubelet runs the pod.**

---

# 6. Controllers

Controllers continuously compare:

```text
Desired State
     vs
Actual State
```

Example:

```text
Deployment wants 3 replicas

Running:
Pod1 = OK
Pod2 = OK
Pod3 = missing

Controller detects mismatch
        |
        v
Create replacement Pod
```

This continuous convergence is called **reconciliation**.

---

# 7. System Node Pool vs User Node Pool

## System node pool

Used primarily for critical AKS/Kubernetes components, for example:

- CoreDNS
- metrics-server
- CSI components
- networking agents
- monitoring agents
- system add-ons

## User node pool

Used for application workloads:

- frontend
- backend
- APIs
- batch
- memory-intensive apps
- GPU workloads

```text
AKS
 |
 +-- System Node Pool
 |     +-- CoreDNS
 |     +-- CSI
 |     +-- system add-ons
 |
 +-- User Node Pool
       +-- frontend
       +-- payments
       +-- orders
       +-- batch
```

A common production design is:

```text
System Pool
  3 nodes
      |
      +-- system components

General App Pool
  autoscale 2 -> 20
      |
      +-- regular workloads

Memory Pool
  autoscale 0 -> 10
      |
      +-- memory-heavy workloads

GPU Pool
  autoscale 0 -> 5
      |
      +-- AI/ML workloads
```

---

# 8. VMSS-Based Worker Nodes vs Virtual Machines-Based Node Pools

Both models still provide Azure VMs that become Kubernetes worker nodes.

The major difference is **how AKS manages the collection of VMs**.

## VMSS-based pool

```text
AKS
 |
 v
Virtual Machine Scale Set
 |
 +-- VM1 -> Node1
 +-- VM2 -> Node2
 +-- VM3 -> Node3
```

Characteristics:

- mature and widely used AKS model
- nodes are managed through a VM Scale Set
- typically homogeneous VM configuration within the pool
- scaling changes VMSS instance count
- different VM requirements commonly result in additional pools

## Virtual Machines-based pool

```text
AKS
 |
 +-- VM1 -> Node1
 +-- VM2 -> Node2
 +-- VM3 -> Node3
```

AKS manages individual VMs more directly rather than one VMSS model.

This provides more flexibility around VM selection and scale-profile behavior.

### Interview shortcut

> **VMSS node pools optimize for homogeneous scale-set management; Virtual Machines node pools provide more per-VM flexibility.**

---

# 9. kubelet

kubelet runs on every worker node.

Its job is to make sure the pods assigned to that node are actually running.

```text
API Server
    |
    | Pod assigned to Node2
    v
kubelet
    |
    v
containerd
    |
    v
container
```

kubelet handles/coordinates:

- pod lifecycle
- container runtime requests
- probes
- volume mounts
- pod status
- node status
- CNI/CSI integration

---

# 10. Container Runtime — containerd

AKS Linux nodes use containerd.

```text
kubelet
   |
   | CRI
   v
containerd
   |
   +-- pull image
   +-- create container
   +-- start container
   +-- stop container
   +-- manage lifecycle
```

Memory line:

> **kubelet manages Pod lifecycle; containerd manages container lifecycle.**

---

# 11. CoreDNS

CoreDNS provides Kubernetes DNS and service discovery.

Example:

```text
frontend pod
     |
     | DNS: payments
     v
CoreDNS
     |
     v
payments.default.svc.cluster.local
     |
     v
Service ClusterIP
```

Applications generally use service names instead of hardcoding pod IP addresses.

---

# 12. Service Routing — kube-proxy vs Cilium

Traditional Kubernetes service path:

```text
Client Pod
   |
   v
Service ClusterIP
   |
   v
kube-proxy
   |
   v
iptables/IPVS rules
   |
   v
Backend Pod
```

With Azure CNI Powered by Cilium:

```text
Client Pod
   |
   v
Service
   |
   v
Cilium eBPF
   |
   v
Backend Pod
```

In this mode Cilium handles Kubernetes service routing, so kube-proxy is not used for the normal service data plane.

---

# 13. Azure CNI — What It Actually Means

It is useful to separate:

```text
IPAM / Azure network integration
              vs
Service-routing / policy data plane
```

Azure CNI primarily handles:

- pod networking
- IP allocation/IPAM
- Azure VNet integration

Cilium or Calico can provide additional data-plane/network-policy functionality.

---

# 14. Azure CNI Overlay

Typical example:

```text
VNet: 10.20.0.0/16

Node subnet:
10.20.10.0/24

Node1 = 10.20.10.4
Node2 = 10.20.10.5
Node3 = 10.20.10.6

Pod CIDR:
10.244.0.0/16
```

Nodes use real VNet addresses.

Pods use addresses from a separate overlay CIDR.

```text
Node1
10.20.10.4
   |
   +-- Pod 10.244.0.5
   +-- Pod 10.244.0.6

Node2
10.20.10.5
   |
   +-- Pod 10.244.1.5
   +-- Pod 10.244.1.6
```

Main benefit:

> Pods do not consume one VNet IP each.

---

# 15. Who Creates the Pod CIDR?

The overall pod CIDR is a cluster networking setting.

You can explicitly define it during AKS creation.

Example:

```bash
az aks create \
  --network-plugin azure \
  --network-plugin-mode overlay \
  --pod-cidr 10.244.0.0/16
```

Conceptually:

```text
Pod CIDR
10.244.0.0/16
      |
      v
AKS / Azure CNI IPAM
      |
      +-- Node1 receives pod range
      +-- Node2 receives pod range
      +-- Node3 receives pod range
```

The important design rule is:

> Pod CIDR must not overlap networks the cluster needs to reach.

Avoid:

```text
On-prem       10.244.0.0/16
Pod CIDR      10.244.0.0/16
```

Prefer separated address spaces such as:

```text
VNet          10.20.0.0/16
On-prem       10.50.0.0/16
Pod CIDR      172.20.0.0/16
Service CIDR  172.30.0.0/16
```

---

# 16. Pod Egress in Azure CNI Overlay

When an overlay pod accesses something outside the overlay network, its source is typically translated to a node/VNet address.

```text
Pod
10.244.1.7
    |
    v
Node
10.20.10.5
    |
    | SNAT
    v
VNet / external destination
```

So external systems commonly see a node or configured egress address rather than the overlay pod IP.

---

# 17. Azure CNI Powered by Cilium

Azure CNI does not disappear when Cilium is selected.

Think of it as:

```text
+-------------------------------------+
| Azure CNI                           |
|                                     |
| IP allocation / IPAM                |
| Azure networking integration        |
+------------------+------------------+
                   |
                   v
+-------------------------------------+
| Cilium                              |
|                                     |
| eBPF data plane                     |
| Service routing                     |
| NetworkPolicy                       |
| load balancing                      |
| observability capabilities          |
+-------------------------------------+
```

Key interview point:

> **Azure CNI handles the addressing/Azure integration; Cilium can provide the eBPF data plane.**

---

# 18. Azure CNI + Cilium vs Azure CNI + Calico

| Area | Azure CNI + traditional data plane | Azure CNI + Cilium | Azure CNI + Calico |
|---|---|---|---|
| Pod IP/IPAM | Azure CNI | Azure CNI | Azure CNI |
| Azure VNet integration | Yes | Yes | Yes |
| Service routing | kube-proxy | Cilium/eBPF | kube-proxy |
| Network policy | separate engine | Cilium | Calico |
| Main service dataplane | iptables/IPVS | eBPF | iptables/IPVS |
| kube-proxy | Yes | No for service routing | Yes |
| Strong point | familiar/mature | performance, scale, richer dataplane | mature policy model/ecosystem |
| Typical managed-AKS role | baseline | modern Linux dataplane | policy engine |

### Do not say

> We choose Azure CNI or Cilium.

### Better answer

> Azure CNI is responsible for AKS IPAM and Azure network integration. Cilium can be used as the data plane on top of Azure CNI, replacing kube-proxy service routing with eBPF and also enforcing network policy.

Likewise, Calico in managed AKS commonly works alongside Azure CNI as the network-policy engine.

---

# 19. Public AKS Cluster — Control-Plane Connectivity

A public AKS cluster exposes the Kubernetes API server using a public endpoint/FQDN.

```text
Azure-managed control plane

+----------------------+
| API Server           |
| Public endpoint      |
+----------+-----------+
           ^
           |
           | HTTPS/TLS
           |
============================

Customer VNet
     |
Worker Node
     |
   kubelet
```

Important nuance:

The endpoint is public, but it is better not to oversimplify the physical packet path as "plain open Internet".

The key architectural fact is:

> The node reaches a publicly addressed API endpoint secured by TLS/authentication.

API Server Authorized IP Ranges can restrict allowed source ranges.

---

# 20. Private AKS Cluster

Traditional private AKS exposes the API server privately using Azure Private Link / private endpoint and private DNS.

```text
Azure-managed control plane
       |
       v
    API Server
       |
       v
Azure Private Link
       |
============================
       |
Private Endpoint
       |
Customer VNet
       |
Worker Node
```

The API server is not exposed as the normal public cluster endpoint for cluster access.

---

# 21. Where Konnectivity Fits

This is a frequent interview trap.

The clean memory model:

```text
Node -> API Server
      normal HTTPS API communication

API Server -> Node/kubelet
      Konnectivity tunnel
      in traditional AKS networking
```

## Node to API Server

kubelet must:

- register the node
- update node status
- update pod status
- watch pod assignments
- communicate with Kubernetes APIs

```text
kubelet
   |
   | HTTPS
   v
API Server
```

## Control Plane to Node

Operations such as:

- kubectl logs
- kubectl exec
- kubectl port-forward
- API-server calls requiring kubelet

can require control-plane-to-kubelet communication.

Instead of relying on a simple inbound connection from the managed control plane into the node subnet, traditional AKS uses Konnectivity.

```text
CONTROL PLANE

API Server
    |
    v
Konnectivity Server
    |
    | secure mTLS tunnel
    |
    v
Konnectivity Agent
    |
    v
kubelet :10250

WORKER NODE
```

Example:

```text
kubectl logs
     |
     v
API Server
     |
     v
Konnectivity Server
     |
     v
secure tunnel
     |
     v
Konnectivity Agent
     |
     v
kubelet
     |
     v
container logs
```

### Interview answer

> Konnectivity solves the reverse-connectivity problem. Nodes can initiate HTTPS requests to the API server, while traditional AKS uses an mTLS Konnectivity tunnel when the managed control plane needs to communicate with kubelet on the worker node.

---

# 22. API Server VNet Integration

API Server VNet Integration changes the traditional model.

A dedicated subnet in your VNet is delegated for AKS API-server integration.

```text
Your VNet

+------------------------------------------------+
| API-server delegated subnet                    |
|                                                |
|      API Server private ILB/VIP                |
|              |                                 |
|              | VNet connectivity               |
|              |                                 |
| Node subnet  |                                 |
|      |       |                                 |
|      v       v                                 |
|   Worker Nodes                                 |
+------------------------------------------------+
```

This provides direct private VNet connectivity between the integrated API-server endpoint and node network.

For normal API-server-to-node connectivity in this architecture, the traditional Konnectivity tunnel is no longer required in the same way.

---

# 23. What Is a Delegated Subnet?

A delegated subnet is a normal Azure subnet that you explicitly assign to an Azure service.

Delegation tells Azure:

> This service is allowed to place/manage its service-specific networking resources in this subnet.

For AKS API Server VNet Integration the subnet is delegated to the AKS managed-cluster service.

```text
VNet 10.10.0.0/16
 |
 +-- api-server-subnet
 |     10.10.0.0/28
 |     delegated to AKS service
 |
 +-- node-subnet
       10.10.1.0/24
       worker nodes
```

Think:

```text
Normal subnet
  -> general customer resources

Delegated subnet
  -> reserved/authorized for a specific Azure service
```

Important:

The AKS control-plane VMs are still Microsoft managed.  
Delegation does **not** mean the control-plane VMs become customer VMs.

The API-server networking endpoint is integrated/projected into your VNet.

---

# 24. Worker Node <-> Control Plane Communication Summary

## Traditional public cluster

```text
Node
  |
  | HTTPS
  v
Public API endpoint

Reverse:
API Server
  |
  v
Konnectivity
  |
  v
kubelet
```

## Traditional private cluster

```text
Node
  |
  v
Private Endpoint / Private Link
  |
  v
API Server

Reverse:
API Server
  |
  v
Konnectivity
  |
  v
kubelet
```

## API Server VNet Integration

```text
API Server delegated subnet
          ^
          |
    direct VNet path
          |
          v
Node subnet
```

---

# 25. How to Detect Node-to-Control-Plane Latency

First separate:

```text
Network latency
       vs
API-server processing latency
```

Symptoms may include:

- slow kubectl commands
- kubectl exec/logs timeouts
- delayed node status
- intermittent NotReady
- API request timeout
- slow deployments/controllers

## Step 1 — node health

```bash
kubectl get nodes
kubectl describe node <node>
```

Check:

- Ready
- MemoryPressure
- DiskPressure
- PIDPressure
- NetworkUnavailable

## Step 2 — Konnectivity health

```bash
kubectl get pods -n kube-system -o wide | grep konnectivity
```

Inspect agent logs if required:

```bash
kubectl logs -n kube-system <konnectivity-agent-pod>
```

Look for:

- timeouts
- connection reset
- EOF
- repeated reconnects

## Step 3 — measure API endpoint timing

From an appropriate node/network context:

```bash
curl -k -o /dev/null -s \
  -w 'TCP=%{time_connect} TLS=%{time_appconnect} TTFB=%{time_starttransfer} TOTAL=%{time_total}\n' \
  https://<AKS-API-FQDN>/readyz
```

Interpretation:

```text
High TCP connect
  -> route / NSG / firewall / network issue

High TLS
  -> TLS/network negotiation issue

Fast TCP/TLS but high TTFB
  -> API server / admission / etcd / load issue
```

Do not rely only on ping; ICMP may be blocked while HTTPS is healthy.

## Step 4 — control-plane metrics

Useful API-server metrics include:

- apiserver_request_duration_seconds
- apiserver_request_total
- apiserver_current_inflight_requests

Example reasoning:

```text
Node -> API TCP = 8 ms
API request total = 900 ms

Likely not raw network latency.
Investigate API load, admission webhooks,
etcd pressure, throttling/429s, etc.
```

---

# 26. Azure Load Balancer -> Node -> Pod Traffic Flow

For a Kubernetes Service of type LoadBalancer:

```text
Internet
   |
   v
Azure Public IP
   |
   v
Azure Load Balancer
   |
   | backend pool
   v
Worker Node
   |
   v
kube-proxy OR Cilium
   |
   v
Service endpoint / Pod
```

Important:

> The Azure Load Balancer backend pool is typically the worker-node layer. It does not need to understand normal pod-overlay routing itself.

Example:

```text
Client
100.10.10.10
     |
     v
Azure LB
20.30.40.50:443
     |
     v
Node2
10.20.10.5
     |
     v
kube-proxy / Cilium
     |
     v
Pod
10.244.1.7:8443
```

---

# 27. Cross-Node Service Hop

The node selected by the Azure Load Balancer does not necessarily host the selected application pod.

Possible flow:

```text
Azure LB
   |
   v
Node2
   |
   | service routing
   v
Node1
   |
   v
Pod
```

That extra node hop is important when discussing source IP preservation and traffic efficiency.

---

# 28. externalTrafficPolicy

## Cluster

```yaml
externalTrafficPolicy: Cluster
```

Traffic can arrive at one node and be forwarded to a matching pod on another node.

```text
LB -> Node2 -> Node1 -> Pod
```

## Local

```yaml
externalTrafficPolicy: Local
```

Traffic is accepted only on nodes that have a local eligible endpoint for that Service.

```text
LB
 |
 +--> Node1 -> local Pod
 +--> Node3 -> local Pod

Node2 has no local endpoint
```

This can avoid cross-node forwarding and is useful when preserving client source IP is required.

---

# 29. Azure LB + In-Cluster Ingress

If NGINX/Envoy/Gateway runs inside AKS:

```text
Internet
   |
   v
Azure Load Balancer
   |
   v
Worker Node
   |
   v
kube-proxy / Cilium
   |
   v
Ingress/Gateway Pod
   |
   | host/path routing
   v
Kubernetes Service
   |
   v
Application Pod
```

This is a very common whiteboard packet flow.

---

# 30. Azure Application Gateway

Application Gateway is a Layer-7 HTTP/HTTPS load balancer and can provide:

- host/path routing
- TLS termination
- WAF

For an AKS cluster using an **in-cluster ingress controller** such as NGINX or Istio, remember the logical flow as:

```text
Internet
   |
   v
Application Gateway / WAF
   |
   v
Ingress Controller
   |
   v
Kubernetes Service
   |
   v
Application Pods
```

In practice, the ingress controller normally needs a private reachable frontend. A common implementation is:

```text
Application Gateway
        |
        v
Internal Load Balancer
(private frontend for ingress)
        |
        v
Ingress Controller Pod
        |
        v
Service
        |
        v
Application Pods
```

The **Internal Load Balancer is not mandatory in every Application Gateway design**. It is simply one common way to expose an in-cluster ingress controller privately.

### AGIC exception

When using **Application Gateway Ingress Controller (AGIC)**, Application Gateway itself acts as the AKS ingress data plane and can use pod private IPs directly:

```text
Internet
   |
   v
Application Gateway / WAF
   |
   v
Application Pods
```

AGIC watches Kubernetes Ingress/Service/endpoint state and programs Application Gateway accordingly, so an extra internal Load Balancer, NodePort, or kube-proxy hop isn't required for this path.

### Memory line

```text
In-cluster ingress:
App Gateway -> (private ILB) -> Ingress -> Service -> Pods

AGIC:
App Gateway -> Pods
```

Remember:

```text
Azure Load Balancer
  -> Layer 4

Application Gateway
  -> Layer 7 + WAF
```

---


# 30A. Backend Pools — Where They Fit

A **backend pool** is the set of destinations that an Azure Load Balancer or Application Gateway can send traffic to.

Think:

```text
Frontend IP / Listener
        |
        v
Load-balancing / routing rule
        |
        v
Backend Pool
        |
        v
Backend targets
```

The backend targets can be things such as:

- VM instances
- VM Scale Set instances
- private IPs
- FQDNs
- pod IPs in supported Application Gateway/AGIC designs

The important point is:

> **A VM or VMSS does not replace the backend pool. The VM/VMSS instances are members or targets of the backend pool.**

## Azure Standard Load Balancer

For AKS with an Azure Standard Load Balancer:

```text
Client
  |
  v
Azure Standard Load Balancer
  |
  v
Backend Pool
  |
  +--> AKS Node1
  +--> AKS Node2
  +--> AKS Node3
  |
  v
kube-proxy / Cilium
  |
  v
Pod
```

If the AKS node pool is VMSS-based:

```text
Azure Standard Load Balancer
        |
        v
Backend Pool
        |
        v
AKS VMSS instances
   +--> Node1
   +--> Node2
   +--> Node3
```

In AKS, when you create a Kubernetes Service of type `LoadBalancer`, AKS/Azure normally manages the load-balancer rule, probe, frontend and backend-pool membership for you.

So:

> **Backend pool is required architecturally, but you usually do not manually maintain it in AKS.**

## Application Gateway

Application Gateway also uses backend pools.

### In-cluster ingress pattern

```text
Application Gateway
        |
        v
Backend Pool
        |
        v
Private ingress frontend
(often an Internal LB IP)
        |
        v
Ingress Controller
        |
        v
Service
        |
        v
Pods
```

### AGIC pattern

```text
Application Gateway
        |
        v
Backend Pool
        |
        v
Pod IPs
```

With AGIC, the controller updates Application Gateway based on Kubernetes state, so pod IPs can become backend targets directly.

## Memory line

```text
Backend Pool
    =
the list/group of destinations
the Azure load-balancing service
is allowed to send traffic to.

VM / VMSS / IP / Pod
    =
members of that backend pool.
```

---

# 31. ACR Integration

Azure Container Registry stores container images.

```text
CI/CD
  |
  | push
  v
ACR
  |
  | pull
  v
AKS Node
  |
kubelet
  |
containerd
  |
  v
Pod
```

Do not confuse the identities.

```text
Cluster/control-plane identity
  -> AKS management of Azure resources

Kubelet identity
  -> node-level Azure access
  -> common example: ACR image pull

Workload Identity
  -> Pod/application access to Azure services
```

---

# 32. Managed Identities

A useful mental model:

```text
CONTROL-PLANE MANAGED IDENTITY
AKS control plane -> Azure resources

KUBELET MANAGED IDENTITY
Worker/kubelet -> Azure resources
Example: pull image from ACR

WORKLOAD IDENTITY
Pod -> Microsoft Entra ID -> Azure service
Example: Key Vault
```

The identities solve different problems and should not be treated as one generic "AKS identity".

---

# 33. Key Vault Integration

Preferred pattern:

```text
Pod
 |
 | Kubernetes ServiceAccount
 v
Workload Identity / OIDC
 |
 v
Microsoft Entra ID
 |
 v
Azure Key Vault
 |
 v
Secret / key / certificate
```

Another common integration is the Secrets Store CSI Driver with Azure Key Vault provider.

```text
Azure Key Vault
      |
      v
Secrets Store CSI Driver
      |
      v
mounted secret
      |
      v
Pod
```

Avoid embedding long-lived Azure credentials directly in application manifests.

---

# 34. Azure Monitor / Container Insights

Separate telemetry into metrics, logs and application traces.

```text
AKS
 |
 +--> Metrics
 |      |
 |      v
 |   Managed Prometheus
 |      |
 |      v
 |   Azure Managed Grafana
 |
 +--> Logs
 |      |
 |      v
 |   Container Insights
 |      |
 |      v
 |   Log Analytics
 |
 +--> Application telemetry
        |
        v
    Application Insights / OpenTelemetry
```

Typical log/telemetry areas:

- container stdout/stderr
- Kubernetes events
- node/container inventory
- control-plane diagnostics
- application traces
- Prometheus metrics

---

# 35. Full Deployment Flow

```text
GitLab / Azure DevOps / Argo CD
            |
            v
       API Server
            |
            v
           etcd
            |
            v
 Controller Manager
            |
            v
        Scheduler
            |
            v
       chosen Node
            |
          kubelet
            |
            +--> ACR image pull
            |
            v
        containerd
            |
            v
           Pod
```

---

# 36. Full North-South Traffic Flow

A classic AKS ingress flow:

```text
Client
  |
  v
Azure Public IP
  |
  v
Azure Load Balancer / Application Gateway
  |
  v
Worker Node
  |
  v
Ingress/Gateway
  |
  v
Kubernetes Service
  |
  v
kube-proxy / Cilium
  |
  v
Application Pod
```

If Azure CNI Powered by Cilium is used, Cilium performs the service-routing dataplane rather than kube-proxy.

---

# 37. Interview Follow-Up Questions and Answers

## Q1. Is the AKS control plane inside my subscription?

**Answer:** The control plane is Microsoft managed. You interact with it through the API-server endpoint. With API Server VNet Integration, its networking endpoint is integrated into your VNet, but that does not mean you manage the control-plane VMs.

---

## Q2. Does the scheduler start the container?

**Answer:** No. The scheduler selects a suitable node. kubelet on that node uses the container runtime to create/start the containers.

---

## Q3. Does Azure CNI Powered by Cilium mean Azure CNI is removed?

**Answer:** No. Azure CNI still handles Azure networking/IPAM. Cilium supplies the eBPF-based data plane, service routing and network-policy enforcement.

---

## Q4. Is Calico a replacement for Azure CNI?

**Answer:** In common managed-AKS designs, no. Azure CNI can handle IPAM/network integration while Calico is used as the network-policy engine.

---

## Q5. What is the simplest difference between Calico and Cilium in AKS?

**Answer:** Calico is commonly used primarily for network policy alongside Azure CNI and kube-proxy. Cilium can be the eBPF data plane, handling both service routing and network policy and removing kube-proxy from the normal service-routing path.

---

## Q6. Who assigns pod IPs with Azure CNI Overlay?

**Answer:** Azure CNI/IPAM allocates addresses from the configured cluster pod CIDR and assigns pod ranges to nodes.

---

## Q7. Does every pod consume a VNet IP in Overlay mode?

**Answer:** No. Nodes consume VNet addresses. Pods use addresses from the separate overlay pod CIDR.

---

## Q8. Who defines the pod CIDR?

**Answer:** It is part of cluster network planning. You can explicitly define it at cluster creation. The CIDR must not overlap VNet, peered, on-prem or other networks the cluster needs to reach.

---

## Q9. When a pod in Overlay mode talks to an external VNet resource, what source IP is seen?

**Answer:** Overlay pod traffic leaving the overlay is typically source-NATed so the destination sees a node/configured egress address rather than the overlay pod IP.

---

## Q10. How does a worker node communicate with the control plane?

**Answer:** kubelet and node components communicate with the Kubernetes API server using secured HTTPS API connections. The exact network path depends on whether the cluster uses a public API endpoint, Private Link/private endpoint, or API Server VNet Integration.

---

## Q11. Where does Konnectivity fit?

**Answer:** In traditional AKS networking, Konnectivity provides secure reverse connectivity from the managed control plane to kubelet on worker nodes. It is especially relevant to operations such as logs, exec and port-forward.

---

## Q12. Does a private cluster automatically mean API Server VNet Integration?

**Answer:** No. Traditional private AKS can use Private Link/private endpoints. API Server VNet Integration is a separate control-plane networking model.

---

## Q13. What is a delegated subnet?

**Answer:** A normal VNet subnet explicitly assigned to an Azure service so that service is allowed to create/manage its required networking resources there. AKS API Server VNet Integration uses a subnet delegated to the AKS managed-cluster service.

---

## Q14. Is the API-server VM actually deployed as my VM into that delegated subnet?

**Answer:** No. The control plane remains Microsoft managed. The API-server networking endpoint is integrated/projected into the delegated subnet.

---

## Q15. How would you detect latency between a node and the control plane?

**Answer:** Check node conditions, Konnectivity health where applicable, measure API endpoint TCP/TLS/TTFB timings, and inspect control-plane metrics such as API request duration. Separate raw network latency from slow API processing.

---

## Q16. How does Azure Load Balancer reach a pod?

**Answer:** Normally the load balancer sends traffic to worker-node backends. Kubernetes service routing on the node, implemented by kube-proxy or Cilium depending on the dataplane, forwards the request to an eligible pod endpoint.

---

## Q17. Can the selected pod be on another node?

**Answer:** Yes. With Cluster traffic policy, LB -> Node2 -> Node1 -> Pod is possible.

---

## Q18. What does externalTrafficPolicy: Local change?

**Answer:** It restricts external service traffic to nodes with local endpoints, reducing cross-node forwarding and commonly helping preserve source client IP.

---

## Q19. What is the difference between Azure Load Balancer and Application Gateway?

**Answer:** Azure Load Balancer is primarily Layer 4 TCP/UDP load balancing. Application Gateway is Layer 7 HTTP/HTTPS routing and can provide TLS termination and WAF. With an in-cluster ingress controller, Application Gateway can forward to a private ingress frontend; with AGIC, Application Gateway can route directly to pod private IPs.

---

## Q20. Which identity usually pulls from ACR?

**Answer:** The kubelet/node identity is commonly used for ACR pull permissions. Do not confuse this with the control-plane identity or pod Workload Identity.

---


## Q21. What is the difference between AKS Automatic and AKS Standard?

**Answer:** Both are AKS Kubernetes clusters. Automatic is the opinionated, highly managed operating model: Azure preconfigures and operates more of the node management, scaling, security, monitoring, networking and upgrade lifecycle. Standard gives the platform team granular control over those choices.

---

## Q22. Does AKS Automatic mean I no longer have Kubernetes worker nodes?

**Answer:** No. Workloads still run on Kubernetes worker compute. The difference is that Azure manages node provisioning and scaling much more aggressively through the Automatic operating model instead of requiring you to manually design and maintain normal user node pools.

---

## Q23. What networking should I associate with AKS Automatic?

**Answer:** The production baseline uses Azure CNI Overlay powered by Cilium, with more networking defaults managed by AKS. AKS Standard gives you a broader set of explicit networking choices.

---

## Q24. When would you choose AKS Standard instead of Automatic?

**Answer:** Use Standard when you need granular infrastructure control such as custom node-pool topology, Windows nodes, specific VM requirements, custom networking/routing, explicit upgrade control, or existing platform automation that depends on direct cluster lifecycle management.

---


## Q25. Is a backend pool required for Azure Standard Load Balancer?

**Answer:** Yes. A backend pool is a core part of Azure Standard Load Balancer. In AKS, the backend pool normally contains the worker-node instances, and AKS/Azure manages that membership automatically.

---

## Q26. If my AKS workers are in a VMSS, do I still need a backend pool?

**Answer:** Yes. The VMSS does not replace the backend pool. The VMSS instances are the backend targets/members that the load balancer sends traffic to.

---

## Q27. Can I use individual VMs instead of a VMSS in a backend pool?

**Answer:** Yes. Azure load-balancing services can use individual VM/IP targets as backend members. The backend pool is still required; only the type of backend target changes.

---

## Q28. Where does the backend pool fit with Application Gateway and AKS?

**Answer:** Application Gateway sends traffic to a configured backend pool. With an in-cluster ingress design, the backend target can be the private ingress frontend such as an Internal Load Balancer IP. With AGIC, the backend pool can be maintained with pod IPs directly.

---

# 38. 60-Second AKS Architecture Pitch

> AKS separates an Azure-managed Kubernetes control plane from customer-managed worker node pools. The control plane contains the API server, etcd, scheduler and controllers. When a deployment is submitted, the API server validates and persists desired state, controllers reconcile it, and the scheduler selects a worker node. kubelet on that node uses containerd to run containers. Azure CNI provides pod networking and Azure integration; on modern Linux clusters Cilium can provide an eBPF service and policy data plane, while Calico is commonly used as a policy engine with traditional kube-proxy routing. CoreDNS provides service discovery. External traffic commonly enters through Azure Load Balancer or Application Gateway, reaches the node/ingress layer, then a Kubernetes Service and finally the pod. Images come from ACR, workloads can use Workload Identity and Key Vault, and Azure Monitor/Prometheus/Container Insights provide telemetry.

---

# 39. Whiteboard Diagram to Remember

```text
                           AKS CONTROL PLANE
                  +------------------------------+
                  | API Server                   |
                  | etcd                         |
                  | Scheduler                    |
                  | Controllers                  |
                  +---------------+--------------+
                                  |
                       API / control traffic
                                  |
         +------------------------+----------------------+
         |                                               |
         v                                               v
 +-------------------+                         +-------------------+
 | System Node Pool  |                         | User Node Pool    |
 |                   |                         |                   |
 | kubelet           |                         | kubelet           |
 | containerd        |                         | containerd        |
 | Azure CNI         |                         | Azure CNI         |
 | Cilium/kube-proxy |                         | Cilium/kube-proxy |
 | CoreDNS/add-ons   |                         | App Pods          |
 +---------+---------+                         +---------+---------+
           |                                             |
           +---------------- Azure VNet -----------------+
                                  ^
                                  |
                      Azure LB / App Gateway
                                  ^
                                  |
                               Internet

Supporting services:
ACR | Key Vault | Managed Identity / Workload Identity | Azure Monitor
```

---

# 40. Official Microsoft References

- AKS core concepts: https://learn.microsoft.com/azure/aks/core-aks-concepts
- AKS networking concepts: https://learn.microsoft.com/azure/aks/concepts-network
- Azure CNI Overlay: https://learn.microsoft.com/azure/aks/azure-cni-overlay
- Azure CNI Powered by Cilium: https://learn.microsoft.com/azure/aks/azure-cni-powered-by-cilium
- AKS network policy: https://learn.microsoft.com/azure/aks/use-network-policies
- Private AKS clusters: https://learn.microsoft.com/azure/aks/private-clusters
- API Server VNet Integration: https://learn.microsoft.com/azure/aks/api-server-vnet-integration
- AKS control-plane networking: https://learn.microsoft.com/azure/aks/plan-control-plane-networking
- System node pools: https://learn.microsoft.com/azure/aks/use-system-pools
- Managed identities: https://learn.microsoft.com/azure/aks/managed-identity-overview
- Workload Identity: https://learn.microsoft.com/azure/aks/workload-identity-overview
- Key Vault CSI integration: https://learn.microsoft.com/azure/aks/csi-secrets-store-driver
- Container Insights: https://learn.microsoft.com/azure/azure-monitor/containers/container-insights-overview

---

## Revision Rule

When new AKS/Azure interview questions are discussed, add the answer into the relevant technical section **and** add a concise version to the follow-up Q&A section so this document stays both explanatory and interview-revision friendly.
