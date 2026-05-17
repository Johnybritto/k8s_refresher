# Day 6 – Security, RBAC, Certificates & Compliance

Status: Covered in chat. These notes capture Day 6 learning and the RBAC troubleshooting exercise review.

---

## 1. Kubernetes Security Model

Kubernetes security is layered. It is not solved by RBAC alone.

Main layers:

```text
Authentication -> Authorization -> Admission -> Pod Security -> Runtime Security -> Network Security -> Secrets -> Certificates -> Audit
```

Meaning:

```text
Authentication = Who are you?
Authorization = What are you allowed to do?
Admission = Should this object be accepted before storing?
Runtime security = Can this container run safely?
Network security = Who can talk to whom?
Audit = Who did what?
```

Senior interview line:

```text
Kubernetes security is layered across identity, RBAC, admission policy, runtime restrictions, network controls, secret protection, certificate management, scanning, and audit.
```

---

## 2. Authentication

Authentication answers:

```text
Who is making this request?
```

Common identities:

- Human users
- ServiceAccounts
- CI/CD systems
- GitOps controllers
- Cloud IAM identities
- OIDC users

Important:

```text
Kubernetes does not usually store human users as Kubernetes API objects. ServiceAccounts are Kubernetes API objects.
```

Examples:

- EKS may integrate AWS IAM.
- AKS may integrate Entra ID / Azure AD.
- OpenShift commonly integrates OAuth, LDAP, or identity providers.

---

## 3. Authorization

Authorization answers:

```text
Are you allowed to perform this action?
```

Examples:

```text
Can this user create deployments in prod?
Can Jenkins update services in dev?
Can ArgoCD sync applications in this namespace?
Can this ServiceAccount list secrets?
```

The most common Kubernetes authorization model is RBAC.

---

## 4. RBAC Deep Dive

RBAC means Role-Based Access Control.

Main objects:

```text
Role
ClusterRole
RoleBinding
ClusterRoleBinding
```

---

## 5. Role

A Role grants permissions inside one namespace.

Example permission:

```text
Allow get/list/watch Pods in prod namespace.
```

A Role alone does nothing until it is bound to a subject.

---

## 6. RoleBinding

A RoleBinding attaches a Role or ClusterRole to a subject inside a namespace.

Subjects can be:

- User
- Group
- ServiceAccount

Example:

```text
User johny gets pod-reader Role in prod namespace.
```

---

## 7. ClusterRole

A ClusterRole can define permissions for cluster-scoped resources or reusable permissions.

Cluster-scoped resources include:

- Nodes
- Namespaces
- PersistentVolumes
- ClusterRoles
- ClusterRoleBindings
- CRDs

Important nuance:

```text
A ClusterRole does not automatically mean cluster-wide access. If it is bound using a RoleBinding, access is limited to that namespace. If it is bound using a ClusterRoleBinding, access is cluster-wide.
```

---

## 8. ClusterRoleBinding

ClusterRoleBinding grants ClusterRole permissions across the cluster.

This is powerful and should be used carefully.

Risky example:

```text
Binding a developer or application ServiceAccount to cluster-admin.
```

Senior point:

```text
Prefer namespace RoleBinding where possible. Use ClusterRoleBinding only when cluster-wide access is truly required.
```

---

## 9. ServiceAccount

A ServiceAccount is an identity for Pods and controllers.

Flow:

```text
Pod uses ServiceAccount -> token is mounted/projected -> Pod calls API Server -> RBAC authorizes or denies
```

Important:

```text
A ServiceAccount does not automatically get useful permissions. It needs RoleBinding or ClusterRoleBinding.
```

Strong answer:

```text
ServiceAccounts provide identities for workloads running inside Kubernetes. RBAC permissions are assigned to ServiceAccounts using RoleBindings or ClusterRoleBindings. This is how Pods, controllers, CI/CD agents, and GitOps tools safely access the API Server.
```

---

## 10. Least Privilege

Least privilege means granting only the exact permissions needed.

Avoid:

```text
verbs: ["*"]
resources: ["*"]
cluster-admin for application ServiceAccounts
broad ClusterRoleBindings
get/list secrets unless truly needed
```

Prefer:

```text
Specific verbs
Specific resources
Namespace-scoped RoleBindings
Separate identities for apps, CI/CD, GitOps, and humans
Periodic RBAC audits
```

Interview line:

```text
Most RBAC incidents happen because users or ServiceAccounts are granted broader permissions than required.
```

---

## 11. Checking RBAC Permissions

Useful commands:

```bash
kubectl auth can-i create deployments -n prod
kubectl auth can-i create deployments -n prod --as=<user>
kubectl auth can-i list secrets -n prod --as=system:serviceaccount:prod:<serviceaccount>
```

Trace bindings:

```bash
kubectl get rolebinding -n prod
kubectl get clusterrolebinding
kubectl describe rolebinding <name> -n prod
kubectl describe clusterrolebinding <name>
kubectl describe role <role> -n prod
kubectl describe clusterrole <clusterrole>
```

