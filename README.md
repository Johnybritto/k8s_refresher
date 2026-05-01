# 🚀 Principal Engineer (Kubernetes + OpenShift + Istio) – 2 Week Deep Revision Master Sheet

---

# 📅 DAY 1 – Control Plane & API Internals

## 📚 Topics
- Kubernetes architecture overview
- API Server request lifecycle (authn → authz → admission → etcd)
- Admission Controllers (mutating vs validating)
- Scheduler internals (filters, scoring, preemption)
- etcd (Raft, quorum, leader election, backup/restore, compaction)

## ⚠️ Scenarios
- API server latency spike → root causes?
- etcd disk full → impact?
- etcd quorum loss → what happens?

## 🏗️ Architecture
kubectl → API Server → Auth → Admission → etcd → Controller → Scheduler → Node

## 🔥 Top Interview Questions
- Explain API server request flow in detail
- How does etcd ensure consistency?
- What happens if etcd quorum is lost?
- How scheduler decides placement?

---

# 📅 DAY 2 – Workloads & Debugging

## 📚 Topics
- Pod lifecycle (init containers, sidecars)
- Deployments (rolling, canary, blue-green)
- StatefulSets (identity, storage)
- DaemonSets, Jobs, CronJobs

## ⚠️ Scenarios
- CrashLoopBackOff → debug steps
- Pending pod → resource/affinity issue
- ImagePullBackOff → root cause

## 🏗️ Architecture
Deployment → ReplicaSet → Pod → Node

## 🔥 Top Interview Questions
- Deployment vs StatefulSet?
- How rolling updates work internally?
- Debug stuck pod?

---

# 📅 DAY 3 – Networking (Core)

## 📚 Topics
- Pod networking model
- kube-proxy (iptables vs IPVS vs eBPF)
- Services (ClusterIP, NodePort, LoadBalancer)

## ⚠️ Scenarios
- Pod reachable via IP but not service
- External traffic not reaching pod

## 🏗️ Architecture
Client → LoadBalancer → Node → kube-proxy → Pod

## 🔥 Top Interview Questions
- How Kubernetes networking works?
- Why NodePort is not scalable?

---

# 📅 DAY 4 – Networking Advanced + Istio

## 📚 Topics
- CNI (Calico / Cilium)
- Network Policies
- DNS (CoreDNS)
- Istio:
  - Istiod + Envoy
  - VirtualService, DestinationRule
  - Traffic splitting, retries, circuit breaking
  - mTLS basics

## ⚠️ Scenarios
- Service fails after Istio injection
- Traffic split not working
- DNS resolution failure

## 🏗️ Architecture
Client → Envoy → Service → Envoy → Pod

## 🔥 Top Interview Questions
- How Istio works internally?
- mTLS flow in Istio?
- Service mesh vs API gateway?

---

# 📅 DAY 5 – Storage & Stateful Systems

## 📚 Topics
- PV/PVC lifecycle
- StorageClasses
- CSI drivers
- Stateful workloads

## ⚠️ Scenarios
- Pod restart → data loss
- PVC stuck pending

## 🏗️ Architecture
Pod → PVC → PV → Storage

## 🔥 Top Interview Questions
- PV vs PVC?
- Running DB on Kubernetes challenges?

---

# 📅 DAY 6 – Security + Certificates + Compliance

## 📚 Topics
- RBAC deep dive
- Service Accounts
- Pod Security
- cert-manager (TLS lifecycle)
- OPA / Kyverno
- Vulnerability scanning

## ⚠️ Scenarios
- Expired cert outage
- RBAC misconfiguration breach

## 🏗️ Architecture
User → API Server → RBAC → Policy → Workload

## 🔥 Top Interview Questions
- How RBAC works?
- Certificate rotation?
- Policy-as-code?

---

# 📅 DAY 7 – Kubernetes Internals

## 📚 Topics
- Controllers & Informers
- CRDs vs Operators
- Finalizers
- Garbage collection

## ⚠️ Scenarios
- Resource stuck terminating
- Operator not reconciling

## 🏗️ Architecture
CRD → Controller → Reconcile → Desired State

## 🔥 Top Interview Questions
- Reconciliation loop?
- CRD vs Operator?

---

# 📅 DAY 8 – GitOps + Drift

## 📚 Topics
- ArgoCD / Flux
- Helm vs Kustomize
- Drift detection

## ⚠️ Scenarios
- Manual changes overwritten
- Git sync failure

## 🏗️ Architecture
Git → GitOps Tool → Cluster

## 🔥 Top Interview Questions
- GitOps principles?
- ArgoCD working?

---

# 📅 DAY 9 – Observability + SRE

## 📚 Topics
- Prometheus
- Loki / EFK
- Jaeger
- SLO / MTTR

## ⚠️ Scenarios
- Latency spike debugging
- Alert fatigue

## 🏗️ Architecture
App → Metrics/Logs → Monitoring → Alerts

## 🔥 Top Interview Questions
- Design observability system?
- Reduce MTTR?

---

# 📅 DAY 10 – Scaling + Upgrades

## 📚 Topics
- HPA / VPA / Cluster Autoscaler
- API limits
- etcd tuning
- Cluster upgrades
- Node draining

## ⚠️ Scenarios
- Cluster slow at scale
- Upgrade failure

## 🏗️ Architecture
Metrics → Autoscaler → Scaling

## 🔥 Top Interview Questions
- Scaling limits?
- Zero downtime upgrade?

---

# 📅 DAY 11 – OpenShift Deep Dive

## 📚 Topics
- OpenShift architecture
- Routes vs Ingress
- SCC vs RBAC
- OpenShift Virtualization
- OLM
- Upgrade lifecycle

## ⚠️ Scenarios
- Why OpenShift?
- VM migration challenges

## 🏗️ Architecture
User → Route → Service → Pod

## 🔥 Top Interview Questions
- OpenShift vs Kubernetes?
- SCC vs RBAC?

---

# 📅 DAY 12 – Multi-Cluster + Istio

## 📚 Topics
- Multi-cluster Kubernetes
- Identity federation
- Istio multi-cluster
- Failover

## ⚠️ Scenarios
- Cluster failure reroute
- Cross-cluster communication issue

## 🏗️ Architecture
Cluster A ↔ Mesh ↔ Cluster B

## 🔥 Top Interview Questions
- Multi-cluster design?
- Failover strategy?

---

# 📅 DAY 13 – Platform Engineering

## 📚 Topics
- Internal Developer Platform
- Backstage
- Crossplane
- API-driven infra

## ⚠️ Scenarios
- Self-service platform
- Platform drift

## 🏗️ Architecture
Developer → Platform API → Infra → Cluster

## 🔥 Top Interview Questions
- Platform engineering vs DevOps?
- Build IDP?

---

# 📅 DAY 14 – Failure Scenarios + Mock

## 📚 Topics
- etcd failure
- API server down
- network partition
- GitOps failure
- security breach

## ⚠️ Scenarios
- Full cluster outage
- Secret leak
- Upgrade rollback

## 🏗️ Architecture
Failure → Detection → Recovery → Prevention

## 🔥 Top Interview Questions
- Disaster recovery strategy?
- Real production incident?

---

# ✅ FINAL SELF-CHECK

After each day:
- Can I explain clearly?
- Can I debug real issue?
- Can I design at scale?

---

# 🚨 FINAL QUESTION

👉 Have you written / saved all topics you studied today?

If NO → You will forget  
If YES → You are preparing like a Principal Engineer
