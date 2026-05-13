# Day 3 – Kubernetes Networking Core, Services & Service Discovery

Status: Covered in chat. These notes capture Day 3 learning and follow-up questions about Service registration and discovery.

---

## 1. Core Idea

Kubernetes networking is based on this model:

```text
Every Pod gets its own IP, and Pods should be able to communicate across nodes.
```

But Pod IPs are temporary. Therefore applications should not depend on Pod IPs directly. They should call a stable Kubernetes Service name.

---

## 2. Pod IP vs Service IP

### Pod IP

A Pod IP belongs to a specific Pod.

If the Pod dies and is recreated, the new Pod may get a different IP.

### Service IP

A Service gets a stable virtual IP called ClusterIP.

Even if backend Pods change, the Service IP and DNS name remain stable.

Interview line:

```text
Pods are ephemeral. Services provide stable access to dynamic Pods.
```

---

## 3. Service Selector, EndpointSlice and Traffic

A Kubernetes Service uses selectors to find backend Pods.

Example:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment
  ports:
  - port: 80
    targetPort: 8080
```

This means:

```text
Send traffic to Ready Pods with label app=payment.
```

Kubernetes creates EndpointSlices containing the Ready backend Pod IPs and ports.

Flow:

```text
Pod labels
  -> Service selector
  -> EndpointSlice
  -> kube-proxy / eBPF dataplane
  -> Backend Pod IP
```

Important:

```text
A Service normally sends traffic only to Ready endpoints.
```

A Pod can be Running but not Ready. If readiness probe fails, it is usually removed from normal Service traffic.

---

## 4. Service Types

### ClusterIP

Default Service type. Exposes the Service only inside the cluster.

Use case:

```text
Internal microservice-to-microservice communication.
```

Strong answer:

```text
ClusterIP provides a stable virtual IP inside the cluster. It is used for internal service-to-service communication and is not directly reachable from outside the cluster.
```

---

### NodePort

Exposes the Service on a port on every node.

Example:

```text
http://<node-ip>:30080
```

Why NodePort is not ideal at scale:

- Opens ports on every node.
- Port range is limited.
- Firewall management becomes difficult.
- External clients need node IP awareness.
- Node lifecycle changes can break assumptions.
- Usually still needs a Load Balancer in front.

Strong answer:

```text
NodePort is useful for simple exposure or as a building block behind a LoadBalancer, but it is not ideal as the main production access pattern because it exposes ports on every node and becomes difficult to manage at scale.
```

---

### LoadBalancer

Asks the cloud provider to provision an external load balancer.

Typical path:

```text
External user -> Cloud Load Balancer -> Node/Pod -> Service -> Pod
```

Production note:

```text
For many microservices, using one LoadBalancer per Service can become expensive and hard to manage. A common pattern is Load Balancer -> Ingress Controller/API Gateway -> Services -> Pods.
```

---

## 5. Ingress vs Service

Service provides stable access to Pods.

Ingress provides HTTP/HTTPS routing to Services.

Example:

```text
/payment -> payment-service
/orders  -> order-service
```

Interview line:

```text
Service is mainly L4 service abstraction. Ingress is L7 HTTP/HTTPS routing.
```

---

## 6. kube-proxy, IPVS and eBPF

kube-proxy runs on every node and implements Service traffic routing by watching Services and EndpointSlices.

### iptables mode

Programs iptables NAT rules.

Pros:

- Stable
- Common
- Simple conceptually

Cons:

- Can become difficult at very large scale
- Troubleshooting iptables chains can be complex

### IPVS mode

Uses Linux IP Virtual Server.

Pros:

- Better load-balancing model at scale
- Multiple algorithms

Cons:

- Requires IPVS kernel modules and more operational knowledge

### eBPF mode

Used by CNIs like Cilium to replace or enhance kube-proxy functionality.

Pros:

- High performance
- Better observability
- Advanced networking/security capability

Cons:

- CNI-specific troubleshooting
- Requires deeper knowledge

Strong answer:

```text
kube-proxy implements Kubernetes Service routing. In iptables mode it programs NAT rules, in IPVS mode it uses Linux IPVS, and in eBPF-based CNIs like Cilium, Service routing can be handled by eBPF instead of kube-proxy.
```

---

## 7. Service Registration in Kubernetes

Service registration means:

```text
How does Kubernetes know which backend Pods belong to a Service?
```

In Kubernetes, apps usually do not manually register themselves like Eureka or Consul.

Registration is automatic through:

```text
Pod labels + Service selector + EndpointSlice
```

Step-by-step:

1. Deployment creates Pods.
2. Pods get labels such as `app=payment`.
3. Pods get IPs from the CNI.
4. Service has selector `app=payment`.
5. EndpointSlice controller watches Pods and Services.
6. Matching Ready Pods are added to EndpointSlice.
7. Service now has backend endpoints.

Example backend registry:

```text
payment-service endpoints:
10.244.1.10:8080
10.244.2.11:8080
10.244.3.12:8080
```

Interview line:

```text
In Kubernetes, EndpointSlice acts like the dynamic backend registry for a Service.
```

---

## 8. Service Discovery in Kubernetes

Service discovery means:

```text
How does one application find another application?
```

Kubernetes commonly uses CoreDNS.

A Service named `payment-service` in namespace `prod` gets DNS:

```text
payment-service.prod.svc.cluster.local
```

From the same namespace, applications can usually call:

```text
payment-service
```

From another namespace:

```text
payment-service.prod
```

Full DNS:

```text
payment-service.prod.svc.cluster.local
```

---

## 9. Full Service Discovery Request Flow

Example:

```text
order-service calls payment-service
```

Step-by-step:

1. `order-service` sends request to `http://payment-service:80`.
2. Pod resolver asks CoreDNS to resolve `payment-service`.
3. CoreDNS returns the Service ClusterIP, for example `10.96.20.15`.
4. Client sends traffic to `10.96.20.15:80`.
5. kube-proxy/IPVS/eBPF intercepts Service IP traffic.
6. Dataplane chooses one Ready backend endpoint from EndpointSlice.
7. Traffic is forwarded to a Pod IP, for example `10.244.2.11:8080`.
8. Response returns to the client Pod.

