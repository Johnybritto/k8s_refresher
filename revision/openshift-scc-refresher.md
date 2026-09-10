# OpenShift SCC Refresher

## What is SCC?

**SCC = Security Context Constraints**.

It controls what security privileges a pod is allowed to use in OpenShift.

Think of it as:

```text
Pod asks for privileges
        ↓
SCC checks whether they are allowed
        ↓
Allowed → Pod runs
Denied  → Pod is rejected
```

Common controls include:

- Running as root / specific UID
- Privileged containers
- Linux capabilities
- hostNetwork / hostPID / hostPath
- SELinux
- Allowed volume types

---

## Key SCCs

### restricted-v2
Default restrictive SCC for normal applications.

### anyuid
Allows a workload to run with any UID, including UID 0 (root), if the pod requests it.

### privileged
Allows very broad host-level privileges. Use only when genuinely required by node-level agents, storage/network components, etc.

---

## `runAsUser: 0`

```yaml
securityContext:
  runAsUser: 0
```

Means:

```text
UID 0 = Linux root user
```

So the container is requesting to run as root.

This setting alone is not enough in OpenShift. The pod's ServiceAccount must be allowed to use an SCC that permits UID 0, such as `anyuid`.

---

## Why ServiceAccount is involved

A ServiceAccount is the pod's identity inside Kubernetes/OpenShift.

Example:

```yaml
spec:
  serviceAccountName: myapp-sa
```

Flow:

```text
Pod
 ↓
runs as ServiceAccount: myapp-sa
 ↓
OpenShift checks which SCCs myapp-sa may use
 ↓
Pod requests runAsUser: 0
 ↓
If myapp-sa can use anyuid → allowed
If not → rejected
```

---

## Example: Grant anyuid to a ServiceAccount

Create ServiceAccount:

```bash
oc create sa myapp-sa -n myproject
```

Grant anyuid:

```bash
oc adm policy add-scc-to-user anyuid \
  -z myapp-sa \
  -n myproject
```

Deployment snippet:

```yaml
spec:
  template:
    spec:
      serviceAccountName: myapp-sa
      containers:
      - name: app
        image: nginx
        securityContext:
          runAsUser: 0
```

---

## anyuid vs privileged

```text
anyuid
= "You may run as root / arbitrary UID"

privileged
= "You may run with broad host-level privileges"
```

A pod can need `anyuid` without needing `privileged`.

Example `privileged` request:

```yaml
securityContext:
  privileged: true
```

Typical examples:

- Legacy vendor app requiring root → `anyuid`
- Dynatrace/monitoring agent requiring host-level access → suitable elevated SCC, sometimes privileged depending on deployment mode
- Storage or networking DaemonSet requiring host devices/namespaces → privileged or carefully scoped custom SCC

---

## Useful Commands

Check which SCC a running pod received:

```bash
oc get pod <pod-name> -n <namespace> \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
```

Check whether a ServiceAccount can use `anyuid`:

```bash
oc auth can-i use scc/anyuid \
  --as=system:serviceaccount:<namespace>:<serviceaccount>
```

Check who can use an SCC:

```bash
oc adm policy who-can use scc anyuid
```

---

## Interview Answer

> SCC is an OpenShift security policy that controls what privileges a pod can run with. The pod's ServiceAccount provides its identity, and OpenShift checks whether that identity is allowed to use an SCC that permits the requested security settings. For example, `runAsUser: 0` requests root, and a ServiceAccount would need permission to use an SCC such as `anyuid`. I would always use the least-privileged SCC required instead of assigning `privileged` by default.
