# Day 6 Follow-up – Authentication, OpenShift Login, Default Kubernetes Service, NetworkPolicy & etcd

Status: Follow-up revision notes from Day 6 discussion.

---

## 1. Default `kubernetes` Service in the `default` Namespace

When you run:

```bash
kubectl get svc -n default
```

You usually see:

```text
NAME         TYPE        CLUSTER-IP   PORT(S)
kubernetes   ClusterIP   10.96.0.1    443/TCP
```

This Service is created automatically by Kubernetes.

### Purpose

It gives Pods inside the cluster a stable internal address to reach the Kubernetes API Server.

```text
Pod / Operator / Controller
  -> https://kubernetes.default.svc
  -> kubernetes Service ClusterIP
  -> kube-apiserver
```

Pods, operators, controllers, ArgoCD, cert-manager, External Secrets Operator, and other in-cluster tools may use it to talk to the API Server.

### Important Point

This Service is in the `default` namespace, but it can be reached from other namespaces using:

```text
kubernetes.default.svc
kubernetes.default.svc.cluster.local
```

Breakdown:

```text
kubernetes    = Service name
default       = namespace where the Service exists
svc           = Kubernetes Service DNS zone
cluster.local = cluster DNS suffix
```

Namespace is not automatically a network boundary. Services in one namespace can be reached from another namespace if network policies allow it.

### Authentication Still Applies

Reaching the API Server network endpoint does not mean the Pod has permission.

A Pod usually authenticates using its ServiceAccount token mounted at:

```text
/var/run/secrets/kubernetes.io/serviceaccount/token
```

RBAC then decides what the Pod can do.

### Interview Answer

```text
The default kubernetes Service in the default namespace is a ClusterIP Service automatically created by Kubernetes to provide a stable in-cluster endpoint for the Kubernetes API Server. Pods and controllers can reach it using kubernetes.default.svc instead of depending on control-plane node IPs. Network reachability is separate from authentication and authorization; the Pod still needs a ServiceAccount token and RBAC permission.
```

---

## 2. Cross-Namespace Access to the Default Kubernetes Service

A Pod in namespace `prod` can reach the API Server using:

```text
https://kubernetes.default.svc
```

This works because the DNS name explicitly identifies the Service and namespace.

```text
service.namespace.svc.cluster.local
```

Namespaces provide logical separation for:

- Object names
- RBAC
- Quotas
- NetworkPolicy scope
- Service discovery names

But namespaces do not automatically block network traffic.

### Memory Line

```text
Namespace controls naming and policy scope. It does not automatically isolate network traffic.
A Service in another namespace can be reached as service.namespace.svc.cluster.local.
```

---

## 3. Default-Deny NetworkPolicy and API Server Access

If default deny is enabled only for ingress:

```yaml
policyTypes:
- Ingress
```

Pods can still make outbound calls unless another egress policy blocks them.

If default deny is enabled for egress:

```yaml
policyTypes:
- Egress
```

or:

```yaml
policyTypes:
- Ingress
- Egress
```

then Pods cannot call DNS, internal Services, external APIs, or `kubernetes.default.svc` unless explicitly allowed.

### Minimum Egress Usually Needed

Most namespaces need egress to CoreDNS:

```text
CoreDNS UDP/TCP 53
```

Only selected workloads should get egress to the Kubernetes API Server:

```text
kubernetes.default.svc:443
```

Examples of workloads that may need API Server access:

- Operators
- Controllers
- ArgoCD / Flux
- cert-manager
- External Secrets Operator
- Monitoring agents
- CI/CD agents inside cluster
- Custom apps watching Kubernetes resources

Normal business apps usually should not need direct API Server access.

### Important Distinction

```text
NetworkPolicy allows or blocks the network path.
RBAC allows or denies API actions.
Both are required for successful API access.
```

### Interview Answer

```text
If default deny is enabled only for ingress, Pods may still reach the API Server because egress is not blocked. If default-deny egress is enabled, we must explicitly allow DNS and allow API Server egress on TCP 443 only for workloads that need it, such as operators or GitOps controllers. RBAC is still required separately because NetworkPolicy only controls connectivity, not API permissions.
```

---

## 4. AD Group Authentication and RBAC Flow

Example:

```text
AD group: ABC
User: johny@company.com
Permission: create pods in namespace prod
```

### Flow

```text
User logs in through enterprise identity provider
  -> token contains username and groups
  -> kubectl sends token to API Server
  -> API Server authenticates token
  -> API Server extracts username and groups
  -> RBAC checks RoleBinding/ClusterRoleBinding
  -> Admission controllers run
  -> object is stored in etcd
```

### Authentication

Authentication answers:

```text
Who are you?
```

The API Server validates the token and extracts identity:

```text
User: johny@company.com
Groups:
- ABC
- system:authenticated
```

Kubernetes does not blindly trust that the user says they are in group ABC. It trusts the signed token from the configured identity provider.

### Authorization

RBAC checks whether group ABC has permission.

Example Role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: pod-creator
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create", "get", "list"]
```

Example RoleBinding:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: abc-pod-creator-binding
  namespace: prod
subjects:
- kind: Group
  name: ABC
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-creator
  apiGroup: rbac.authorization.k8s.io
```

### Direct Pod Creation vs Deployment Creation

Creating a Pod directly requires:

```text
apiGroups: [""]
resources: ["pods"]
verbs: ["create"]
```

Creating a Deployment requires:

```text
apiGroups: ["apps"]
resources: ["deployments"]
verbs: ["create"]
```

Important interview line:

```text
create pods is not the same as create deployments.apps.
```

### Interview Answer

```text
When a user from AD group ABC creates a Pod, kubectl sends the request with the user's token. The API Server authenticates the token, extracts the username and group claims, and then RBAC checks whether group ABC is bound to a Role or ClusterRole allowing create pods in that namespace. If RBAC allows it, admission controllers run. If admission passes, the Pod object is stored in etcd and the scheduler/kubelet workflow starts. If the user creates a Deployment instead, the required permission is create deployments.apps, not create pods.
```

---

## 5. First-Time Login Without kubeconfig

If there is no kubeconfig, the user cannot directly use `kubectl`, even if AD group and RBAC permissions are already configured.

`kubectl` needs:

```text
1. API Server URL
2. Cluster CA/trust information
3. Authentication method/token/client certificate
4. Context mapping cluster + user + namespace
```

These are normally stored in kubeconfig.

### Common Enterprise First-Login Models

1. Platform/admin provides kubeconfig.
2. Login CLI generates kubeconfig.
3. User logs into a web portal and downloads kubeconfig.
4. OpenShift users run `oc login`.

### Memory Line

```text
Permission decides what the user can do.
kubeconfig tells kubectl where and how to connect.
Both are required.
```

---

## 6. What Happens During `oc login`

`oc login` authenticates the user to OpenShift and writes token/cluster/context information into kubeconfig.

Example:

```bash
oc login https://api.cluster.company.com:6443
```

or:

```bash
oc login https://api.cluster.company.com:6443 -u johny -p password
```

### Step-by-Step Flow

1. `oc` contacts the OpenShift API endpoint.
2. OpenShift OAuth server starts the authentication flow.
3. Credentials are validated against the configured identity provider, such as AD, LDAP, OIDC, or SAML.
4. OpenShift maps the external identity to an OpenShift user and groups.
5. OpenShift OAuth server issues an access token.
6. `oc` stores the cluster, context, and token in kubeconfig.
7. Future `oc`/`kubectl` commands send the token to the API Server.
8. API Server authenticates the token and RBAC authorizes actions.

### Login Success vs Permission

`oc login` success means:

```text
Authentication successful.
Token issued.
kubeconfig updated.
```

It does not automatically mean:

```text
User can deploy to prod.
User can create Pods.
User can view Secrets.
User is cluster-admin.
```

Those are controlled by RBAC.

### Useful Commands

```bash
oc whoami
oc whoami -t
oc whoami --show-server
oc project
oc auth can-i create deployments -n prod
```

### Interview Answer

```text
When I run oc login, the oc client contacts the OpenShift API server and uses the OpenShift OAuth flow. The OAuth server validates my credentials against the configured enterprise identity provider such as AD, LDAP, OIDC, or SAML. After successful authentication, OpenShift maps the external identity to an OpenShift user and groups, then issues an OAuth access token. The oc client stores the cluster details, context, and token in kubeconfig. Future oc commands send that token to the API Server, where authentication identifies the user and groups, and RBAC decides what actions are allowed.
```

### Memory Line

```text
oc login = authenticate user + get token + update kubeconfig.
RBAC = decide what the logged-in user can do.
```

---

## 7. etcd Snapshot Backup During Kubernetes Upgrade

During a Kubernetes version upgrade, an etcd snapshot is the recovery point for the control-plane state.

etcd stores Kubernetes objects such as:

- Namespaces
- Deployments
- ReplicaSets
- Pods metadata
- Services
- ConfigMaps
- Secrets
- RBAC
- CRDs
- Ingress objects
- NetworkPolicies
- PVC/PV metadata
- Node objects

It does not back up:

- Application database rows
- Actual PersistentVolume disk data
- Container images
- External cloud resources directly

### Purpose Before Upgrade

If upgrade causes etcd corruption, API Server failure, bad object state, or accidental deletion of resources, the snapshot can restore cluster state to the pre-upgrade point.

### Important Limitation

