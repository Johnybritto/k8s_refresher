# Ingress vs Gateway API in Kubernetes — and Why ingress-nginx Was Retired

> Interview revision notes | Updated: 25 October 2026

## 1. Ingress vs Gateway API

Both manage traffic entering Kubernetes, but **Gateway API is a more flexible, modern replacement for the Ingress API**.

### Architecture comparison

**Traditional Ingress**

```text
Client
  ↓
External Load Balancer
  ↓
NGINX Ingress Controller
  ↓  (reads Ingress host/path rules and configures NGINX)
Kubernetes Service
  ↓
Application Pods
```

**Gateway API**

```text
Client
  ↓
External Load Balancer
  ↓
Gateway implementation / data plane  (e.g., Envoy, Cilium, Istio, NGINX Gateway Fabric)
  ↑  configured by Gateway API controller using:
GatewayClass → Gateway (listeners, TLS, ports)
                      +
                 HTTPRoute (host/path/header/traffic weights)
  ↓
Kubernetes Service
  ↓
Application Pods
```

**Key difference:** Ingress combines basic application routing in one resource. Gateway API separates concerns:

- **GatewayClass:** selects a controller/implementation.
- **Gateway:** defines the traffic-handling infrastructure, listeners, ports, and TLS configuration.
- **HTTPRoute:** defines application-level routing to Services.

> Gateway API is an **API specification**, not itself an installed controller or proxy. A compatible controller and data plane are still required.

## 2. Feature comparison

| Feature | Ingress | Gateway API |
|---|---|---|
| API design | Single Ingress resource | GatewayClass, Gateway, HTTPRoute |
| Routing | Host and path | Host, path, headers, query parameters |
| Traffic splitting | Typically controller-specific configuration | HTTPRoute weighted backends |
| Canary deployment | Often controller annotations | Native HTTPRoute weights |
| TLS termination | Supported | Supported with flexible Gateway/listener configuration |
| TCP/UDP | Not part of standard Ingress API | TCPRoute/UDPRoute where supported (experimental API types) |
| Team ownership | Less separation in the API model | Platform teams own Gateways; app teams own Routes |
| Extensibility | Controller-specific annotations | Standardized resource and policy extension model |

**Note:** Actual features depend on the chosen Gateway API controller's implementation and conformance.

## 3. Why was the community NGINX Ingress Controller retired?

**The Kubernetes community-maintained `kubernetes/ingress-nginx` controller was retired on 24 March 2026.**

Main reasons:

1. **Security concerns:** Powerful NGINX configuration customizations and legacy architectural decisions made its security model hard to maintain.
2. **Limited maintainers:** A critical and widely used project had effectively been sustained by only one or two maintainers.
3. **Technical limitations:** Dependence on custom annotations, accumulated technical debt, and difficult-to-evolve design.

After retirement, **no new releases, bug fixes, or security patches** are provided. Existing deployments continue running, but staying on an unmaintained controller increases security risk.

**Important distinction:**

- **NGINX the web server/reverse proxy is not retired.**
- **F5 NGINX Ingress Controller** is a separate project; do not confuse it with the retired community `ingress-nginx` controller.
- **Kubernetes Ingress API still exists**, although it is frozen; the retirement applies to the community controller, not the API itself.

## 4. What replaces ingress-nginx?

Gateway API needs an implementation. Examples include:

- Envoy Gateway
- Cilium Gateway API
- Istio
- NGINX Gateway Fabric

### Example: 90/10 canary rollout

```text
                       Gateway
                          ↓
                      HTTPRoute
                        /    \
                    90%      10%
                     ↓        ↓
                 Service v1  Service v2
                     ↓        ↓
                 Stable Pods  Canary Pods
```

The split can be configured through **HTTPRoute `backendRefs.weight`**, without NGINX-specific canary annotations:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  parentRefs:
    - name: app-gateway
  rules:
    - backendRefs:
        - name: my-app-v1
          port: 80
          weight: 90
        - name: my-app-v2
          port: 80
          weight: 10
```

**Note:** Weights are relative; 90 and 10 result in a 90%/10% distribution here.

## 5. What should you do in EKS, AKS, or on-prem Kubernetes?

First identify whether the cluster runs the retired community controller:

```bash
kubectl get pods -A -l app.kubernetes.io/name=ingress-nginx
```

Then plan a migration to a supported Gateway API implementation or another maintained Ingress controller.

Suggested migration flow:

```text
Inventory current Ingress resources and annotations
                     ↓
Choose a supported Gateway API implementation
                     ↓
Map Ingress rules / TLS / annotations to Gateway + HTTPRoute
                     ↓
Deploy and validate in parallel (where possible)
                     ↓
Test routing, TLS, auth, rewrites, traffic policies and rollback
                     ↓
Cut over traffic; monitor; retire old controller
```

The Kubernetes **ingress2gateway** tool can help convert configurations, but conversion is not guaranteed to be one-to-one. Pay particular attention to controller-specific annotations and security settings.

## 6. Interview takeaway

> **Ingress** is the older Kubernetes API for basic HTTP/HTTPS routing. **Gateway API** separates infrastructure configuration from application routing and offers standardized advanced traffic management. The **community ingress-nginx controller** was retired because of security, maintainability, and staffing challenges — not because NGINX itself has been discontinued.

## Official references

- [Kubernetes: Ingress NGINX retirement announcement](https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/)
- [Kubernetes: Confirmed retirement on March 24, 2026](https://kubernetes.io/blog/2026/03/30/kubernetes-v1-36-sneak-peek/)
- [Kubernetes: Gateway API concepts](https://kubernetes.io/docs/concepts/services-networking/gateway/)
- [Gateway API: Traffic splitting with HTTPRoute](https://gateway-api.sigs.k8s.io/guides/user-guides/traffic-splitting/)
- [Kubernetes: ingress2gateway 1.0 migration assistant](https://kubernetes.io/blog/2026/03/20/ingress2gateway-1-0-release/)
