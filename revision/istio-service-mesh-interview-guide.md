# Istio Service Mesh — Interview Preparation Guide

## 1. Why Istio Exists

Kubernetes gives us basic service discovery and connectivity, but as microservices grow, every application team starts solving the same communication problems repeatedly.

Without a service mesh, each service may need to implement:

- TLS between services
- Authentication between services
- Authorization
- Retries
- Timeouts
- Circuit breaking
- Load balancing behavior
- Canary routing
- Request metrics
- Access logs
- Distributed tracing

That creates duplicated logic across many languages and teams.

Example:

```text
Service A
  |
  | TLS code
  | retry code
  | tracing code
  | auth code
  v
Service B
```

If 100 services implement this separately, consistency becomes difficult.

Istio moves these common communication concerns into infrastructure:

```text
Service A
   |
 Envoy
   |
   | mTLS
   | retry
   | routing
   | telemetry
   v
 Envoy
   |
Service B
```

The application focuses on business logic. The mesh handles communication policy.

### Problems Istio Solves

| Problem | Istio capability |
|---|---|
| Plain-text service traffic | mTLS |
| Dynamic pod IPs | Service discovery + Envoy endpoint config |
| Inconsistent retry logic | Central retry policy |
| Slow/failing dependencies | Timeouts + circuit breaking |
| Canary deployments | Weighted routing |
| Different security logic per app | Workload identity + AuthorizationPolicy |
| Poor service visibility | Metrics, logs, tracing |
| Multi-cluster service communication | Mesh gateways and shared trust |

### Memory line

```text
Kubernetes connects services.
Istio secures, controls and observes service-to-service communication.
```

---

# 2. What Is Istio?

Istio is a service mesh used to manage service-to-service communication without requiring every application to implement the same networking logic.

Its main functions are:

- Traffic management
- Security
- Resilience
- Observability
- Multi-cluster communication

---

# 3. Istio Architecture

Istio has two logical planes:

```text
                         ISTIO

                 +----------------+
                 | CONTROL PLANE  |
                 |     istiod     |
                 +-------+--------+
                         |
                 config / certs
                       xDS
                         |
             +-----------+-----------+
             |                       |
             v                       v

      +-------------+         +-------------+
      | Service A   |         | Service B   |
      | App + Envoy | <-----> | App + Envoy |
      +-------------+  mTLS   +-------------+

                    DATA PLANE
```

The control plane configures the proxies.

The data plane carries the real application traffic.

### Important interview point

Application traffic does **not** pass through `istiod`.

```text
WRONG:
Service A -> istiod -> Service B

CORRECT:
Service A -> Envoy A -> Envoy B -> Service B
```

---

# 4. Control Plane — istiod

`istiod` is the current Istio control-plane component.

It performs functions such as:

- Service discovery
- Proxy configuration
- Certificate management
- Security policy distribution
- Conversion of Istio config into Envoy configuration

Flow:

```text
Kubernetes API
      |
      | Services
      | Pods
      | VirtualServices
      | DestinationRules
      | Security Policies
      v
    istiod
      |
      | xDS
      v
 Envoy Proxies
```
Istiod converts Kubernetes and Istio configuration into configuration understood by Envoy and distributes it dynamically.


### Historical names

Older Istio versions had separate components:

```text
Pilot   -> proxy configuration / discovery
Citadel -> certificate authority
Galley  -> configuration processing
Mixer   -> old telemetry/policy component
```

Today these should not be presented as separate active control-plane deployments. Their responsibilities were consolidated or removed.

---

# 5. Data Plane — Envoy

In sidecar mode, every participating workload gets an Envoy proxy.

```text
Pod A
+------------------+
| App Container    |
| Envoy Sidecar    |
+------------------+
```

Envoy handles:

- Routing
- Load balancing
- mTLS
- Certificate validation
- Retries
- Timeouts
- Circuit breaking
- Authorization
- Metrics
- Access logs
- Tracing

---

# 6. Service-to-Service Request Flow

Example:

```text
Frontend Pod                         Payment Pod
+------------------+               +------------------+
| Frontend App     |               | Payment App      |
|       |          |               |       ^          |
|       v          |               |       |          |
| Envoy Sidecar    |==============>| Envoy Sidecar    |
+------------------+     mTLS      +------------------+
```

Step by step:

