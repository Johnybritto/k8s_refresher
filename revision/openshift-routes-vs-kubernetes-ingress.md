# OpenShift Routes vs Kubernetes Ingress

## Simple Meaning

Both **OpenShift Route** and **Kubernetes Ingress** are mainly used to expose HTTP/HTTPS applications running inside a cluster to users outside the cluster.

The easiest way to remember the difference is:

```text
Ingress = Kubernetes standard
Route   = OpenShift-specific
```

A Route is tightly integrated with OpenShift's built-in HAProxy-based Ingress Controller and exposes some OpenShift-specific routing and TLS capabilities directly.

---

## Simple Traffic Flow

### Kubernetes Ingress

```text
User
 |
 v
Ingress Controller
 |
 v
Ingress rule
 |
 v
Service
 |
 v
Pods
```

### OpenShift Route

```text
User
 |
 v
OpenShift HAProxy Ingress Controller / Router
 |
 v
Route
 |
 v
Service
 |
 v
Pods
```

---

## Important OpenShift Behaviour

OpenShift supports both Kubernetes `Ingress` objects and OpenShift `Route` objects.

When you create a Kubernetes `Ingress` object in OpenShift, OpenShift automatically creates and manages corresponding `Route` objects for it. When the Ingress is deleted, those managed Routes are deleted as well.

This is useful for applications or Helm charts that already use standard Kubernetes Ingress resources.

```text
Kubernetes Ingress
        |
        v
OpenShift automatically manages Route object(s)
        |
        v
HAProxy Ingress Controller
        |
        v
Service -> Pods
```

Important: do not manually modify a Route that is managed from an Ingress and expect those changes to be permanent. The Ingress is the source configuration for that generated Route.

---

## Direct Comparison

| Feature | OpenShift Route | Kubernetes Ingress |
|---|---|---|
| API type | OpenShift-specific | Kubernetes standard |
| Portability | Mainly OpenShift | Portable across Kubernetes platforms |
| Controller in OpenShift | Built-in HAProxy-based Ingress Controller | Uses the same OpenShift ingress infrastructure when handled by OpenShift |
| HTTP/HTTPS routing | Yes | Yes |
| Host/path routing | Yes | Yes |
| Basic TLS termination | Yes | Yes |
| Edge / passthrough / re-encrypt | Native Route concepts | On OpenShift, Route-specific annotations can be used when converting Ingress to Route |
| Weighted backends | Native Route backend weights | Not part of the standard Ingress API; usually controller-specific extensions are needed |
| Wildcard hosts | Supported, subject to OpenShift wildcard policy | Standard Kubernetes Ingress also supports wildcard hosts |
| Automatic hostname | A Route can have its hostname assigned from the Ingress Controller domain | Upstream Kubernetes Ingress can omit host; however, OpenShift's Ingress-to-Route conversion requires an explicit host |
| Best fit | OpenShift-native applications and advanced Route features | Portable manifests and Helm charts |

---

# TLS Modes in OpenShift Routes

One of the most useful Route features is the native TLS termination model.

## 1. Edge Termination

TLS ends at the OpenShift router.

```text
Client -- HTTPS --> OpenShift Router -- HTTP --> Service/Pod
```

Use this when the router should handle the certificate and backend traffic does not need to remain encrypted.

---

## 2. Passthrough Termination

The router does not decrypt the TLS traffic. TLS terminates at the application/backend.

```text
Client -- HTTPS --> OpenShift Router -- HTTPS --> Pod
                                      TLS ends here
```

Use this when the application itself must own or terminate the TLS connection.

---

## 3. Re-encrypt Termination

TLS terminates at the router, and the router establishes another encrypted TLS connection to the backend.

```text
Client -- HTTPS --> OpenShift Router -- HTTPS --> Pod
         TLS #1                       TLS #2
```

Use this when you want encrypted traffic from the client to the router and also from the router to the backend.

### Memory Line

```text
Edge       = TLS ends at router.
Passthrough = TLS ends at application.
Re-encrypt = TLS ends at router and starts again to backend.
```

---

# Weighted Traffic Splitting

OpenShift Route supports multiple backend Services with weights.

Example:

```text
Route
 |
 +--> service-v1 weight 90
 |
 +--> service-v2 weight 10
```

This can be useful for A/B testing or canary-style traffic distribution.

OpenShift supports primary and alternate backend Services with weights. The router distributes traffic according to those relative weights.

Example concept:

```yaml
spec:
  to:
    kind: Service
    name: app-v1
    weight: 90
  alternateBackends:
    - kind: Service
      name: app-v2
      weight: 10
```

This is an OpenShift Route capability and is not part of the standard Kubernetes Ingress API.

---

# Hostname Behaviour — Important Interview Detail

There are two different facts that are easy to mix up.

## Upstream Kubernetes

A Kubernetes Ingress rule does **not always require a host**. If the host is omitted, the rule can match traffic reaching the Ingress controller based on the configured paths/default backend.

## OpenShift Ingress-to-Route Conversion

When using a Kubernetes Ingress in OpenShift and having OpenShift create a corresponding Route, Red Hat documents an explicit `rules.host` as mandatory for this conversion path.

So say:

```text
Kubernetes Ingress itself can work without a host.
OpenShift's managed Ingress-to-Route conversion expects an explicit host.
```