---

## 12. Pod Security

Pod security controls what containers are allowed to do at runtime.

Risky settings:

- privileged: true
- hostNetwork: true
- hostPID: true
- hostPath volumes
- running as root
- allowPrivilegeEscalation: true
- extra Linux capabilities
- writable root filesystem

Safer controls:

```text
runAsNonRoot: true
allowPrivilegeEscalation: false
readOnlyRootFilesystem: true
drop Linux capabilities
seccomp profile
```

Strong answer:

```text
For Pod security, I would restrict privileged containers, hostPath, hostNetwork, root user, privilege escalation, and unnecessary Linux capabilities. I would enforce these using Pod Security Admission, Kyverno, or OPA Gatekeeper.
```

---

## 13. Pod Security Admission

PodSecurityPolicy is removed in modern Kubernetes. Current Kubernetes uses Pod Security Admission.

Pod Security levels:

```text
Privileged
Baseline
Restricted
```

- Privileged: allows almost everything; only for trusted system workloads.
- Baseline: prevents known privilege escalations while allowing common workloads.
- Restricted: strict profile for stronger security.

Namespace label example:

```bash
kubectl label namespace prod pod-security.kubernetes.io/enforce=restricted
```

Interview line:

```text
Pod Security Admission is namespace-label based and enforces built-in Pod security standards such as baseline and restricted.
```

---

## 14. Secrets

Kubernetes Secrets store sensitive data such as:

- Passwords
- Tokens
- Certificates
- API keys
- Registry credentials

Important:

```text
Kubernetes Secrets are base64-encoded, not automatically encrypted unless encryption at rest is enabled.
```

Production best practices:

- Enable encryption at rest.
- Restrict RBAC access to secrets.
- Avoid broad get/list secrets permissions.
- Use external secret managers where possible.
- Rotate secrets.
- Avoid printing secrets in logs/env dumps.

Strong answer:

```text
Kubernetes Secrets are used to store sensitive data, but base64 encoding is not encryption. For production, I would enable encryption at rest, restrict RBAC access, rotate secrets, and integrate with external secret managers such as AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, or External Secrets Operator.
```

---

## 15. Certificates in Kubernetes

Kubernetes uses certificates for trust between components:

- kubectl and API Server
- API Server and kubelet
- API Server and etcd
- API Server and webhooks
- kubelet client/server certs
- front-proxy components
- ingress TLS
- service mesh mTLS

Expired certificate symptoms:

- x509 errors
- API Server cannot connect to etcd
- kubelet authentication issues
- webhooks fail
- nodes NotReady
- ingress TLS errors

---

## 16. Certificate Rotation

For kubeadm clusters:

```bash
kubeadm certs check-expiration
kubeadm certs renew all
```

For managed Kubernetes:

```text
Control plane certs are often handled by the provider, but ingress certs, webhook certs, service mesh certs, and application certs still need monitoring.
```

Strong answer:

```text
Certificate rotation depends on the platform. In kubeadm clusters, I would use kubeadm certs check-expiration and follow the renewal procedure. In managed Kubernetes, the provider usually handles control-plane certificates, but I still monitor ingress TLS, webhook certs, service mesh certs, and application certificates.
```

---

## 17. cert-manager

cert-manager automates certificate issuance and renewal in Kubernetes.

It can use:

- Let's Encrypt
- HashiCorp Vault
- Venafi
- Private CA
- Self-signed issuer
- ACME providers

Main resources:

- Issuer
- ClusterIssuer
- Certificate
- CertificateRequest

Interview line:

```text
cert-manager automates certificate lifecycle: issuing, renewing, and storing TLS certificates as Kubernetes Secrets.
```

---

## 18. OPA Gatekeeper and Kyverno

Both enforce policy-as-code at admission time.

Examples:

- Block privileged containers.
- Require resource requests/limits.
- Require labels.
- Allow only approved registries.
- Block hostPath.
- Require runAsNonRoot.
- Require NetworkPolicy for production namespaces.

Difference:

```text
OPA Gatekeeper uses Rego.
Kyverno uses Kubernetes-native YAML policies.
```

Strong answer:

```text
OPA Gatekeeper and Kyverno enforce policy-as-code through admission control. They help standardize security and compliance by rejecting or mutating resources that violate platform rules. Gatekeeper uses Rego; Kyverno uses Kubernetes-native YAML policies.
```

---

## 19. Vulnerability Scanning

Scan areas:

- Container images
- Base images
- OS packages
- Language dependencies
- Kubernetes manifests
- Helm charts
- Running workloads
- Cluster misconfigurations

Common tools:

- Trivy
- Grype
- Clair
- Anchore
- Snyk
- Prisma
- Aqua

Best practice:

```text
Scan in CI/CD, scan registry, enforce admission policy, and continuously scan runtime workloads.
```

Strong answer:

```text
I would integrate vulnerability scanning at CI/CD, registry, admission, and runtime levels. Images with critical vulnerabilities should be blocked or require approval based on severity, exploitability, and business context.
```

---

## 20. Scenario: Expired Certificate Outage

Symptoms:

- kubectl fails with x509 errors
- API Server cannot talk to etcd
- kubelet cannot authenticate
- webhooks fail
- ingress TLS fails
- metrics unavailable

Troubleshooting:

```bash
kubectl get nodes
kubectl get pods -A
journalctl -u kubelet
kubeadm certs check-expiration
```

Check:

- API Server cert
- etcd cert
- kubelet cert
- front-proxy cert
- webhook certs
- ingress certs
- service mesh certs

Strong answer:

```text
For a certificate outage, I identify which certificate path is failing based on x509 errors. If the API Server is unavailable, I check control-plane logs and certificate expiry. In kubeadm clusters, I use kubeadm certs check-expiration and follow the renewal procedure. I also check etcd, kubelet, webhook, ingress, and service mesh certificates because not all certificate failures are control-plane certificate failures.
```

---

## 21. Scenario: RBAC Misconfiguration Breach

Example:

```text
A developer or application ServiceAccount can list Secrets in prod.
```

Impact:

- Database passwords exposed
- API keys exposed
- Cloud credentials exposed
- Lateral movement possible
- Compliance breach

Troubleshooting:

```bash
kubectl auth can-i list secrets -n prod --as=<user>
kubectl get rolebinding,clusterrolebinding -A | grep <user-or-serviceaccount>
kubectl describe rolebinding <name> -n prod
kubectl describe clusterrolebinding <name>
```

Remediation:

- Remove broad bindings.
- Replace ClusterRoleBinding with namespace RoleBinding where possible.
- Remove wildcard verbs/resources.
- Rotate exposed secrets.
- Review audit logs.
- Add preventive policy.

Strong answer:

```text
For RBAC misconfiguration, I first confirm the effective permission using kubectl auth can-i. Then I trace the RoleBinding or ClusterRoleBinding that grants the permission. I remove broad access, replace cluster-wide bindings with namespace-scoped bindings where possible, rotate exposed secrets, review audit logs, and add preventive policy to block overly broad permissions.
```

---

## 22. Day 6 Exercise Review

### Exercise

A developer reports they cannot deploy an application in the prod namespace. The error says: `forbidden: User cannot create deployments.apps in namespace prod`. How will you troubleshoot and fix this?

### User Answer

```text
the error itself says that the user does not have enough permission to create deployment > will check which rolebinding he belongs to and then from there which role is associated and will check the permission of the roles
```

### Review

This is correct. The error clearly points to an RBAC authorization failure. You correctly identified that we need to trace RoleBinding/Role permissions.

To make it senior-level, add:

- Confirm the exact user identity.
- Use `kubectl auth can-i`.
- Check RoleBinding and ClusterRoleBinding.
- Check Role or ClusterRole rules for `deployments.apps` and verb `create`.
- Confirm namespace is `prod`.
- Follow least privilege before granting access.
- Avoid giving cluster-admin.
- If access is approved, create/update a namespace-scoped RoleBinding.

### Strong Interview Version

```text
The error indicates an RBAC authorization issue. I would first confirm the identity of the user and the namespace. Then I would run kubectl auth can-i create deployments.apps -n prod --as=<user> to confirm the permission failure.

Next, I would check RoleBindings in the prod namespace and ClusterRoleBindings that may apply to the user or their group. From the binding, I would inspect the referenced Role or ClusterRole and verify whether it includes apiGroup apps, resource deployments, and verb create.

If the user is supposed to deploy in prod, I would grant least-privilege access using a Role or ClusterRole with only the required verbs and bind it with a RoleBinding in prod. I would avoid cluster-admin or broad wildcard permissions. If the user should not deploy directly to prod, I would route deployment through the approved CI/CD or GitOps ServiceAccount instead.
```

### Useful Commands

```bash
kubectl auth can-i create deployments.apps -n prod --as=<user>
kubectl get rolebinding -n prod
kubectl get clusterrolebinding
kubectl describe rolebinding <binding-name> -n prod
kubectl describe clusterrolebinding <binding-name>
kubectl describe role <role-name> -n prod
kubectl describe clusterrole <clusterrole-name>
```

### Example Least-Privilege Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: prod
  name: deployment-manager
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
```

Bind it to a user/group/ServiceAccount using a RoleBinding in the prod namespace.

---

## 23. Day 6 Final Summary

```text
Authentication = who are you?
Authorization = what can you do?
Admission = should this object be allowed?
Pod Security = how safely can the container run?
Secrets = how sensitive data is stored and accessed?
Certificates = how components trust each other?
Policy-as-code = how rules are enforced automatically?
Audit = how actions are traced?
```

Most important senior-level line:

```text
Kubernetes security is layered. RBAC alone is not enough; you need identity, least privilege, admission policy, runtime restrictions, secret protection, certificate management, network controls, scanning, and audit.
```
