# OpenShift ACM and Hub-and-Spoke — Interview Consolidated Notes

## 1. What Hub-and-Spoke Means

In an enterprise OpenShift environment, hub-and-spoke usually means one central management cluster manages multiple independent Kubernetes/OpenShift clusters.

```text
                        ACM HUB
                 OpenShift Cluster
                       |
        --------------------------------
        |              |               |
        v              v               v
   Spoke Cluster 1 Spoke Cluster 2 Spoke Cluster 3
      Production       Production           DR
```

Each spoke is normally an independent cluster with its own:

- API server
- etcd
- scheduler/controllers
- worker nodes
- networking
- storage
- ingress
- applications

ACM does not merge these clusters into one large Kubernetes cluster.

---

## 2. Multi-AZ vs Hub-and-Spoke

This is an important interview distinction.

### Multi-AZ

One OpenShift cluster spans multiple failure domains:

```text
              ONE OPENSHIFT CLUSTER
                       |
          ---------------------------
          |            |            |
         AZ-1         AZ-2         AZ-3
        workers      workers      workers
```

This is for availability within one cluster.

### Hub-and-Spoke

Multiple independent clusters are centrally managed:

```text
                 ACM HUB
                    |
        -------------------------
        |           |           |
        v           v           v
     Cluster 1   Cluster 2   Cluster 3
```

This is for centralized fleet management.

Interview answer:

> Worker nodes across AZs in one OpenShift cluster are not hub-and-spoke. That is a multi-AZ cluster. Hub-and-spoke is generally used when a central ACM hub manages multiple independent OpenShift/Kubernetes clusters.

---

## 3. What is Red Hat Advanced Cluster Management (ACM)?

Red Hat Advanced Cluster Management for Kubernetes provides centralized management for multiple Kubernetes/OpenShift clusters.

The simplest interview explanation:

> ACM provides a hub-and-spoke architecture where the hub centrally manages cluster lifecycle, governance, placement, visibility and GitOps integration across multiple managed clusters.

Core areas:

```text
                    ACM
                     |
        --------------------------------
        |              |               |
   Cluster Lifecycle Governance   Fleet Visibility
                                         |
                                    GitOps integration
```

---

## 4. Hub Cluster

The hub is an OpenShift cluster where ACM is installed.

Typical logical components:

```text
OpenShift Hub Cluster
       |
       +-- ACM Operator
       |
       +-- MultiClusterHub
       |
       +-- Cluster lifecycle management
       |
       +-- Governance
       |
       +-- Placement
       |
       +-- Observability
       |
       +-- GitOps integration
```

The hub should itself be deployed as a production-grade HA OpenShift cluster.

---

## 5. Managed Cluster / Spoke

A spoke is normally referred to as a managed cluster.

Examples:

- On-prem OpenShift
- AWS OpenShift / ROSA
- Azure OpenShift
- OpenShift in another region/site
- Supported Kubernetes distributions

Conceptually:

```text
ACM Hub
 |
 +--- OpenShift Production Cluster
 |
 +--- OpenShift DR Cluster
 |
 +--- AWS Cluster
 |
 +--- Azure Cluster
```

Each remains operationally independent.

---

## 6. How a Cluster is Imported

An existing cluster can be imported into ACM.

High-level flow:

```text
ACM Hub
   |
   | Create/import managed cluster
   v
Managed Cluster
   |
   | ACM management agents installed
   v
Klusterlet
   |
   | Secure registration/communication
   v
ACM Hub
```

A key term to know is **Klusterlet**.

### Klusterlet

Klusterlet is the ACM management agent/component deployed on the managed cluster.

It enables the managed cluster to:

- register with the hub
- report cluster status
- receive work/policies
- participate in centralized management

Interview answer:

> When a cluster is imported into ACM, ACM deploys management agents such as Klusterlet on the managed cluster. These agents establish a secure relationship with the hub and allow centralized policy, placement, visibility and management.

