# Day 3 Exercise Review – Pod IP Works but Service Fails

## Exercise

A Pod is reachable directly using its Pod IP, but the application is not reachable through the Kubernetes Service. How will you troubleshoot this step by step?

---

## User Answer

```text
check the service selector and pod labels whether they are matching 
check the endpoint slice whether they are showing pod IP's 
check the pod readiness 
check the networkpolicy , coredns  and CNI pods and kubeproxy whether they are working properly
```

---

## Review

This is a strong answer. You correctly covered the most important checks:

- Service selector
- Pod labels
- EndpointSlice
- Pod readiness
- NetworkPolicy
- CoreDNS
- CNI
- kube-proxy

The most important first three checks are exactly right:

```text
Selector/labels -> EndpointSlice -> Readiness
```

---

## What To Add For Senior-Level Answer

If Pod IP works but Service fails, the app is likely running, so focus on the Service path.

Add these checks:

1. Service port and targetPort mapping.
2. Whether EndpointSlice has Ready endpoints.
3. Whether the application is listening on the targetPort.
4. Whether the Service is in the correct namespace.
5. Whether DNS is the issue or Service routing is the issue.
6. Whether NetworkPolicy is blocking the client Pod, not the Service object.
7. kube-proxy/eBPF dataplane only after selector, endpoint, readiness, and port checks.

Important distinction:

```text
If ClusterIP works but DNS name fails, check CoreDNS.
If Pod IP works but ClusterIP fails, check Service selector, EndpointSlice, targetPort, readiness, NetworkPolicy, and dataplane.
```

---

## Strong Interview Answer

```text
If a Pod is reachable directly by Pod IP but not through the Kubernetes Service, I would first check the Service selector and compare it with the Pod labels. If the selector does not match, the Service will not have endpoints.

Next, I would check EndpointSlice to confirm whether the Pod IPs are registered as Ready endpoints for the Service. If EndpointSlice is empty, I would check Pod readiness because Running Pods that are not Ready are normally not used for Service traffic.

Then I would verify the Service port and targetPort mapping and confirm the application is listening on the expected container port. After that, I would check NetworkPolicy because traffic comes from the source Pod, not from the Service object. If DNS name fails but ClusterIP works, I would check CoreDNS. If all Kubernetes objects look correct, I would check kube-proxy, CNI, or eBPF dataplane health.
```

---

## Useful Commands

```bash
kubectl get svc <service> -n <namespace>
kubectl describe svc <service> -n <namespace>
kubectl get pods -n <namespace> --show-labels
kubectl get endpointslice -n <namespace>
kubectl describe endpointslice <endpoint-slice> -n <namespace>
kubectl describe pod <pod> -n <namespace>
kubectl get networkpolicy -n <namespace>
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup <service>.<namespace>.svc.cluster.local
```

---

## Final Memory Line

```text
Pod IP works but Service fails: selector/labels -> EndpointSlice -> readiness -> port/targetPort -> NetworkPolicy -> DNS only if name fails -> kube-proxy/CNI/eBPF.
```
