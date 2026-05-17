# Day 2 Exercise Review – Stuck Deployment Rollout

## Exercise

A Deployment rollout is stuck. The new Pods are not becoming Ready. How will you troubleshoot it step by step?

---

## User Answer

```text
check the deployment status , then replicaset status and then the pod status and then check the pod logs or describe the pods for there errors
```

---

## Review

This is a good high-level answer. The order is correct:

```text
Deployment -> ReplicaSet -> Pod -> describe/logs
```

That is the right troubleshooting path because a Deployment rollout issue can happen at different layers:

- Deployment may not be progressing.
- ReplicaSet may not be creating Pods.
- Pods may be Pending, CrashLoopBackOff, ImagePullBackOff, or Running but not Ready.
- Readiness probe may be failing even if the container is running.

---

## What To Add For Senior-Level Answer

When the question says **new Pods are not becoming Ready**, specifically check:

1. Deployment rollout status and conditions.
2. ReplicaSet status: old RS vs new RS.
3. Pod status: Pending, Running, CrashLoopBackOff, ImagePullBackOff.
4. Pod events using `kubectl describe pod`.
5. Readiness probe failure reason.
6. Application logs.
7. Previous logs if container restarted.
8. Service port and container port mismatch.
9. ConfigMap/Secret/env issues.
10. Dependency failures such as DB/API not reachable.
11. Resource limits, CPU throttling, OOMKilled.
12. Admission/policy/quota failures.
13. Rollout settings such as `maxSurge` and `maxUnavailable`.

---

## Strong Interview Answer

```text
For a stuck Deployment rollout where new Pods are not becoming Ready, I would start with kubectl rollout status and kubectl describe deployment to check Deployment conditions and rollout progress. Then I would check the ReplicaSets to confirm whether the new ReplicaSet was created and whether it is scaling up correctly.

Next, I would check the new Pods. If they are Pending, I would inspect scheduler events for resource, affinity, taint, or PVC issues. If they are ImagePullBackOff, I would verify image name, tag, registry access, and imagePullSecrets. If they are CrashLoopBackOff, I would check current and previous logs, exit code, config, secrets, dependencies, and resource limits.

If the Pods are Running but not Ready, I would focus on readiness probe failures, application port binding, dependency health, wrong target port, slow startup, and whether a startupProbe is needed. I would also check events, resource pressure, and recent config changes. Finally, I would review rollout strategy values like maxSurge and maxUnavailable and confirm there is enough cluster capacity for the rollout.
```

---

## Useful Commands

```bash
kubectl rollout status deployment/<deployment> -n <namespace>
kubectl describe deployment <deployment> -n <namespace>
kubectl get rs -n <namespace>
kubectl describe rs <replicaset> -n <namespace>
kubectl get pods -n <namespace> -o wide
kubectl describe pod <pod> -n <namespace>
kubectl logs <pod> -n <namespace>
kubectl logs <pod> -n <namespace> --previous
kubectl top pod <pod> -n <namespace>
```

---

## Final Memory Line

```text
For rollout stuck: Deployment conditions -> ReplicaSet -> Pod state -> Events -> Readiness probe -> Logs -> Config/Secrets -> Resources -> Dependencies -> Rollout strategy.
```
