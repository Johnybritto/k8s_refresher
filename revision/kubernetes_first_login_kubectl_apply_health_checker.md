# Kubernetes First-Time Access, kubectl Apply Flow & Simple Health Checker

## 1. First-time access to Kubernetes

```text
User obtains cluster credentials / kubeconfig
  |
  v
kubectl reads ~/.kube/config (or KUBECONFIG)
  |
  v
Finds cluster API endpoint and verifies API server TLS certificate
  |
  v
API Server authenticates user (certificate / token / OIDC)
  |
  v
API Server authorizes requested action using RBAC
  |
  v
Request is allowed or denied
```

**Note:** kubectl does not perform a separate Kubernetes "login". The first-time setup is obtaining valid credentials and configuring kubeconfig. Authentication/authorization occur on each API request.

## 2. What happens when running `kubectl apply -f deploy.yaml`

```text
User
  |
  v
kubectl apply -f deploy.yaml
  |
  v
1. kubectl reads kubeconfig + deployment YAML
  |
  v
2. API Server
   - TLS connection + Authentication
   - Authorization (RBAC)
   - Admission / validation
  |
  v
3. Desired Deployment object persisted in etcd
  |
  v
4. Deployment Controller
   Creates/updates ReplicaSet
  |
  v
5. ReplicaSet Controller
   Creates Pods
  |
  v
6. Scheduler
   Selects worker node and records Pod binding
  |
  v
7. Kubelet (selected worker)
   Observes assigned Pod through API Server
  |
  v
8. Container Runtime (containerd / CRI-O)
   Creates Pod sandbox (CNI sets up Pod network)
   Pulls images and starts containers
  |
  v
9. Kubelet evaluates startup/readiness/liveness probes (if configured)
  |
  v
10. Pod becomes Ready when all required containers are Ready
```

### Component cheat sheet

| Component | Responsibility |
|---|---|
| kubectl | Sends API requests |
| API Server | Authenticates, authorizes, validates and processes requests |
| etcd | Stores desired and observed cluster state |
| Deployment Controller | Manages ReplicaSets |
| ReplicaSet Controller | Maintains requested Pods |
| Scheduler | Picks a node for unscheduled Pods |
| Kubelet | Runs/reconciles assigned Pods |
| Container Runtime | Creates Pod sandbox and containers |
| CNI | Configures Pod networking |

**Interview point:** API Server does **not** directly create containers. Controllers reconcile, Scheduler assigns a node, and Kubelet talks to the container runtime. A successful `kubectl apply` means the resource was accepted, **not** that Pods are Ready.

## 3. Simple application health checker (Python)

```python
import requests

def health_check(url):
    try:
        response = requests.get(url, timeout=3)
        return response.status_code == 200
    except requests.RequestException:
        return False

url = "http://my-app:8080/health"

if health_check(url):
    print("Application is Healthy")
else:
    print("Application is Unhealthy")
```

Requires `requests` (`pip install requests`). Replace the URL with the application health endpoint.

### Kubernetes readiness and liveness probes

Add these beneath a container definition in a Deployment manifest:

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  periodSeconds: 10
```

- **Readiness failure:** Pod becomes NotReady and is removed from matching Service endpoint routing.
- **Liveness failure:** after repeated failures (default `failureThreshold: 3`), Kubelet restarts the container.
- Use timeouts and sensible check intervals to keep probes lightweight. In practice, readiness and liveness endpoints can differ.