Do not say that Kubernetes Ingress universally requires a hostname.

---

# Wildcard Hosts — Correction

Wildcard routing is **not unique to OpenShift Routes**.

Standard Kubernetes Ingress supports wildcard hostnames such as:

```text
*.example.com
```

OpenShift Route also supports wildcard/subdomain routes, but the OpenShift Ingress Controller has a `wildcardPolicy` that controls whether wildcard Routes are admitted.

Therefore the useful distinction is not simply "Route supports wildcard and Ingress does not".

---

# Example: Kubernetes Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp
            port:
              number: 8080
```

Traffic:

```text
app.example.com
       |
       v
Ingress
       |
       v
myapp Service
       |
       v
Pods
```

---

# Example: OpenShift Route

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: myapp
spec:
  host: app.example.com
  to:
    kind: Service
    name: myapp
  port:
    targetPort: 8080
  tls:
    termination: edge
```

Traffic:

```text
app.example.com
       |
       v
OpenShift Router
       |
       v
Route
       |
       v
myapp Service
       |
       v
Pods
```

---

# When Should I Use Route?

Use OpenShift Route when:

- The application is staying on OpenShift.
- You want OpenShift-native TLS modes such as edge, passthrough, or re-encrypt.
- You want native weighted Route backends.
- You want close integration with the OpenShift HAProxy Ingress Controller.
- Portability to non-OpenShift Kubernetes clusters is not a major requirement.

---

# When Should I Use Ingress?

Use Kubernetes Ingress when:

- The same Helm chart or manifest should work across OpenShift, EKS, AKS, GKE, or other Kubernetes platforms.
- Your application already uses standard Kubernetes Ingress resources.
- You do not require Route-specific capabilities directly in the manifest.

On OpenShift, the cluster can create managed Route objects from that Ingress.

---

# Practical Migration Example

Suppose an application currently runs on EKS:

```text
Internet
   |
   v
Ingress
   |
   v
Service
   |
   v
Pods
```

If you migrate it to OpenShift and want maximum manifest portability, you can keep the Kubernetes Ingress:

```text
Existing Ingress manifest
        |
        v
OpenShift creates managed Route(s)
        |
        v
HAProxy Ingress Controller
        |
        v
Service -> Pods
```

If later you require OpenShift-specific features such as native re-encrypt TLS or weighted Route backends, you can consider using Route directly.

---

# Interview Traps

## Trap 1

Wrong:

```text
Ingress and Route are completely different.
```

Better:

```text
They solve the same main problem: exposing HTTP/HTTPS services externally. Ingress is the Kubernetes-standard API; Route is OpenShift-specific and provides tighter OpenShift integration and Route-specific capabilities.
```

## Trap 2

Wrong:

```text
Ingress always requires a host.
```

Correct:

```text
Upstream Kubernetes Ingress can omit a host, but OpenShift's Ingress-to-Route conversion requires an explicit host.
```

## Trap 3

Wrong:

```text
Only OpenShift Route supports wildcard hostnames.
```

Correct:

```text
Both can support wildcard hosts. OpenShift additionally controls wildcard Route admission through its Ingress Controller wildcard policy.
```

## Trap 4

Wrong:

```text
Creating an Ingress on OpenShift means I need to deploy NGINX.
```

Correct:

```text
OpenShift ships with an HAProxy-based Ingress Controller managed by the Ingress Operator and can handle both Route and Kubernetes Ingress resources.
```

---

# Interview-Ready Answer

> OpenShift Route and Kubernetes Ingress solve the same primary problem: exposing HTTP and HTTPS services outside the cluster. Ingress is the upstream Kubernetes standard, so it is more portable across platforms such as EKS, AKS, GKE, and OpenShift. Route is an OpenShift-specific API that integrates tightly with OpenShift's HAProxy-based Ingress Controller and provides native features such as edge, passthrough, and re-encrypt TLS termination and weighted service backends. OpenShift also supports Kubernetes Ingress resources and automatically manages corresponding Route objects for them. For portable Helm charts I would normally keep Ingress, while for an OpenShift-only workload that needs Route-specific capabilities I would use Route directly.

---

# Memory Line

```text
Ingress = portable Kubernetes standard.
Route   = OpenShift-native external routing with richer OpenShift-specific features.
```

---

# One More Modern Kubernetes Note

Kubernetes Ingress is stable but the upstream Kubernetes project has frozen further development of the Ingress API and recommends Gateway API for new capabilities.

That does not make Ingress obsolete; existing Ingress workloads continue to work. It simply means Gateway API is the newer direction for advanced Kubernetes traffic management.

---

# Official References

- Red Hat OpenShift Container Platform 4.20 — Ingress and load balancing / Routes
  - https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/ingress_and_load_balancing/ingress_and_load_balancing
- Red Hat OpenShift Container Platform 4.20 — Networking overview
  - https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html-single/networking_overview/index
- Red Hat OpenShift Container Platform 4.20 — Ingress Operator
  - https://docs.redhat.com/en/documentation/openshift_container_platform/4.20/html/networking_operators/configuring-ingress
- Kubernetes — Ingress
  - https://kubernetes.io/docs/concepts/services-networking/ingress/
- Kubernetes — Ingress Controllers
  - https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
