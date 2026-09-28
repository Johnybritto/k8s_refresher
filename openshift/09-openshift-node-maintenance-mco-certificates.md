# OpenShift Node Maintenance, MachineConfig/MCO & Certificate Management

This guide is focused on production operations and interview preparation for OpenShift Lead / Architect roles.

---

# 1. Node Maintenance and Replacement

## Machine vs Node

```text
Machine
= infrastructure object
= VMware VM / AWS EC2 / Azure VM etc.

        ↓ boots + kubelet joins

Node
= Kubernetes representation of that machine
```

On vSphere:

```text
vSphere VM
     ↓
Machine CR (if Machine API manages it)
     ↓
RHCOS boots
     ↓
kubelet joins
     ↓
OpenShift Node
```

## Planned worker-node maintenance

Safe sequence:

```text
Check capacity
    ↓
Cordon
    ↓
Drain
    ↓
Perform maintenance
    ↓
Bring node back
    ↓
Verify Ready
    ↓
Uncordon
```

Commands:

```bash
oc get nodes
oc adm top nodes
oc get pods -A -o wide
oc get pdb -A

oc adm cordon worker01

oc adm drain worker01 \
  --ignore-daemonsets \
  --delete-emptydir-data

oc get node worker01

oc adm uncordon worker01
```

### Cordon vs drain

```text
Cordon
= stop NEW workloads from being scheduled

Drain
= evict movable existing workloads from the node

Uncordon
= allow scheduling again
```

Before draining, verify the remaining cluster has enough CPU/memory capacity and that PodDisruptionBudgets will not make application availability unsafe.

## Worker replacement with MachineSet

If the worker belongs to a MachineSet:

```bash
oc get machine -n openshift-machine-api
oc get machineset -n openshift-machine-api
```

Conceptually:

```text
MachineSet desires 3 workers

worker-1
worker-2  ← delete failed machine
worker-3
      ↓
MachineSet sees only 2
      ↓
creates replacement Machine
      ↓
new VM is provisioned
      ↓
RHCOS boots
      ↓
new node joins
```

Example:

```bash
oc delete machine <machine-name> -n openshift-machine-api
```

Important distinction:

```text
Delete Node
≠ necessarily replace the underlying infrastructure

Delete Machine managed by MachineSet
→ MachineSet can create replacement infrastructure
```

## UPI / manually provisioned vSphere

If the VMware worker is not managed by Machine API:

```text
cordon/drain old worker
        ↓
decommission old VM
        ↓
create new RHCOS VM in vSphere
        ↓
apply the supported provisioning/Ignition process
        ↓
kubelet submits certificates
        ↓
approve valid CSRs if required
        ↓
node joins
        ↓
MCO brings it to desired configuration
```

## Control-plane maintenance / replacement

Treat control-plane nodes differently because of etcd quorum.

```text
Take etcd backup
     ↓
Verify etcd health
     ↓
Replace ONE control-plane node
     ↓
Wait for recovery
     ↓
Verify etcd member health
     ↓
Verify ClusterOperators
     ↓
Only then consider another node
```

Never replace all masters together.

---

# 2. MachineConfig and Machine Config Operator (MCO)

## What problem does MCO solve?

RHCOS nodes are intended to be managed declaratively.

Instead of manually SSHing to every node and changing files/services, OpenShift declares the desired OS configuration and MCO reconciles nodes to that desired state.

Think:

```text
Deployment
→ desired state for application workloads

MachineConfig
→ desired state for node operating system configuration
```

MachineConfig/MCO can manage host-level configuration such as:

- files
- systemd units
- kubelet configuration
- CRI-O configuration
- kernel arguments
- registries
- NetworkManager-related configuration
- SSH keys and other supported RHCOS settings

## MachineConfig

Example skeleton:

```yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  name: 99-worker-custom
  labels:
    machineconfiguration.openshift.io/role: worker
spec:
  config:
    ignition:
      version: 3.x.x
```

The role label determines which MachineConfigPool includes the configuration.

