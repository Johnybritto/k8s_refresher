# OpenShift OAuth Outage & Recovery — Interview Guide

This guide covers how to troubleshoot and recover from OpenShift authentication/OAuth outages when normal user login is unavailable.

---

# 1. First Separate OAuth Failure from API Failure

OAuth failure does NOT automatically mean the Kubernetes/OpenShift API is down.

Typical authentication path:

```text
User
 ↓
OAuth Route
 ↓
Ingress Router
 ↓
oauth-openshift Service
 ↓
OAuth Pods
 ↓
Identity Provider
 ↓
OAuth token
 ↓
Kubernetes API :6443
```

Always answer this first:

```text
Is the API healthy and only OAuth broken?

or

Is the API itself also unavailable?
```

---

# 2. Administrative Access Paths — Do Not Mix These Up

## kubeadmin

```text
kubeadmin
= bootstrap/admin OpenShift user
= OAuth-based
= normally removed after enterprise IdP + cluster-admin are configured
```

Do not depend on kubeadmin as the long-term production break-glass path.

## Enterprise cluster-admin

```text
LDAP / AD / OIDC user
      ↓
OAuth
      ↓
cluster-admin RBAC
```

This is the normal administrative path in enterprise environments.

If OAuth is broken, this path can also be unavailable.

## system:admin kubeconfig

```text
system:admin kubeconfig
      ↓
client certificate authentication
      ↓
kube-apiserver
```

This does not depend on OAuth.

If the organization securely retains an approved administrative kubeconfig, it can be used to troubleshoot OAuth as long as the API is healthy.

Important distinction:

```text
kubeadmin
≠
system:admin
```

## RHCOS core SSH key

```text
SSH private key
      ↓
core@master / core@worker
      ↓
Linux/RHCOS node access
```

SSH is independent of OAuth.

However:

> SSH access to a node does NOT automatically provide Kubernetes cluster-admin access.

---

# 3. Preferred Troubleshooting Flow

If OAuth login fails:

```text
OAuth login failure
      ↓
Check whether API is reachable
      ↓
Use approved admin kubeconfig if available
      ↓
Check Authentication ClusterOperator
      ↓
Check Authentication Operator
      ↓
Check OAuth pods
      ↓
Check Service / Endpoints
      ↓
Check OAuth Route
      ↓
Check Ingress / DNS / Load Balancer
      ↓
Check OAuth CR
      ↓
Check Secrets / Certificates
      ↓
Check external IdP
      ↓
Fix root cause
      ↓
Validate fresh login
```

---

# 4. Check Authentication ClusterOperator

Start with:

```bash
oc get co authentication
```

or:

```bash
oc get co
```

Then:

```bash
oc describe co authentication
```

Look for:

```text
AVAILABLE
PROGRESSING
DEGRADED
Reason
Message
Conditions
```

Common failure directions can include:

```text
OAuth server deployment issue
OAuth route inaccessible
Identity provider configuration failure
Certificate / TLS error
```

---

# 5. Check the Authentication Operator

Namespace:

```text
openshift-authentication-operator
```

Commands:

```bash
oc get pods -n openshift-authentication-operator
```

```bash
oc logs deployment/authentication-operator \
  -n openshift-authentication-operator
```

The Authentication Operator reconciles the cluster OAuth configuration.

If it is unhealthy, OAuth configuration may not reconcile correctly.

---

# 6. Check the OAuth Server Pods

Namespace:

```text
openshift-authentication
```

Commands:

```bash
oc get pods -n openshift-authentication
```

Then:

```bash
oc describe pod <oauth-pod> \
  -n openshift-authentication
```

```bash
oc logs <oauth-pod> \
  -n openshift-authentication
```

Look for:

```text
CrashLoopBackOff
ImagePullBackOff
invalid OAuth config
certificate errors
missing secret
DNS failure
LDAP/OIDC connectivity failure
permission/RBAC errors
```

---

# 7. Check OAuth Service and Endpoints

```bash
oc get svc -n openshift-authentication
```

Look for:

```text
oauth-openshift
```

Then:

```bash
oc get endpoints oauth-openshift \
  -n openshift-authentication
```

Interpretation:

```text
Service exists + endpoints exist
→ backend pods are likely available

Service exists + no endpoints
→ OAuth pods/readiness/selector issue
```

---

# 8. Check OAuth Route

```bash
oc get route -n openshift-authentication
```

Then:

```bash
oc describe route oauth-openshift \
  -n openshift-authentication
```

External request path:

```text
Client
 ↓
DNS
 ↓
Load Balancer
 ↓
Ingress Router
 ↓
OAuth Route
 ↓
oauth-openshift Service
 ↓
OAuth Pod
```

Useful isolation logic:

```text
External OAuth URL fails
but internal oauth-openshift service works
→ investigate DNS / LB / Ingress / Route / certificate

External OAuth URL fails
and internal service also fails
→ investigate OAuth pods / service / endpoints / operator
```

---

# 9. Check OAuth Cluster Configuration

```bash
oc get oauth cluster -o yaml
```

Inspect:

```text
identityProviders
referenced Secrets
ConfigMaps
CA bundles
client secrets
LDAP bind credentials
OIDC configuration
```

Typical identity providers include:

```text
LDAP
Active Directory-backed LDAP
OIDC
HTPasswd
GitHub
etc.
```

---

# 10. LDAP / AD / OIDC Failure Scenario

OAuth pods can be perfectly healthy while the external IdP is unavailable.

Example:

```text
OpenShift OAuth
      ↓
Corporate LDAP/AD
      ❌
```

Check:

```text
DNS resolution
network connectivity
firewall
LDAP/OIDC endpoint
bind account
password expiration
TLS certificate
CA trust
search base/filter
client ID / client secret
redirect URI
```

Do not assume:

> OAuth login failure = OAuth pod failure.

---

# 11. Certificate Failure Scenario

Common symptoms:

```text
x509 certificate expired
certificate signed by unknown authority
TLS handshake failure
```

Check certificate paths such as:

```text
OAuth route certificate
Ingress wildcard certificate
LDAP/OIDC CA
OAuth-serving certificates
referenced TLS secrets
```

External inspection example:

```bash
openssl s_client \
  -connect oauth-openshift.apps.<cluster-domain>:443 \
  -servername oauth-openshift.apps.<cluster-domain>
```

---

# 12. Events

```bash
oc get events \
  -n openshift-authentication \
  --sort-by=.lastTimestamp
```

```bash
oc get events \
  -n openshift-authentication-operator \
  --sort-by=.lastTimestamp
```

Events may reveal:

```text
FailedMount
missing secret
certificate error
failed scheduling
configuration error
image pull issue
```

---

# 13. If kubeadmin Is Removed

This is normal in a properly configured enterprise cluster.

A strong interview answer is:

> I would not depend on kubeadmin for recovery. I would use an approved administrative kubeconfig from the secure credential store if available. If OAuth is unavailable but the API is healthy, certificate-based admin access can still work. If API credentials are not available, I would move to node-level troubleshooting through approved SSH access and follow the formal break-glass recovery procedure.

---

# 14. SSH to Control-Plane Nodes

During installation, an SSH public key is normally provided in `install-config.yaml`.

The matching private key allows node access:

```bash
ssh -i <private-key> core@<master-hostname-or-ip>
```

Then:

```bash
sudo -i
```

Use this mainly for debugging/disaster recovery.

Node-level checks:

```bash
sudo systemctl status kubelet
sudo journalctl -b -u kubelet.service

sudo crictl ps
sudo crictl ps | grep kube-apiserver
sudo crictl ps | grep etcd
sudo crictl logs <container-id>
```

This helps answer:

```text
Is kubelet healthy?
Is kube-apiserver running?
Is etcd running?
Is this really an OAuth-only failure?
```

---

# 15. Important Limitation of SSH

Do NOT say:

> I will SSH to the master, become root, and then I automatically have cluster-admin.

That is incorrect.

Keep these separate:

```text
SSH authentication
private key
   ↓
core/root on RHCOS
```

vs:

```text
Kubernetes/OpenShift authentication
certificate or token
   ↓
kube-apiserver
   ↓
RBAC
```

Node-local component credentials are not a supported substitute for a cluster-admin credential.

---

# 16. If No kubeadmin, No Admin Kubeconfig, but SSH Exists

Treat this as a real break-glass scenario.

Flow:

```text
SSH to control-plane node
       ↓
Check kubelet
       ↓
Check static/control-plane containers
       ↓
Check kube-apiserver
       ↓
Check etcd
       ↓
Determine API vs OAuth scope
       ↓
If API healthy, focus on authentication stack
       ↓
Use documented enterprise/Red Hat recovery process
```

Do not manufacture credentials from control-plane files.

---

# 17. If SSH Private Key Is Also Missing

RHCOS normally has no default password for the `core` user.

Therefore:

```text
No OAuth admin access
+
No approved admin kubeconfig
+
No SSH private key
=
formal disaster-recovery / break-glass incident
```

The correct enterprise response is to recover approved credentials from the organization's secure vault/backup/recovery process.

---

# 18. Interview Answer to Memorize

> First I separate OAuth failure from API failure. If the API is healthy, I use an approved administrative kubeconfig from the secure credential store rather than depending on kubeadmin. I check the Authentication ClusterOperator, authentication-operator, OAuth pods, service/endpoints, route, ingress path, OAuth CR, certificates and external IdP. If normal API credentials are unavailable, I use the installation-time SSH key to access a control-plane node as `core` and inspect kubelet, kube-apiserver and etcd with systemd, journalctl and crictl. SSH root access does not equal Kubernetes cluster-admin, so if all administrative API credentials are unavailable I treat it as a formal break-glass recovery rather than fabricating credentials.

---

# 19. Final Memory Map

```text
Normal login
Enterprise IdP
   ↓
OAuth
   ↓
API


OAuth down?
   ↓
Is API healthy?
   ↓
Yes
   ↓
approved admin kubeconfig
   ↓
Authentication CO
   ↓
Authentication Operator
   ↓
OAuth Pods
   ↓
Service / Endpoints
   ↓
Route
   ↓
Ingress / DNS / LB
   ↓
OAuth CR
   ↓
Secrets / Certificates
   ↓
LDAP / AD / OIDC


No admin API credential?
   ↓
SSH core@master
   ↓
kubelet / kube-apiserver / etcd diagnostics
   ↓
formal break-glass recovery if needed
```