1. Frontend calls the Payment Kubernetes Service.
2. Traffic is redirected to Frontend's local Envoy.
3. Envoy evaluates the relevant routing configuration.
4. `VirtualService` may choose version/path/subset.
5. `DestinationRule` applies destination policy.
6. Source Envoy selects a backend endpoint.
7. Source and destination proxies establish mTLS.
8. Destination identity is verified.
9. `AuthorizationPolicy` is evaluated.
10. Destination Envoy forwards the request to the Payment app.

### Memory flow

```text
App
 ↓
Local Envoy
 ↓
mTLS
 ↓
Remote Envoy
 ↓
App
```

---

# 7. North-South Traffic

External traffic normally enters through an Istio ingress gateway.

```text
Internet
   |
   v
External Load Balancer
   |
   v
Istio Ingress Gateway
   |
   v
VirtualService
   |
   v
Kubernetes Service
   |
   v
Application Pod
```

The ingress gateway is itself an Envoy-based proxy running at the edge of the mesh.

---

# 8. Gateway vs VirtualService

Think of it this way:

```text
Gateway
= Which door is open?

VirtualService
= Where should traffic entering that door go?
```

Example:

```text
Gateway:
443 / HTTPS / api.company.com

VirtualService:
/api/orders  -> Order Service
/api/payment -> Payment Service
```

A Gateway by itself does not define the full L7 application routing behavior.

---

# 9. VirtualService

`VirtualService` decides **WHERE traffic goes**.

Uses:

- Host/path routing
- Header routing
- Canary traffic splitting
- Blue/green routing
- Retries
- Timeouts
- Fault injection
- Traffic mirroring

Example:

```text
reviews
 |
 +-- 90% -> v1
 |
 +-- 10% -> v2
```

Memory:

```text
VirtualService = WHERE
```

---

# 10. DestinationRule

`DestinationRule` determines **HOW traffic behaves once a destination is selected**.

Uses:

- Subsets
- Load-balancing policy
- Connection pools
- Outlier detection
- TLS policy
- Circuit-breaking settings

Example:

```text
VirtualService:
Go to reviews-v2

DestinationRule:
For reviews-v2:
- use subset v2
- apply connection limits
- eject unhealthy endpoints
- apply TLS policy
```

Memory:

```text
VirtualService = WHERE
DestinationRule = HOW
```

---

# 11. Important Istio Traffic Resources

| Resource | Interview meaning |
|---|---|
| VirtualService | Where traffic goes |
| DestinationRule | How destination traffic behaves |
| Gateway | Edge listener |
| ServiceEntry | Register an external/non-K8s service |
| Sidecar | Limit proxy configuration scope |
| WorkloadEntry | Represent external/VM workload |

---

# 12. ServiceEntry

A `ServiceEntry` adds an external or non-Kubernetes destination into Istio's service registry.

Example:

```text
Payment Service
      |
      v
api.stripe.com
```

With ServiceEntry:

```text
Istio Service Registry
      |
api.stripe.com
```

Useful when you want mesh routing/security/telemetry behavior for an external destination.

---

# 13. Egress Gateway

Instead of allowing every pod to reach external services independently:

```text
100 Pods
   |
   +----> Internet
```

Use a controlled path:

```text
100 Pods
   |
   v
Egress Gateway
   |
   v
Internet / External APIs
```

Benefits:

- Centralized egress control
- Fixed source IP possibilities
- Auditability
- Security inspection
- Policy enforcement

---

# 14. Canary Deployment

```text
Users
 |
 v
reviews
 |
 +----90%----> v1
 |
 +----10%----> v2
```

This is typically implemented with a VirtualService plus DestinationRule subsets.

---

# 15. Header-Based Routing

```text
Header: user-type=beta
          |
          v
         v2

All others
          |
          v
         v1
```

Useful for controlled testing or internal users.

---

# 16. Traffic Mirroring

```text
Production request
      |
      +-----> v1   (real response)
      |
      +-----> v2   (copy only)
```

The mirrored response is discarded.

Useful for validating a new version using real traffic without serving its response to users.

---

# 17. Retries

Retries help with temporary failures.

```text
A -> B

Attempt 1 -> fail
Attempt 2 -> fail
Attempt 3 -> success
```

But too many retries create retry storms.

Example:

```text
10,000 failing requests
x 3 retries
= potentially 30,000 attempts
```

Senior answer:

> Retries should always be designed together with timeouts, circuit breaking and sensible limits so they do not amplify an outage.

---

# 18. Timeouts

Without timeout:

```text
Service A
   |
   | waits
   | waits
   v
Service B
```

With timeout:

```text
A -> B

Timeout reached
A stops waiting
```

Timeouts stop slow dependencies from holding upstream resources indefinitely.

---

# 19. Circuit Breaking

Circuit-breaking behavior is mainly configured using `DestinationRule`.

Common controls:

- maxConnections
- maxPendingRequests
- maxRequestsPerConnection
- outlierDetection

Example:

```text
Payment Pods

P1 healthy
P2 healthy
P3 repeatedly failing

Envoy ejects P3 temporarily

Traffic -> P1 + P2
```

Important distinction:

```text
VirtualService
-> retries / timeout / routing

DestinationRule
-> LB / connection pool / outlier detection
```

---

# 20. Fault Injection

Istio can intentionally inject failures.

```text
10% requests -> 2-second delay
```

or:

```text
5% requests -> HTTP 500
```

Useful for testing resilience without changing application code.

---

# 21. Istio Security

Three security resources matter most:

```text
PeerAuthentication
RequestAuthentication
AuthorizationPolicy
```

Memory:

```text
PeerAuthentication
= workload-to-workload mTLS

RequestAuthentication
= JWT / end-user authentication

AuthorizationPolicy
= is the request allowed?
```

---

# 22. mTLS

Normal TLS:

```text
Client ----TLS----> Server
Client verifies Server
```

Mutual TLS:

```text
Service A <==== mTLS ====> Service B

A verifies B
B verifies A
```

Istio uses mTLS for encrypted workload communication and workload identity.

---

# 23. Workload Identity

Istio workload identity is normally associated with the Kubernetes ServiceAccount.

```text
Pod
 |
ServiceAccount
 |
Istio identity
 |
Certificate
```

Typical SPIFFE-style identity:

```text
spiffe://cluster.local/ns/payments/sa/payment-sa
```

This lets authorization depend on service identity rather than dynamic pod IPs.

---

# 24. Certificate Flow

```text
Workload starts
      |
      v
Authenticates using Kubernetes identity
      |
      v
    istiod / CA
      |
      v
Workload certificate issued
      |
      v
Proxy receives certificate
      |
      v
mTLS with peer workload
```

Certificates are short-lived and automatically rotated.

---

# 25. PeerAuthentication

Main modes:

```text
STRICT
PERMISSIVE
DISABLE
```

### STRICT

Only mTLS traffic is accepted.

### PERMISSIVE

Both plaintext and mTLS are accepted.

Useful during migration:

```text
Phase 1 -> PERMISSIVE
Phase 2 -> migrate all workloads
Phase 3 -> STRICT
```

---

# 26. RequestAuthentication

Used mainly for JWT validation.

```text
User
 |
JWT
 |
 v
Ingress / Workload
 |
RequestAuthentication
 |
JWT validation
```

Important:

`RequestAuthentication` validates supplied credentials. To require authenticated users, combine it with `AuthorizationPolicy`.

---

# 27. AuthorizationPolicy

Controls who can access what.

Example:

```text
Frontend SA ----ALLOW----> Payment
Reporting SA ----DENY----X Payment
```

Can match on:

- ServiceAccount / principal
- Namespace
- Source IP
- HTTP method
- Path
- Headers
- JWT claims

Important interview distinction:

```text
mTLS identity
= which workload/service is calling

JWT identity
= which end user is represented
```

---

# 28. NetworkPolicy vs AuthorizationPolicy

```text
Kubernetes NetworkPolicy
= network connectivity controls
= mainly L3/L4

Istio AuthorizationPolicy
= workload identity + application-aware controls
= L4/L7 depending mode
```

Example:

NetworkPolicy:

```text
Allow frontend namespace to TCP/8080
```

Istio:

```text
Allow frontend-sa GET /orders
Deny DELETE /orders
```

Use both for defense in depth.

---

# 29. Observability

Because Envoy sees service traffic, Istio can provide telemetry without every app independently implementing it.

Golden signals:

- Latency
- Traffic
- Errors
- Saturation

Typical ecosystem:

```text
Envoy
 |
 +--> Prometheus
 |
 +--> Grafana
 |
 +--> OpenTelemetry / tracing backend
 |
 +--> Kiali
```

Kiali can show service relationships such as:

```text
Frontend
   |
   v
Orders
   |
   +--> Inventory
   |
   +--> Payment
```

---

# 30. Distributed Tracing Trap

Istio can create proxy spans, but applications still need to propagate tracing context across their outbound calls.

Typical headers:

- traceparent
- tracestate
- x-request-id
- B3 headers depending tracing setup

