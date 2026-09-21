# Kubernetes Packet Flow — Load Balancer → Ingress → kube-proxy → Pod

## Scenario

This runbook explains the common Kubernetes/AKS packet path when an ingress controller such as NGINX is exposed through a `Service type: LoadBalancer`.

Logical flow:

```text
Client
  ↓
External / Cloud Load Balancer
  ↓
Worker Node IP : NodePort
  ↓
kube-proxy programmed dataplane rules
  ↓
Ingress Controller Pod
  ↓
Ingress Controller evaluates Host / Path
  ↓
Backend Kubernetes Service
  ↓
kube-proxy programmed service rules
  ↓
Backend Pod
```

> Important: the Ingress object itself is **not** a network hop. It is configuration consumed by the ingress controller.

---

## 1. Client sends traffic to the application URL

Example:

```text
https://app.example.com/orders
```

DNS resolves the hostname:

```text
app.example.com
      ↓
20.30.40.50
```

where `20.30.40.50` is the public IP of the cloud load balancer.

```text
Client
   |
   | HTTPS :443
   v
Azure Load Balancer
20.30.40.50
```

---

## 2. The ingress controller is exposed through a LoadBalancer Service

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ingress-nginx-controller
spec:
  type: LoadBalancer
  ports:
    - port: 443
      targetPort: 443
      nodePort: 30443
```

Conceptually:

```text
LoadBalancer IP
20.30.40.50:443
       ↓
Worker Node
10.0.1.10:30443
```

The cloud load balancer health-checks backend nodes and forwards traffic to one of the healthy nodes.

Example:

```text
Azure LB
   |
   v
Node-2
10.0.1.12:30443
```

---

## 3. Traffic reaches the NodePort

The packet reaches:

```text
Destination:
10.0.1.12:30443
```

This is where kube-proxy becomes relevant.

### Important distinction

Do not say:

> The packet goes to kube-proxy.

A more accurate statement is:

> kube-proxy watches Services and EndpointSlices and programs the Linux dataplane. The Linux kernel forwards packets according to those rules.

In iptables mode, the relevant flow is approximately:

```text
PREROUTING
   ↓
KUBE-SERVICES
   ↓
KUBE-NODEPORTS
   ↓
KUBE-SVC-xxxxx
```

The rules map:

```text
NodePort 30443
     ↓
Service: ingress-nginx-controller
     ↓
Choose one ingress-controller endpoint
```

Suppose ingress-controller pods are:

```text
ingress-pod-1 = 10.244.1.5
ingress-pod-2 = 10.244.2.8
ingress-pod-3 = 10.244.3.4
```

The service rules may choose:

```text
10.244.2.8:443
```

---

## 4. DNAT sends traffic to the ingress controller pod

Originally:

```text
DST = Node-2:30443
```

The Linux service rules translate this to:

```text
DST = 10.244.2.8:443
```

Conceptually:

```text
NodeIP:NodePort
      ↓ DNAT
IngressPodIP:443
```

The packet then travels through the Kubernetes CNI network.

If the selected ingress pod is on another node:

```text
Node-2
   |
   | CNI network
   v
Node-3
   |
   v
Ingress Pod
10.244.2.8
```

### externalTrafficPolicy

With `externalTrafficPolicy: Local`, load-balancer traffic can be directed only to nodes that have a local ingress-controller endpoint, which can avoid an additional cross-node hop and preserve the original source IP.

---

## 5. The ingress controller receives the request

The packet reaches the ingress-controller pod:

```text
NGINX ingress controller
10.244.2.8:443
```

The ingress controller can terminate TLS and inspect:

```text
Host: app.example.com
Path: /orders
```

Example Ingress:

```yaml
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /orders
        backend:
          service:
            name: orders-service
            port:
              number: 8080
```

The ingress controller therefore understands:

```text
app.example.com/orders
          ↓
orders-service:8080
```

### Role of the Ingress object

```text
Ingress object
      ↓
routing configuration
      ↓
Ingress Controller
```

The Ingress object itself never receives packets.

---

## 6. Ingress controller sends traffic to the backend Service

The ingress controller creates a connection toward:

```text
orders-service:8080
```

Suppose the Service ClusterIP is:

```text
10.96.20.100:8080
```

The packet is approximately:

```text
SRC = ingress-pod
10.244.2.8