---

## 7. Communication Model

Do not describe ACM as the hub SSHing to every node.

Think of the model as agent/API-based communication.

```text
Managed Cluster
     |
  Klusterlet
     |
     | HTTPS / secured API communication
     v
   ACM Hub
```

Typical enterprise dependencies:

- DNS
- routing
- certificates
- firewall rules
- Kubernetes/OpenShift API connectivity
- required ACM endpoints

This is particularly useful in controlled enterprise environments because the managed cluster can communicate to the hub without exposing arbitrary node-level administration.

---

## 8. What ACM Manages

Remember these core functions.

### A. Cluster Lifecycle

ACM can centrally help with:

- provisioning supported clusters
- importing existing clusters
- managing cluster inventory
- tracking cluster health
- managing supported lifecycle operations

### B. Governance

Governance is one of the most important ACM interview topics.

Example requirement:

> Every production cluster must have required security/network policies.

Without centralized governance, admins must validate every cluster individually.

With ACM:

```text
                 ACM Policy
                     |
             Production Standard
                     |
          -------------------------
          |           |           |
          v           v           v
       Cluster 1   Cluster 2   Cluster 3
       Compliant   Compliant   NonCompliant
```

Examples:

- RBAC requirements
- namespaces
- NetworkPolicies
- operator configuration
- security controls
- platform standards
- configuration policies

ACM can report whether clusters are compliant or non-compliant.

---

## 9. Placement

You usually do not want to apply every application or policy to every cluster.

Clusters can be labeled by:

- environment
- region
- business unit
- platform
- criticality
- workload type

Example:

```text
Policy / Application
        |
     Placement
        |
 environment=prod
        |
   ----------------
   |      |       |
   v      v       v
 Prod-1 Prod-2  Prod-3
```

Placement lets ACM target appropriate clusters.

Interview line:

> I would label clusters and use placement rules/placement resources so governance policies and GitOps workloads target only the intended clusters.

---

## 10. ACM and GitOps

ACM and Argo CD/OpenShift GitOps complement each other.

Think:

### ACM

- Which clusters exist?
- Which clusters are healthy?
- Which clusters are compliant?
- Which clusters should receive a policy/application?
- How do I manage the fleet?

### Argo CD / OpenShift GitOps

- What application/configuration should run?
- Does live state match Git?
- How do I continuously reconcile desired state?

Architecture:

```text
                Git Repository
                     |
                  Argo CD
                     |
                 ACM Placement
                     |
          -------------------------
          |           |           |
          v           v           v
       Cluster 1   Cluster 2   Cluster 3
```

Interview answer:

> I would use ACM for multi-cluster fleet management, governance and placement, and OpenShift GitOps/Argo CD for declarative application and configuration delivery.

---

## 11. ACM Observability

ACM provides centralized multi-cluster visibility.

Instead of manually logging in to many clusters:

```bash
oc login cluster1
oc login cluster2
oc login cluster3
```

you can view fleet-level health centrally.

Conceptually:

```text
Cluster 1 --Cluster 2 ---Cluster 3 ----> ACM Observability / Fleet View
Cluster 4 ---/
Cluster 5 --/
```

Important:

> ACM does not replace Prometheus/Grafana inside each OpenShift cluster.

It provides centralized multi-cluster visibility on top of the managed environments.

---

## 12. Does ACM Manage Worker Nodes Directly?

Not in the sense of acting as the node-level controller.

Think in layers:

```text
ACM
 |
 v
OpenShift Cluster
 |
 +-- Control Plane
 |
 +-- Machine API / Cluster Operators
 |
 +-- Worker Nodes
```

ACM primarily provides fleet-level cluster management and policy.

OpenShift itself manages the cluster internals.

---

## 13. What Happens if the ACM Hub Fails?

This is a common and important interview question.

