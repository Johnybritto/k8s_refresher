# Day 2 – Workloads, Deployment Strategies & Debugging

Status: Covered in chat. These notes capture the Day 2 learning and follow-up questions.

---

## 1. Core Idea

In Kubernetes, production applications are usually not managed as standalone Pods. They are managed through higher-level workload controllers.

```text
Deployment -> ReplicaSet -> Pods
StatefulSet -> Stable identity Pods + stable storage
DaemonSet -> One Pod per node
Job -> Run-to-completion task
CronJob -> Scheduled Job
```

Senior interview point:

```text
Pods are disposable. Controllers maintain the desired state.
```

---

## 2. Pod Lifecycle

A Pod can be in these phases:

- Pending: Pod object exists, but it is not fully running yet.
- Running: Pod is bound to a node and at least one container is running or starting.
- Succeeded: All containers completed successfully.
- Failed: At least one container failed and the Pod will not recover based on restart policy.
- Unknown: API Server cannot determine Pod state, often due to node communication issue.

Important distinction:

```text
Running does not always mean healthy.
Ready means the Pod can receive Service traffic.
```

A Pod may be Running but not Ready if readiness probe is failing.

---

## 3. Init Containers

Init containers run before application containers.

Use cases:

- Wait for database or dependency.
- Run migration.
- Download config.
- Prepare permissions/files.

If an init container fails, the main app container does not start.

Debug command:

```bash
kubectl logs <pod> -c <init-container-name> -n <namespace>
```

Interview answer:

```text
Init containers are used for setup tasks that must complete before the main container starts. If a Pod is stuck in Init:CrashLoopBackOff, I check the failing init container logs, dependency connectivity, config, secrets, permissions, and events.
```

---

## 4. Sidecars

A sidecar is another container running in the same Pod as the main application.

Common examples:

- Istio Envoy proxy
- Log forwarder
- Config reloader
- Metrics exporter
- Security agent

Containers in the same Pod share:

- Network namespace
- Pod IP
- localhost
- Volumes, if mounted

Interview point:

```text
The app can talk to the sidecar through localhost because both containers share the same network namespace.
```

---

## 5. Deployment and ReplicaSet

A Deployment manages stateless applications. It creates and manages ReplicaSets. ReplicaSets maintain the desired number of Pods.

Flow:

```text
Deployment creates/updates ReplicaSet.
ReplicaSet creates Pods.
Scheduler assigns Pods to nodes.
Kubelet runs containers.
```

Strong answer:

```text
Deployment is the user-facing workload object for stateless apps. It manages rollout history and creates ReplicaSets. ReplicaSets ensure the required number of matching Pods exist.
```

---

## 6. Deployment Strategies

### 6.1 Recreate

Old Pods are terminated first, then new Pods are created.

Use when old and new versions cannot run together.

Pros:

- Simple
- Clean cutover

Cons:

- Downtime possible

Interview summary:

```text
Recreate is simple but can cause downtime. I use it only when old and new versions cannot safely run together.
```

---

### 6.2 Rolling Update

RollingUpdate is the default Deployment strategy.

Kubernetes gradually creates a new ReplicaSet and scales it up while scaling down the old ReplicaSet.

Important fields:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1
    maxUnavailable: 1
```

- maxSurge: Extra Pods allowed temporarily.
- maxUnavailable: How many Pods can be unavailable during rollout.

Production note:

```text
Rolling updates are safe only when readiness probes and backward compatibility are correct.
```

Strong answer:

```text
During a rolling update, the Deployment controller creates a new ReplicaSet for the new Pod template and gradually scales it up while scaling down the old ReplicaSet. maxSurge controls extra capacity and maxUnavailable controls minimum availability. Readiness probes decide when new Pods can receive traffic.
```

---

### 6.3 Blue-Green Deployment

Blue-green means two full versions exist:

```text
Blue = current production
Green = new version
```

Only one version receives traffic at a time.

Traffic is usually switched using the Service selector, ingress, gateway, or service mesh.

Example Service selector before cutover:

```yaml
selector:
  app: payment
  version: blue
```

After cutover:

```yaml
selector:
  app: payment
  version: green