If the app does not forward trace context:

```text
Trace A
   |
break
   |
Trace B
```

So saying “Istio makes distributed tracing fully automatic with zero app responsibility” is inaccurate.

---

# 31. Sidecar Injection

Flow:

```text
Deployment creates Pod
       |
       v
Kubernetes API
       |
       v
Istio Mutating Webhook
       |
       v
Pod spec modified
       |
       +--> App container
       |
       +--> istio-proxy
```

Before injection:

```text
Pod
 App
```

After injection:

```text
Pod
 App
 Envoy
```

Injection applies at pod creation time; existing pods must be recreated to pick up sidecar changes.

---

# 32. Istio CNI

Istio CNI performs traffic-redirection networking setup at the node level.

```text
Node
 |
Istio CNI
 |
configures interception
 |
Pod
 + App
 + Envoy
```

This avoids requiring every workload pod to use a privileged networking init container.

Especially relevant for hardened Kubernetes/OpenShift environments.

Do not describe Istio CNI as simply “eBPF”; these are different concepts.

---

# 33. Ambient Mesh

Istio supports sidecar and ambient data-plane modes.

## Sidecar

```text
Pod A                    Pod B
App                      App
 |                        ^
 v                        |
Envoy ================= Envoy
```

One Envoy per workload.

## Ambient

```text
Node A                           Node B
Pod A                            Pod B
  |                                ^
  v                                |
ztunnel ======== HBONE ========= ztunnel
```

`ztunnel` provides the L4 secure overlay.

It handles capabilities such as:

- mTLS
- workload identity
- L4 authorization
- TCP telemetry

For L7 behavior:

```text
ztunnel
   |
   v
Waypoint Proxy
   |
   v
ztunnel
```

Waypoint proxies provide capabilities such as:

- HTTP routing
- Retry/timeout policy
- L7 authorization
- HTTP telemetry

### Interview line

> Ambient does not remove proxies. It changes the proxy model from one Envoy sidecar per workload to node-level ztunnel for L4, with optional waypoint proxies for L7 functionality.

---

# 34. Sidecar vs Ambient

| Area | Sidecar | Ambient |
|---|---|---|
| Base proxy | Envoy per workload | ztunnel per node |
| L4 mTLS | Envoy | ztunnel |
| L7 features | Envoy | waypoint |
| App pod modification | Yes | No sidecar needed |
| Proxy lifecycle | Coupled to pod | Decoupled |
| Baseline resource overhead | Per workload | Shared L4 layer |

---

# 35. Istio High Availability

## Control Plane

Run multiple `istiod` replicas.

```text
             Service
                |
       +--------+--------+
       |        |        |
    istiod-1 istiod-2 istiod-3
```

Because live traffic does not pass through istiod, losing one control-plane instance does not directly stop data-plane traffic.

## Ingress Gateway

Production:

```text
External Load Balancer
         |
    +----+----+
    |         |
Gateway-1  Gateway-2
    |         |
    +----+----+
         |
      Services
```

Use:

- Multiple replicas
- HPA
- PDB
- Topology spread / anti-affinity
- Multiple nodes/zones

---

# 36. What Happens if istiod Goes Down?

Existing proxies retain their last known configuration, so existing traffic can continue.

However the following start degrading:

- New configuration updates
- New workloads needing config
- Endpoint/service discovery updates
- Certificate issuance
- Certificate rotation

If the outage lasts long enough, stale discovery and certificate expiry can eventually impact traffic.

Senior answer:

> Istiod is not in the data path, so an istiod outage does not immediately take down existing traffic, but it does affect control-plane convergence, service discovery updates, new workloads and eventually certificate lifecycle.

---

# 37. What Happens if Envoy Dies?

For a sidecar workload:

```text
App
 |
Envoy X
```

Mesh networking for that pod is impacted.

The pod should normally be considered unhealthy/restarted rather than bypassing mesh security.

---

# 38. Multi-Cluster Istio

Problem:

```text
Cluster A                Cluster B
Service A  ----------->  Service B
```

If networks are directly routable, services may communicate directly.

For separate networks:

```text
Cluster A
   |
East-West Gateway
   |
   | secure cross-cluster traffic
   |
East-West Gateway
   |
Cluster B
```

Common topologies:

- Multi-primary
- Primary-remote

Shared trust is critical so workloads can verify cross-cluster identities.

For interviews, understand the architecture and trust model before memorizing installation commands.

---

# 39. Troubleshooting Flow