```text
               ACM HUB
                  X
                 DOWN

         -------------------
         |        |        |
         v        v        v
      Cluster1 Cluster2 Cluster3
         OK       OK       OK
```

The managed clusters continue to run their existing workloads because they have their own:

- etcd
- API server
- scheduler/controllers
- workers
- networking
- storage
- applications

The main impact is loss or degradation of centralized management functionality such as:

- governance updates
- centralized visibility
- fleet operations
- policy propagation
- central management actions

Strong interview answer:

> ACM is a management plane, not the runtime control plane for the managed clusters. If the ACM hub is unavailable, applications already running on the spoke clusters continue running.

---

## 14. High Availability for the ACM Hub

The hub itself should be production grade.

Example:

```text
              ACM Hub OpenShift Cluster

         Master-1   Master-2   Master-3
            |          |          |
           AZ-1       AZ-2       AZ-3

               Worker Nodes
                    |
                   ACM
```

For cloud environments, spread control-plane and suitable worker capacity across failure domains.

For on-prem, use independent infrastructure/failure domains wherever possible.

---

## 15. Multiple ACM Hubs

Large enterprises may use more than one ACM hub.

Reasons include:

- geographic separation
- scale
- latency
- regulatory boundaries
- organizational separation
- network segmentation
- independent failure domains

Example:

```text
                  Enterprise

            -------------------
            |                 |
            v                 v
       ACM Hub APAC       ACM Hub EMEA
         |   |   |          |   |   |
        C1  C2  C3         C4  C5  C6
```

You do not need deep multi-hub internals unless the interviewer specifically asks.

---

## 16. ACM in Air-Gapped / Disconnected Environments

This is highly relevant to enterprise OpenShift.

```text
                     INTERNET
                        X
                        |
                 Disconnected Zone
                        |
                Internal Registry
                        |
                     ACM Hub
                        |
          ----------------------------
          |            |             |
          v            v             v
       OCP-1         OCP-2         OCP-3
```

High-level approach:

1. Mirror required OpenShift and ACM/operator content into an internal registry.
2. Configure disconnected catalog/image sources as required.
3. Install ACM from mirrored content.
4. Ensure internal DNS/routing/firewall connectivity.
5. Import or provision managed clusters.
6. Ensure managed clusters can reach required ACM hub endpoints internally.

This ties together:

- OpenShift
- mirror registry
- air-gapped installation
- operators
- ACM
- multi-cluster management

---

## 17. Practical Enterprise Scenario

### Question

You have 30 OpenShift clusters across multiple data centers and cloud regions. How would you manage them?

### Strong answer

> I would avoid managing 30 clusters independently and introduce Red Hat Advanced Cluster Management using a hub-and-spoke architecture. The ACM hub would provide centralized cluster lifecycle management, governance, placement and fleet visibility. Existing OpenShift clusters would be imported as managed clusters and ACM agents such as Klusterlet would run on them.
>
> I would label clusters based on environment, region, business unit and criticality, then use placement to target policies and workloads appropriately. ACM governance policies would enforce or monitor common platform and security standards. For application and configuration delivery, I would integrate ACM with OpenShift GitOps/Argo CD.
>
> The managed clusters remain independent. If the ACM hub becomes unavailable, applications on the managed clusters continue running. I would therefore make the hub itself highly available and ensure DNS, certificates, firewall rules and network connectivity between the hub and managed clusters are designed correctly.

---

## 18. Interview Questions and Answers

### Q1. What is ACM?

ACM is Red Hat Advanced Cluster Management for Kubernetes. It provides centralized lifecycle management, governance, placement, observability and GitOps integration across multiple Kubernetes/OpenShift clusters.

### Q2. Explain hub-and-spoke in ACM.

The hub is the central OpenShift cluster running ACM. Spokes are independent managed Kubernetes/OpenShift clusters registered to the hub.

### Q3. Are worker nodes across multiple AZs a hub-and-spoke design?