DST = Service
10.96.20.100:8080
```

This introduces the **second service-routing event** in the packet path.

---

## 7. kube-proxy programs backend Service routing

kube-proxy watches the backend Service and its EndpointSlices.

Example endpoints:

```text
orders-pod-1 → 10.244.4.10:8080
orders-pod-2 → 10.244.5.11:8080
orders-pod-3 → 10.244.6.12:8080
```

For:

```text
10.96.20.100:8080
```

the service-routing rules select an endpoint, for example:

```text
10.244.5.11:8080
```

Another DNAT occurs:

```text
Service IP:
10.96.20.100:8080

        ↓ DNAT

Pod IP:
10.244.5.11:8080
```

---

## 8. CNI delivers the packet to the application Pod

If the backend pod runs on another worker node:

```text
Ingress Pod
Node-3
   |
   | Kubernetes CNI
   v
Node-4
   |
   v
orders-pod-2
10.244.5.11
```

The application receives the request.

---

# Full End-to-End Packet Flow

```text
CLIENT
  |
  v
DNS
  |
  v
Cloud Load Balancer Public IP :443
  |
  v
Worker Node : NodePort
  |
  v
kube-proxy programmed rules
  |
  | DNAT
  v
Ingress Controller Pod :443
  |
  | Host/path routing
  v
Backend Service ClusterIP :8080
  |
  v
kube-proxy programmed rules
  |
  | DNAT
  v
Backend Pod :8080
```

---

# Where EndpointSlice Fits

EndpointSlice stores endpoint information for Services.

Conceptually:

```text
Service
   |
   v
EndpointSlice

10.244.4.10
10.244.5.11
10.244.6.12
```

kube-proxy watches:

```text
Service objects
+
EndpointSlice objects
```

and programs the dataplane accordingly.

Do **not** think of it as:

```text
Packet → EndpointSlice
```

EndpointSlice is **control-plane metadata**, not a network hop.

---

# The Two kube-proxy / Service-Routing Points

## First service-routing event

External traffic:

```text
Load Balancer
     ↓
Ingress Service / NodePort
     ↓
kube-proxy programmed rules
     ↓
Ingress Controller Pod
```

## Second service-routing event

Internal traffic:

```text
Ingress Controller Pod
     ↓
Application Service
     ↓
kube-proxy programmed rules
     ↓
Application Pod
```

---

# Interview-Ready Answer

> In the common ingress flow, external traffic first reaches the cloud load balancer, which forwards it to a worker node through the ingress controller's LoadBalancer/NodePort Service. kube-proxy does not forward the packet itself; it watches Services and EndpointSlices and programs the node dataplane, such as iptables or IPVS. Those rules DNAT the NodePort traffic to an ingress-controller pod. The ingress controller then performs L7 host/path routing and connects to the backend Kubernetes Service. Service-routing rules are applied again to translate the Service ClusterIP to one of the backend pod endpoints. The CNI then delivers the packet to the selected pod.

---

# Key Interview Nuances

1. **Ingress is not a packet hop.** It is configuration consumed by the ingress controller.
2. **The packet does not go to kube-proxy.** kube-proxy programs Linux dataplane rules.
3. **EndpointSlice is not a packet hop.** It is endpoint metadata.
4. **There can be two service-routing events:** ingress Service and application Service.
5. **DNAT happens at both Service translations.**
6. **Cross-node traffic is possible** depending on where the selected ingress/backend pods are located.
7. **externalTrafficPolicy: Local** can reduce cross-node forwarding and preserve source IP.
8. With **Cilium kube-proxy replacement**, eBPF performs Service translation instead of iptables/IPVS.
9. Some ingress-controller configurations can route directly to pod endpoints instead of going through a Service ClusterIP, so the exact internal hop can vary by implementation.

---

# Quick Memory Flow

```text
Client
  ↓
Cloud LB
  ↓
NodePort
  ↓
kube-proxy programmed dataplane
  ↓
Ingress Controller Pod
  ↓
Ingress rule: Host / Path
  ↓
Backend Service
  ↓
kube-proxy programmed dataplane
  ↓
Backend Pod
```