Do not blame Istio first.

## Step 1 — Verify Kubernetes

```text
kubectl get pods
kubectl get svc
kubectl get endpoints
```

Check application, Service and endpoints.

## Step 2 — Is the proxy injected?

```text
READY 1/1 -> likely app only
READY 2/2 -> app + sidecar
```

## Step 3 — Analyze config

```text
istioctl analyze
```

## Step 4 — Check proxy sync

```text
istioctl proxy-status
```

## Step 5 — Inspect actual Envoy config

```text
istioctl proxy-config listeners <pod>
istioctl proxy-config routes <pod>
istioctl proxy-config clusters <pod>
istioctl proxy-config endpoints <pod>
istioctl proxy-config secret <pod>
```

Memory:

```text
Listener -> Is the port configured?
Route    -> Does routing exist?
Cluster  -> Does Envoy know the destination?
Endpoint -> Does it know backend pods?
Secret   -> Does it have certificates?
```

---

# 40. Common Failure Scenarios

## 503 / no route

Check:

- VirtualService
- Gateway binding
- Host/path mismatch
- DestinationRule subset
- Service/endpoints

## mTLS failure

Check:

- PeerAuthentication
- DestinationRule TLS settings
- Certificates
- ServiceAccount identity
- Certificate validity / time sync

## Canary version gets no traffic

Check:

- VirtualService weights
- DestinationRule subsets
- Pod labels

Example:

```text
Subset expects:
version: v2

Pod has:
version: V2
```

No match.

---

# 41. Useful Envoy Response Flags

Know only the important ones:

```text
NR = No route
UF = Upstream connection failure
UC = Upstream connection termination
UO = Upstream overflow / circuit-breaker pressure
```

Important:

`UO` does not mean “outlier detection ejected the endpoint.”

---

# 42. Istio vs Kubernetes NetworkPolicy

```text
NetworkPolicy
-> Can this network connection happen?

Istio AuthorizationPolicy
-> Is this workload/user allowed to make this request?
```

NetworkPolicy is mainly network-layer isolation.

Istio can make application-aware and identity-aware policy decisions.

---

# 43. Istio vs API Gateway

API Gateway:

```text
External Client
      |
      v
API Gateway
      |
      v
Backend Services
```

Primarily north-south API concerns:

- API keys
- Consumer authentication
- Quotas
- API rate limits
- API lifecycle

Service mesh:

```text
Service A
   |
 Envoy
   |
 Envoy
   |
Service B
```

Primarily east-west concerns:

- mTLS
- Workload identity
- Service routing
- Resilience
- Service-to-service telemetry

Enterprise design may use both:

```text
Internet
   |
API Gateway
   |
Istio Ingress Gateway
   |
Service A
   |
Istio Mesh
   |
Service B
```

---

# 44. OpenShift Service Mesh Angle

On OpenShift, service mesh should be connected to concepts already used by the platform.

```text
OpenShift external ingress
        |
        v
Istio Ingress Gateway
        |
        v
Mesh workloads
```

Security layers are different:

```text
OpenShift SCC
= how a pod is allowed to run

NetworkPolicy
= network connectivity

Istio AuthorizationPolicy
= service/request authorization
```

Istio CNI is especially useful in hardened OpenShift environments because it avoids privileged traffic-redirection setup inside every application pod.

---

# 45. When NOT to Use Istio

Do not say every Kubernetes cluster needs a service mesh.

Istio adds:

- More proxies/components
- Resource consumption
- More CRDs
- Another policy layer
- More debugging complexity
- More operational ownership

Istio becomes valuable when requirements justify that cost:

- Large microservice estate
- Zero-trust workload identity
- Consistent mTLS
- Advanced traffic management
- Canary releases
- Cross-team security policy
- Deep service observability
- Multi-cluster service communication

Senior answer:

> A service mesh should solve a real platform problem, not be introduced because it is fashionable.

---

# 46. Most Important CRDs to Remember

| Resource | Memory |
|---|---|
| VirtualService | WHERE traffic goes |
| DestinationRule | HOW destination traffic behaves |
| Gateway | Edge listener |
| ServiceEntry | External service registry entry |
| PeerAuthentication | Workload mTLS |
| RequestAuthentication | JWT validation |
| AuthorizationPolicy | Access decision |

These seven cover most interview questions.

---

# 47. Important Interview Traps

## Does traffic pass through istiod?

No.

```text
istiod = control plane
Envoy / ztunnel = data plane
```