```

Important correction:

```text
Do not update Service metadata labels for traffic. Update the Service selector.
```

Do Blue Pods shut down automatically?

```text
No. Switching traffic does not kill Pods. Service selector controls traffic. Deployment replica count controls whether Pods keep running.
```

Safe flow:

```text
1. Blue serves production.
2. Green is deployed but gets no traffic.
3. Green is tested.
4. Service selector or router is switched to Green.
5. Blue is kept temporarily for rollback.
6. Blue is scaled down later if Green is stable.
```

If Blue is scaled down before Service points to Green, users can face downtime because the Service may have no ready endpoints.

Strong answer:

```text
In blue-green deployment, switching traffic does not automatically shut down the old Pods. The Service selector decides which version gets traffic, while Deployments decide which Pods continue running. Usually Blue remains running after Green cutover for fast rollback. Once Green is stable, Blue can be scaled down or deleted.
```

---

### 6.4 Canary Deployment

Canary sends a small amount of traffic to a new version first.

Example:

```text
95% traffic -> stable v1
5% traffic -> canary v2
```

A plain Kubernetes Service cannot do exact 95/5 traffic splitting. It can only approximate canary if stable and canary Pods share the same selector and replica counts are adjusted.

Example:

```text
9 stable Pods + 1 canary Pod -> roughly 90/10, but not guaranteed exact.
```

For proper canary, use:

- Istio
- Argo Rollouts
- Flagger
- NGINX Ingress canary
- Gateway API weighted routing
- Service mesh

Strong answer:

```text
A normal Service does not understand canary weights. If stable and canary Pods match the same selector, both receive traffic and distribution is roughly based on backend selection and replica count. For accurate weighted canary, I would use Istio, Argo Rollouts, Flagger, Gateway API, or ingress-based routing.
```

---

### 6.5 A/B Deployment

A/B routes users based on attributes such as header, cookie, geography, user group, or feature flag.

Difference:

```text
Canary = percentage/progressive exposure.
A/B = rule-based exposure.
```

Implemented using API Gateway, Ingress, Istio, or feature flag systems.

---

### 6.6 Shadow / Traffic Mirroring

Real production traffic goes to the stable version, and a copy is mirrored to the new version. The new version's response is ignored.

Use for:

- Testing with real traffic
- Performance validation
- Behavioral comparison

Risk:

```text
Avoid side effects. Do not blindly mirror payment/order/email/database-write traffic without protection.
```

---

### 6.7 Feature Flags

Feature flags separate deployment from release.

New code can be deployed but disabled until enabled for selected users or percentages.

Benefit:

```text
Rollback can be instant by disabling the flag instead of redeploying.
```

---

### 6.8 Progressive Delivery

Progressive delivery automates canary rollout based on metrics.

Example:

```text
5% -> check metrics -> 20% -> check metrics -> 50% -> check metrics -> 100%
Rollback automatically if metrics fail.
```

Common tools:

- Argo Rollouts
- Flagger
- Istio
- Prometheus

---

## 7. Service Behavior During Blue-Green / Canary

A Kubernetes Service does not know blue, green, stable, or canary by itself.

It only knows:

```text
Which Ready Pods match my selector?
```

If new Pods match the Service selector, they receive traffic.

If new Pods do not match the Service selector, they receive no traffic.

If both blue and green match the same selector, traffic is mixed.

If only blue matches, green is deployed but idle.

Key line:

```text
Kubernetes Service gives label-based load balancing, not release-aware traffic management.
```

---

## 8. Deployment vs StatefulSet

Deployment:

- Stateless workloads
- Pods are interchangeable
- Random Pod names
- Easy scaling and rolling updates

StatefulSet:

- Stateful workloads
- Stable Pod names like mysql-0, mysql-1
- Stable network identity
- Stable storage per replica
- Ordered startup/termination

Examples:

```text
Deployment: frontend, REST API, stateless worker
StatefulSet: Kafka, Zookeeper, Elasticsearch, database cluster
```

Strong answer:

```text
A Deployment is used for stateless workloads where Pods are interchangeable. A StatefulSet is used when each Pod needs stable identity, stable network name, stable storage, and ordered lifecycle.
```

---

## 9. DaemonSet

DaemonSet ensures one Pod runs on every matching node.

Use cases:

- Log collectors
- Monitoring agents
- Security agents
- CNI agents
- Storage plugins
- Node exporter

Strong answer:

```text
I use DaemonSet when I need a copy of a Pod on every node or selected nodes, commonly for node-level agents such as logging, monitoring, security, networking, and storage plugins.
```

---

## 10. Job and CronJob

Job runs a task until completion.

CronJob creates Jobs on a schedule.

Use cases:

- Batch processing
- Database migration
- Cleanup job
- Scheduled backup
- Report generation

Important CronJob fields:

```yaml
concurrencyPolicy: Forbid
successfulJobsHistoryLimit: 3
failedJobsHistoryLimit: 3
backoffLimit: 2
```

Senior point:

```text
For CronJobs, think about concurrency, missed schedules, retries, cleanup, idempotency, and failure handling.
```

---

## 11. Troubleshooting: CrashLoopBackOff

CrashLoopBackOff means the container starts, crashes, restarts, and kubelet backs off before retrying again.

It is a symptom, not the root cause.

Common causes:

- App startup error
- Wrong command/args
- Missing Secret or ConfigMap
- Missing environment variable
- Dependency unavailable
- Permission issue
- Liveness probe too aggressive
- OOMKilled
- App exits but Deployment expects long-running process

Debug commands:

```bash
kubectl get pod <pod> -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl top pod <pod> -n <namespace>
```

Strong answer:

```text
For CrashLoopBackOff, I check which container is restarting, then inspect describe output, events, last state, exit code, restart count, OOMKilled status, probe failures, current logs, and previous logs. Then I validate config, secrets, command/args, dependencies, and resource limits.
```

---

## 12. Troubleshooting: Pending Pod

Pending usually means the Pod exists but cannot be scheduled or prepared.

Common causes:

- Insufficient CPU/memory
- Node selector mismatch
- Affinity mismatch
- Taints not tolerated
- PVC not bound
- Pod anti-affinity too strict
- Autoscaler cannot scale

Best command:

```bash
kubectl describe pod <pod> -n <namespace>
```

The Events section usually tells the reason.

Strong answer:

```text
For a Pending Pod, I start with kubectl describe pod and read scheduler events. Then I check node capacity, taints, tolerations, nodeSelector, affinity, resource requests, PVC status, and cluster autoscaler events.
```

---

## 13. Troubleshooting: ImagePullBackOff

ImagePullBackOff means kubelet failed to pull the container image and is backing off before retrying.

Common causes:

- Wrong image name/tag
- Image missing
- Private registry permission issue
- Missing imagePullSecret
- Registry unavailable
- Node DNS/network issue
- TLS/certificate problem
- Rate limiting

Commands:

```bash
kubectl describe pod <pod> -n <namespace>
kubectl get secret -n <namespace>
kubectl get pod <pod> -n <namespace> -o yaml
```

Strong answer:

```text
For ImagePullBackOff, I check Pod events first because kubelet records the exact pull error. Then I verify image name, tag, registry path, credentials, imagePullSecret, service account association, node connectivity, DNS, proxy, and registry availability.
```

---

## 14. Troubleshooting: Stuck Deployment Rollout

Commands:

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
kubectl get rs -n <namespace>
kubectl get pods -n <namespace>
kubectl describe pod <pod> -n <namespace>
```

Check:

- New Pods not created
- New Pods Pending
- New Pods CrashLoopBackOff
- Readiness probe failing
- ImagePullBackOff
- Resource limits/requests
- Admission policy failures
- maxSurge/maxUnavailable
- PDB constraints

Strong answer:

```text
For a stuck rollout, I check Deployment conditions, ReplicaSets, and new Pods. If Pods are not created, I check events, quota, and admission failures. If Pods are created but not Ready, I check readiness probes, logs, config, image pull, resources, and dependency connectivity.
```

---

## 15. Day 2 Final Summary

```text
Deployment manages ReplicaSet.
ReplicaSet manages Pods.
Scheduler places Pods.
Kubelet starts containers.
Readiness controls Service traffic.
Liveness controls container restart.
Events usually reveal the first troubleshooting clue.
```

Most important interview line:

```text
Kubernetes Deployment controls Pod rollout. Kubernetes Service controls which Pods receive traffic through selectors. Ingress/service mesh controls advanced routing such as weighted canary, A/B, and mirroring.
```