Memory line:

```text
Registration = Service selector finds Ready Pods and stores them in EndpointSlice.
Discovery = Client resolves Service DNS name through CoreDNS.
Routing = kube-proxy/eBPF sends traffic to one Ready backend Pod.
```

---

## 10. Where Is the Registry?

In Eureka/Consul, there is a separate registry component.

In Kubernetes, the registry concept is distributed across Kubernetes objects:

```text
Service object        -> stable frontend name/IP
EndpointSlice object  -> current Ready backend Pod IPs
CoreDNS               -> DNS discovery
API Server/etcd       -> stores the objects
```

Strong answer:

```text
Kubernetes does not require a separate Eureka-style registry for normal in-cluster Services. The Service and EndpointSlice objects act as the service registry, while CoreDNS provides service discovery.
```

---

## 11. What Happens When Pods Scale Up?

When replicas increase:

```text
New Pods are created
  -> New Pods get IPs
  -> If labels match and readiness passes
  -> EndpointSlice is updated
  -> Service can route to the new Pods
```

No application config change is needed.

---

## 12. What Happens When a Pod Dies?

When a Pod dies:

```text
Pod becomes unavailable
  -> EndpointSlice removes it or marks it not ready
  -> Service stops sending normal traffic to it
  -> ReplicaSet creates replacement Pod
  -> Replacement Pod gets a new IP
  -> EndpointSlice is updated again
```

The client still calls the same Service DNS name.

---

## 13. Service Discovery vs Load Balancing

Service discovery:

```text
How do I find payment-service?
```

Answer:

```text
CoreDNS resolves payment-service to ClusterIP.
```

Load balancing:

```text
Once I reach payment-service, which backend Pod receives the request?
```

Answer:

```text
kube-proxy/IPVS/eBPF forwards traffic to one Ready endpoint from EndpointSlice.
```

---

## 14. Troubleshooting: Pod Reachable by IP but Not Service

If Pod IP works but Service does not, the application may be healthy but Service registration/routing is broken.

Check:

```bash
kubectl get svc <service> -n <namespace>
kubectl describe svc <service> -n <namespace>
kubectl get pods -n <namespace> --show-labels
kubectl get endpointslice -n <namespace>
kubectl describe endpointslice <name> -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl get networkpolicy -n <namespace>
```

Likely causes:

- Service selector does not match Pod labels.
- Pod is not Ready.
- EndpointSlice is empty.
- targetPort is wrong.
- NetworkPolicy blocks traffic.
- kube-proxy/eBPF dataplane issue.

Strong answer:

```text
If a Pod is reachable directly by Pod IP but not through Service, I first check Service selector and Pod labels. Then I check EndpointSlice for ready endpoints, Pod readiness, Service port and targetPort, NetworkPolicy, and kube-proxy or eBPF dataplane health. Most common causes are selector mismatch, no ready endpoints, or wrong targetPort.
```

---

## 15. Troubleshooting: External Traffic Not Reaching Pod

Trace layer by layer:

```text
User -> DNS -> Load Balancer -> Node/Ingress -> Service -> EndpointSlice -> Pod
```

Check:

- Public DNS resolution
- Load balancer listener
- Load balancer health checks
- Firewall/security group
- Service type and ports
- NodePort if applicable
- EndpointSlice
- Pod readiness
- targetPort
- NetworkPolicy
- Application listening on `0.0.0.0`, not only `127.0.0.1`

Strong answer:

```text
For external traffic not reaching a Pod, I troubleshoot layer by layer: DNS, load balancer, listener, firewall/security group, Service, EndpointSlice, Pod readiness, targetPort, NetworkPolicy, and whether the application is listening on the correct port and interface.
```

---

## 16. Day 3 Final Summary

```text
Pod labels -> Service selector -> EndpointSlice -> kube-proxy/eBPF -> backend Pod
```

```text
Service registration: Pod labels + Service selector + EndpointSlice.
Service discovery: CoreDNS resolves Service DNS name.
Traffic routing: kube-proxy/IPVS/eBPF forwards to Ready backend Pod.
```

Most important senior-level line:

```text
Kubernetes does not require applications to manually register themselves. The platform registers Ready Pods automatically using selectors and EndpointSlices, and clients discover Services using DNS.
```