```text
etcd snapshot restores Kubernetes cluster state.
It does not restore application data inside volumes.
```

For production recovery, you usually need:

```text
1. etcd snapshot
2. Application/PV/database backup
```

### Interview Answer

```text
During a Kubernetes upgrade, an etcd snapshot acts as the recovery point for the cluster control-plane state. If the upgrade corrupts etcd state, breaks API state, or causes accidental deletion of Kubernetes resources, we can restore the snapshot to return the cluster state to the pre-upgrade point. However, etcd snapshot does not back up actual application data inside PVs or databases, so separate data backups are required.
```

---

## 8. Does etcd Snapshot Restore the Old Kubernetes Version?

No.

An etcd snapshot restores cluster state, not Kubernetes binaries or component versions.

### etcd Snapshot Restores

- Kubernetes objects
- Cluster state
- Secrets, ConfigMaps, RBAC, CRDs, Services, Deployments, etc.

### It Does Not Restore

- kube-apiserver version
- kube-controller-manager version
- kube-scheduler version
- kubelet version
- kubeadm/kubectl packages
- CNI version
- Container runtime version

To roll back a Kubernetes version upgrade, you may need both:

```text
1. Restore etcd snapshot/state
2. Roll back Kubernetes control-plane and node component versions/manifests/packages
```

### Interview Answer

```text
Restoring an etcd snapshot alone does not restore the old Kubernetes version. etcd contains cluster state, not kube-apiserver, controller-manager, scheduler, kubelet, kubeadm, CNI, or container runtime binaries. For a real version rollback, we need to restore compatible etcd state and also roll back the Kubernetes component versions or static pod manifests/packages according to the platform procedure.
```

### Memory Line

```text
etcd snapshot = rollback cluster state.
Kubernetes downgrade = rollback binaries/manifests/packages too.
```

---

## 9. etcd Compaction

etcd uses MVCC: Multi-Version Concurrency Control.

This means etcd stores historical revisions of keys.

Example:

```text
Revision 100: Deployment replicas = 2
Revision 101: Deployment replicas = 3
Revision 102: Image changed to v2
```

Kubernetes controllers use watches based on revisions.

If etcd kept all old revisions forever, the database would grow continuously.

### What Compaction Does

Compaction removes old historical MVCC revisions.

Important:

```text
Compaction removes old revision history.
It does not delete the current Kubernetes objects.
```

Example:

```text
Before compaction: revisions 1 to 100000 stored
Compact up to revision 90000
After compaction: old history before 90000 is removed, current state remains
```

### Interview Answer

```text
etcd compaction removes old MVCC revisions from the database. Kubernetes creates many updates and etcd stores historical revisions for watches. Over time, old revisions increase database size. Compaction removes old revision history while keeping the latest cluster state intact. It controls logical database growth, but it does not necessarily shrink the physical database file on disk.
```

---

## 10. etcd Defragmentation

After compaction, old revisions are removed logically, but the physical DB file may still remain large on disk.

Defragmentation reclaims unused physical space.

```text
Compaction = removes old logical revision history.
Defrag = reclaims physical disk space.
Snapshot = backup for recovery.
```

### Production Approach

For a multi-member etcd cluster:

```text
1. Confirm snapshot/backup.
2. Check etcd health and quorum.
3. Compact old revisions.
4. Defrag one member at a time.
5. Verify health after each member.
6. Monitor API Server and etcd latency.
```

Defrag should be done carefully because it can temporarily affect the member being defragmented.

### Important Metrics

```text
etcd_mvcc_db_total_size_in_bytes
etcd_mvcc_db_total_size_in_use_in_bytes
etcd_disk_wal_fsync_duration_seconds
etcd_disk_backend_commit_duration_seconds
etcd_server_has_leader
etcd_server_leader_changes_seen_total
```

If total DB size is much larger than in-use size, defrag may help.

### Interview Answer

```text
etcd defragmentation reclaims physical disk space after compaction. Compaction removes old MVCC revisions, but the backend database file may still remain large. Defrag rewrites the backend DB to remove internal fragmentation and return unused space. In production, I would take or verify a snapshot first, check etcd health, then compact and defrag carefully, usually one member at a time, while monitoring etcd and API Server latency.
```

---

## 11. Final Memory Summary

```text
kubernetes.default.svc = stable in-cluster API Server endpoint.
Namespace does not automatically block network access.
Default-deny egress requires explicit DNS and API Server allow rules.
AD group identity comes through token claims.
RBAC maps group to Kubernetes permissions.
oc login authenticates user, gets token, and updates kubeconfig.
etcd snapshot restores cluster state, not Kubernetes binaries.
Compaction removes old etcd revision history.
Defrag reclaims physical etcd DB disk space.
```