No. Workers across multiple AZs within one cluster are a multi-AZ design. Hub-and-spoke normally refers to centralized management of multiple independent clusters.

### Q4. What is a managed cluster?

A managed cluster is a Kubernetes/OpenShift cluster registered/imported into ACM and centrally managed from the hub.

### Q5. What is Klusterlet?

Klusterlet is the ACM agent/component deployed on the managed cluster to establish communication with the hub and participate in centralized management.

### Q6. Does ACM directly manage every worker node?

No. ACM primarily manages the cluster/fleet layer. The OpenShift control plane and cluster operators manage resources and worker nodes inside each cluster.

### Q7. What if the ACM hub goes down?

Managed cluster workloads continue to run. The impact is on centralized management, governance updates, visibility and fleet operations.

### Q8. How do you manage policies across 50 clusters?

Use ACM Governance with policies, label clusters appropriately and use placement to target the required cluster groups.

### Q9. ACM vs Argo CD?

ACM is primarily for multi-cluster lifecycle, governance, placement and fleet visibility. Argo CD manages declarative application/configuration delivery from Git and reconciles desired state.

### Q10. Can ACM manage cloud and on-prem clusters together?

Yes, subject to supported distributions and versions. A single ACM hub can centrally manage supported clusters across on-prem and cloud environments.

### Q11. How would ACM work in an air-gapped environment?

Mirror required images and operator catalogs to an internal registry, install ACM using the mirrored sources and allow the required internal network connectivity between hub and managed clusters.

### Q12. What network components should you validate?

- DNS resolution
- API endpoint reachability
- HTTPS connectivity
- certificates
- firewall rules
- routes
- load balancers/proxies where applicable

### Q13. Why use labels and placement?

To avoid deploying every policy/application to every cluster and to target clusters based on criteria such as region, environment or business unit.

### Q14. Does ACM replace Prometheus/Grafana?

No. ACM provides fleet-level visibility. The underlying monitoring stack still handles collection and detailed cluster/workload metrics.

---

## 19. Whiteboard Diagram

```text
                         Enterprise Git
                              |
                         OpenShift GitOps
                           / Argo CD
                              |
                        ACM Placement
                              |
                    +-------------------+
                    |     ACM HUB       |
                    | OpenShift Cluster |
                    |                   |
                    | Lifecycle         |
                    | Governance        |
                    | Observability     |
                    +---------+---------+
                              |
          ---------------------------------------------
          |                    |                      |
          v                    v                      v
 +----------------+   +----------------+    +----------------+
 | Managed OCP 1  |   | Managed OCP 2  |    | Managed OCP 3  |
 | Production     |   | Production     |    | DR             |
 |                |   |                |    |                |
 | Klusterlet     |   | Klusterlet     |    | Klusterlet     |
 +----------------+   +----------------+    +----------------+
```

---

## 20. 30-Second Interview Answer

> Red Hat ACM provides centralized multi-cluster management using a hub-and-spoke architecture. The hub is an OpenShift cluster running ACM, while the spokes are independent managed OpenShift or supported Kubernetes clusters. ACM agents such as Klusterlet run on the managed clusters and communicate securely with the hub. ACM provides cluster lifecycle management, governance, placement and fleet visibility and integrates with OpenShift GitOps/Argo CD for application delivery. The important point is that the spoke clusters remain independent, so an ACM hub outage does not stop workloads already running on those clusters.

---

## 21. What to Remember

For interview purposes, remember this model:

```text
ACM Hub
  |
  +-- Cluster lifecycle
  +-- Governance
  +-- Placement
  +-- Visibility
  +-- GitOps integration
  |
  +----> Managed Cluster 1
  +----> Managed Cluster 2
  +----> Managed Cluster 3
```

And the two most important distinctions:

1. **Multi-AZ = HA within one cluster.**
2. **ACM hub-and-spoke = centralized management of multiple independent clusters.**