## VirtualService vs DestinationRule?

```text
VirtualService = WHERE
DestinationRule = HOW
```

## Does mTLS identify the user?

No.

```text
mTLS -> workload identity
JWT  -> end-user identity
```

## NetworkPolicy vs AuthorizationPolicy?

```text
NetworkPolicy -> network connectivity
AuthorizationPolicy -> identity/application-aware authorization
```

## Istio vs API Gateway?

```text
API Gateway -> mainly north-south
Istio -> mainly east-west
```

## Is tracing completely automatic?

No. Applications still need to propagate trace context.

## If istiod dies, does traffic stop immediately?

No. Existing proxies keep their last known config, but control-plane functions degrade.

---

# 48. 3-Minute Interview Answer

> Istio is a service mesh used to move common service-to-service concerns such as mTLS, workload identity, traffic routing, retries, timeouts, circuit breaking and telemetry out of application code and into the platform.
>
> Architecturally it has a control plane and a data plane. The control plane is `istiod`, which discovers services, processes Istio configuration, distributes proxy configuration through xDS and manages workload certificates. Application traffic does not flow through istiod.
>
> In traditional sidecar mode, the data plane consists of Envoy proxies deployed alongside the application containers. A request from Service A is intercepted by its Envoy, routing and resilience policies are applied, mTLS is established with Service B's Envoy, authorization is evaluated and the request is then forwarded to Service B.
>
> For traffic management, `VirtualService` determines where traffic should go, while `DestinationRule` determines how traffic behaves once a destination is selected, including subsets, load balancing, connection pools and outlier detection. This supports canary deployment, retries, timeouts, fault injection and traffic mirroring.
>
> For security, `PeerAuthentication` controls workload mTLS, `RequestAuthentication` validates JWTs and `AuthorizationPolicy` decides which workloads or users can access a service. Workload identity is normally tied to Kubernetes ServiceAccounts rather than dynamic pod IPs.
>
> Istio also provides service metrics, access logs and tracing through the data-plane proxies, although applications still need to propagate trace context for complete distributed traces.
>
> Operationally, I would deploy multiple istiod and ingress-gateway replicas, use topology spread and PDBs, and troubleshoot from Kubernetes first before using `istioctl analyze`, `proxy-status` and `proxy-config` to inspect what Envoy actually received.
>
> Modern Istio also supports ambient mode, where ztunnel provides the L4 secure overlay at node level and optional waypoint proxies provide L7 functionality.
>
> I would introduce Istio only when service count, security, traffic-management or observability requirements justify the additional operational complexity.

---

# 49. Final Architecture Diagram

```text
                              ISTIO SERVICE MESH

                              CONTROL PLANE

                                  istiod
                                    |
                  +-----------------+-----------------+
                  |                 |                 |
                 xDS          Certificates        Discovery
                  |                 |                 |
                  +-----------------+-----------------+
                                    |
                                    v

                               DATA PLANE

External User
     |
     v
External LB
     |
     v
Istio Ingress Gateway
     |
     v
+----------------------+                 +----------------------+
| Frontend Pod         |                 | Payment Pod          |
|                      |                 |                      |
| Frontend App         |                 | Payment App          |
|      |               |                 |      ^               |
|      v               |                 |      |               |
| Envoy Sidecar        |================>| Envoy Sidecar        |
+----------------------+      mTLS       +----------------------+
          |                                     |
          | VirtualService                      | AuthorizationPolicy
          | DestinationRule                     | PeerAuthentication
          | Retry / Timeout                     |
          | Circuit Breaker                     |
          | Telemetry                           |
          +------------------+------------------+
                             |
                     Metrics / Logs / Traces

Security:
ServiceAccount -> Workload identity -> Certificate -> mTLS

Traffic:
VirtualService  = WHERE
DestinationRule = HOW

Authentication:
PeerAuthentication    = workload mTLS
RequestAuthentication = JWT
AuthorizationPolicy   = access decision
```

---

# 50. Final Revision — 10 Things to Remember

```text
1. Istio exists to remove repeated networking/security logic from applications.

2. istiod = control plane.

3. Envoy / ztunnel = data plane.

4. Application traffic does NOT pass through istiod.

5. VirtualService = WHERE.

6. DestinationRule = HOW.

7. PeerAuthentication = mTLS.

8. RequestAuthentication = JWT.

9. AuthorizationPolicy = access control.

10. Troubleshoot:
    Kubernetes -> analyze -> proxy-status -> proxy-config.
```