## Worker vs master MachineConfig

There is NOT one single MachineConfig for the whole cluster.

For workers:

```yaml
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
```

For control-plane nodes:

```yaml
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
```

Apply:

```bash
oc apply -f 99-worker-custom.yaml
```

Conceptually:

```text
MachineConfig A
role: worker
     ↓
worker MCP
     ↓
worker nodes

MachineConfig B
role: master
     ↓
master MCP
     ↓
control-plane nodes
```

## MachineConfigPool (MCP)

List pools:

```bash
oc get mcp
```

Typical pools:

```text
master
worker
```

Custom pools can also exist, for example:

```text
infra
gpu
storage
```

A custom MCP is useful when a subset of workers needs different host-level configuration.

Mental model:

```text
MachineConfig
     ↓ selected by role
MachineConfigPool
     ↓ selects nodes
Nodes
```

## Rendered MachineConfig

Multiple MachineConfigs for a pool are merged into a rendered desired configuration.

```text
00-worker
01-worker-container-runtime
50-worker-chrony
99-worker-custom
       ↓
rendered-worker-abc123
       ↓
desired configuration for worker MCP
```

Check:

```bash
oc get mc
```

## How MCO rolls out changes

Typical flow:

```text
Create/change MachineConfig
        ↓
MCO detects change
        ↓
new rendered MachineConfig
        ↓
MCP becomes Updating
        ↓
select node
        ↓
cordon
        ↓
drain
        ↓
apply configuration
        ↓
reboot if required
        ↓
node returns Ready
        ↓
uncordon
        ↓
move to next node
```

Watch rollout:

```bash
oc get mcp
oc describe mcp worker
```

Typical status:

```text
NAME     UPDATED   UPDATING   DEGRADED
master   True      False      False
worker   False     True       False
```

## MCO components

Remember these three:

```text
machine-config-controller
→ coordinates desired configuration

machine-config-daemon
→ runs on nodes and applies host changes

machine-config-server
→ provides Ignition/configuration to joining machines
```

## maxUnavailable

Controls how many nodes in an MCP can be unavailable during rollout.

Typical/default behavior is conservative, usually one node at a time.

```text
worker1 update
   ↓ healthy
worker2 update
   ↓ healthy
worker3 update
```

For control-plane nodes, keep disruption especially conservative.

## Does every MachineConfig require reboot?

No.

Many host-level changes may trigger drain + reboot, but modern OpenShift also supports node-disruption policies that can allow actions such as:

```text
None
Drain
service Reload
service Restart
DaemonReload
Reboot
```

So the correct interview answer is:

> Many MachineConfig changes can trigger drain and reboot, but not every change necessarily requires a reboot.

## Configuration drift

If someone manually edits an MCO-managed file on a node:

```text
Desired config
≠
Actual config
```

The Machine Config Daemon can detect drift and mark the node/pool degraded.

Therefore:

> Do not make persistent node configuration changes manually over SSH. Use MachineConfig or the appropriate supported OpenShift API.

## MCO troubleshooting

Start here:

```bash
oc get co machine-config
oc get mcp
oc describe mcp worker
oc get mc
oc get nodes
oc get pods -n openshift-machine-config-operator -o wide
```

Common causes of a degraded MCP:

```text
Drain blocked by PDB
Configuration drift
Bad/invalid MachineConfig
Node failed to reboot
Node did not return Ready
Filesystem/disk issue
Invalid file/service configuration
```

---

# 3. Certificate Management

Do not memorize every OpenShift certificate. Group them into four areas:

| Certificate type | Example | Typical management |
|---|---|---|
| Internal platform | etcd/API/internal components | OpenShift Operators |
| Node/kubelet | node ↔ API | OpenShift / CSR workflow |
| Service certificates | service ↔ service TLS | service-ca |
| External/user-facing | API and *.apps wildcard | Enterprise/public PKI commonly used |

## A. Node / kubelet certificates

When a new node joins:

```text
RHCOS starts
    ↓
kubelet starts
    ↓
CSR submitted
    ↓
certificate approved/issued
    ↓
kubelet authenticates to API
    ↓
Node becomes usable
```

Commands:

```bash
oc get csr
```

If a CSR is valid and expected:

```bash
oc adm certificate approve <csr-name>
```

Do not blindly approve CSRs. Validate the node identity/request first.

In UPI environments, some serving CSRs may require approval depending on the setup.

## B. API server certificate

Users/clients access:

```text
https://api.cluster.example.com:6443
```

Enterprises commonly replace the default external API certificate with one signed by an enterprise/public CA.

Inspect:

```bash
oc get apiserver cluster -o yaml
```

Important:

> Do not treat the internal API endpoint (`api-int`) like the external API certificate endpoint.

## C. Ingress / wildcard certificate

Application routes commonly use:

```text
app1.apps.cluster.example.com
app2.apps.cluster.example.com
console-openshift-console.apps.cluster.example.com
```

So the default ingress certificate generally covers:

```text
*.apps.cluster.example.com
```

The custom TLS secret is commonly stored in:

```text
openshift-ingress
```

and referenced by the default IngressController in:

```text
openshift-ingress-operator
```

Inspect:

```bash
oc get ingresscontroller default -n openshift-ingress-operator -o yaml
oc get secret -n openshift-ingress
```

## D. Service serving certificates

OpenShift can issue internal service certificates.

Example:

```bash
oc annotate service myservice \
  service.beta.openshift.io/serving-cert-secret-name=myservice-tls
```

OpenShift creates a secret containing:

```text
tls.crt
tls.key
```

for internal service DNS such as:

```text
myservice.mynamespace.svc
```

## Certificate troubleshooting

First identify WHICH certificate path is broken.

```text
API certificate issue
→ oc/API client connectivity affected

Ingress certificate issue
→ console/routes/apps affected

Kubelet certificate issue
→ node/API communication affected

Service certificate issue
→ internal service-to-service TLS affected
```

Start with:

```bash
oc get co
```

Check endpoint certificate:

```bash
openssl s_client \
  -connect api.cluster.example.com:6443 \
  -servername api.cluster.example.com
```

Inspect validity:

```bash
openssl s_client \
  -connect api.cluster.example.com:6443 \
  -servername api.cluster.example.com 2>/dev/null \
| openssl x509 -noout -subject -issuer -dates
```

For routes, use port 443.

---

# 4. How These Three Topics Connect

New worker lifecycle:

```text
New worker created
      ↓
Machine Config Server provides bootstrap/Ignition config
      ↓
RHCOS starts
      ↓
kubelet submits CSR
      ↓
certificate established
      ↓
Node joins cluster
      ↓
MCO sees worker pool membership
      ↓
MCO applies/validates rendered configuration
      ↓
Node Ready
      ↓
scheduler places workloads
```

Maintenance lifecycle:

```text
Cordon
  ↓
Drain
  ↓
repair / replace
  ↓
new or repaired node joins
  ↓
certificates established
  ↓
MCO validates/applies desired config
  ↓
Ready
  ↓
Uncordon
```

---

# 5. Interview Memory Points

1. Cordon stops new scheduling; drain evicts movable existing workloads; uncordon restores scheduling.
2. Worker replacement through a MachineSet can be automated; UPI replacement is more manual.
3. Control-plane replacement must protect etcd quorum and should be done one node at a time.
4. MachineConfig defines node OS desired state; MachineConfigPool groups target nodes; MCO rolls out rendered configuration.
5. Worker and master MachineConfigs are targeted using `machineconfiguration.openshift.io/role`.
6. Custom MCPs such as infra/gpu can isolate special node configuration.
7. MCO commonly performs cordon → drain → apply → reboot if needed → uncordon.
8. Persistent manual OS changes can create configuration drift and degrade the pool.
9. Most internal certificates are Operator-managed; administrators commonly deal directly with API/Ingress certificates and CSR troubleshooting.
10. Node joining, MCO reconciliation and certificate issuance are part of the same node lifecycle story.
