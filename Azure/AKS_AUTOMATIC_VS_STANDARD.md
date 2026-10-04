# AKS Automatic vs AKS Standard

## Overview

AKS has two main operating models:

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

Both are still AKS and both use the same Kubernetes concepts.

The key difference is:

> **AKS Automatic lets Azure operate more of the platform for you. AKS Standard gives the platform team more direct control over the platform.**

---

## AKS Automatic

AKS Automatic is the more opinionated, production-ready operating model.

Azure preconfigures and manages more of the platform, including:

- node provisioning
- node scaling
- system-node management
- cluster upgrades
- node-image upgrades
- security defaults
- networking baseline
- monitoring defaults
- OIDC issuer / Workload Identity
- workload scaling capabilities

Conceptually:

```text
Application workload
       |
       v
Kubernetes Scheduler
       |
       v
Node Auto Provisioning
       |
       v
AKS selects/provisions
appropriate compute
       |
       v
Pod scheduled
```

The platform team spends less time building and maintaining node-pool topology.

---

## AKS Automatic Networking

A key networking baseline is:

```text
AKS Automatic
      |
      v
Azure CNI Overlay
      +
Cilium
      |
      v
eBPF-based service routing
and network policy
```

So many networking choices that are explicit in Standard are already selected as production defaults in Automatic.

---

## AKS Automatic Upgrades

The lifecycle is more automated:

```text
Azure-managed lifecycle
       |
       +--> Kubernetes upgrade channel
       |
       +--> Node-image upgrade channel
       |
       +--> managed maintenance behavior
```

This reduces manual lifecycle work.

---

# AKS Standard

AKS Standard provides the traditional, highly configurable AKS model.

Azure still manages the Kubernetes control plane, but the platform team directly controls more of:

- node-pool design
- VM SKU
- system vs user pools
- autoscaling
- networking
- CNI/data plane
- ingress
- egress
- maintenance windows
- Kubernetes version upgrade strategy
- node-image upgrade strategy
- monitoring
- security controls
- public/private API exposure
- API Server VNet Integration

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
     |
     v
AKS Standard
```

---

# Side-by-Side Comparison

| Area | AKS Automatic | AKS Standard |
|---|---|---|
| Operating model | Opinionated / highly managed | Granular operator control |
| Control plane | Azure managed | Azure managed |
| Node management | Mostly Azure managed | Platform team manages |
| Node provisioning | Automatic / workload driven | Explicit node pools / autoscaling |
| Networking | Opinionated defaults | Broad configuration choice |
| Common modern dataplane | Azure CNI Overlay + Cilium | Supported choices configured by operator |
| Security | Hardened defaults | Explicit configuration |
| Monitoring | Preconfigured/defaulted | Explicitly enabled/configured |
| Upgrades | Automated baseline | Manual or configured auto channels |
| VM/node topology | Less granular | Greater control |
| Operational effort | Lower | Higher |
| Flexibility | Lower | Higher |

---

# How This Relates to Node Pools

## Standard

```text
Platform Team
   |
   +--> System Pool
   +--> General Pool
   +--> GPU Pool
   +--> Memory Pool
   +--> Spot Pool
```

You define and operate those pools.

## Automatic

```text
Workload requests
      |
      v
Node Auto Provisioning
      |
      v
AKS provisions
appropriate compute
```

The infrastructure is more workload-driven.

---

# How This Relates to Networking

## Standard

You may explicitly choose/design:

```text
Azure CNI Overlay
Azure CNI Pod Subnet
Cilium
Calico
custom VNet
private cluster
API Server VNet Integration
custom ingress
custom egress
```

## Automatic

You start with a more opinionated baseline:

```text
Azure CNI Overlay
       +
Cilium
       +
managed networking defaults
```

---

# How This Relates to Upgrades

## Standard

```text
Platform Team
   |
   +--> choose Kubernetes target version
   +--> control-plane upgrade
   +--> node-pool upgrade
   +--> node-image strategy
   +--> maintenance window
```

## Automatic

```text
Azure-managed upgrade channels
       |
       +--> Kubernetes version lifecycle
       +--> node-image lifecycle
       |
       v
less manual intervention
```

---

# When to Use AKS Automatic

AKS Automatic is a good fit when:

- you are building a new production platform
- you want strong defaults
- you want lower operational overhead
- application teams should not manage node infrastructure
- standard Linux workloads fit the supported model
- you want Azure to automate more scaling, security and upgrades

---

# When to Use AKS Standard

AKS Standard is a better fit when:

- you need custom node-pool topology
- you need specific VM SKUs
- you need Windows node pools
- you require custom networking or routing
- you need strict change control
- you need explicit maintenance windows
- you already have mature AKS automation
- your enterprise architecture needs granular lifecycle control

Example:

```text
Hub / Spoke
    |
Azure Firewall
    |
Custom UDR
    |
Private AKS
    |
API Server VNet Integration
    |
Dedicated System Pool
    |
General Pool
    |
GPU Pool
    |
Spot Pool
    |
Custom upgrade waves
```

This type of environment generally benefits from AKS Standard.

---

# Important Clarification

AKS Automatic does **not** mean your application becomes fully managed.

You still manage:

- Deployments
- StatefulSets
- Services
- ConfigMaps
- application configuration
- container images
- CI/CD or GitOps
- application SLOs
- application security
- release strategy

Think:

```text
AKS Automatic
   =
Azure operates more of the platform

AKS Standard
   =
Platform team operates more of the platform
```

---

# Interview Questions

## Q1. What is the difference between AKS Automatic and AKS Standard?

**Answer:** Both are managed AKS Kubernetes clusters. Automatic is an opinionated, highly managed operating model where Azure handles more node provisioning, scaling, security, networking, monitoring and upgrades. Standard gives the platform team more direct control over node pools, networking, scaling and lifecycle management.

## Q2. Does AKS Automatic mean there are no worker nodes?

**Answer:** No. Workloads still run on Kubernetes worker compute. The difference is that AKS manages node provisioning and scaling much more automatically.

## Q3. What networking should I associate with AKS Automatic?

**Answer:** Azure CNI Overlay with Cilium is the key production networking baseline to remember.

## Q4. When would you choose Standard instead of Automatic?

**Answer:** When you need granular infrastructure control: custom networking, Windows nodes, specific VM requirements, custom node pools, explicit maintenance/upgrade control, or integration with existing platform automation.

---

# 60-Second Interview Answer

> AKS Automatic and AKS Standard use the same Kubernetes fundamentals, but they differ in operational ownership. Automatic is an opinionated production-ready mode where Azure manages much more of node provisioning, scaling, security, networking, monitoring and upgrades. Standard still has an Azure-managed control plane, but the platform team gets more direct control over node pools, VM choices, networking, autoscaling, ingress and lifecycle operations. I would use Automatic when the workload fits the standard production model and Standard when enterprise infrastructure, networking or change-control requirements need granular control.

---

# Memory Line

> **Automatic = Azure operates more of the platform.**  
> **Standard = the platform team controls more of the platform.**
