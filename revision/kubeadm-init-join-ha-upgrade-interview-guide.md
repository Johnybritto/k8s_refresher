# kubeadm: init, join, HA API Access and Upgrade — Interview Guide

> **Purpose:** Interview-ready understanding of how an upstream/self-managed Kubernetes cluster is bootstrapped and upgraded with kubeadm.
>
> **Scope:** `kubeadm init`, `kubeadm join`, internal phases, HA API access, stacked etcd, additional control-plane nodes, worker joins, certificates/tokens, and rolling upgrades.
>
> **Important:** kubeadm bootstraps Kubernetes. It does **not** provision VMs/bare-metal servers, load balancers, storage, monitoring, or application ingress for you.

---

# 1. Reference HA Architecture

For a production-style kubeadm cluster:

```text
                   kubectl / kubelets
                          |
                          v
                  DNS: k8s-api.example.com
                          |
                          v
                    Load Balancer / VIP
                         :6443
                          |
             +------------+------------+
             |            |            |
             v            v            v
           CP-1          CP-2          CP-3
        kube-apiserver kube-apiserver kube-apiserver
        scheduler      scheduler      scheduler
        controller     controller     controller
        etcd-1         etcd-2         etcd-3
             \            |            /
              +-----------+-----------+
                          |
                     etcd quorum
                          |
             +------------+------------+
             |                         |
             v                         v
          Worker-1                  Worker-N
          kubelet                   kubelet
          containerd/CRI-O          containerd/CRI-O
          CNI                       CNI
          workloads                 workloads
```

The common kubeadm HA design is:

- 3 control-plane nodes
- stacked etcd on those control-plane nodes
- a load balancer/VIP in front of the API servers
- multiple workers
- CNI installed after first control plane is initialized

A stacked topology is the kubeadm default. Each control-plane node runs a local etcd member.

---

# 2. What kubeadm Actually Does

Think of kubeadm as the **cluster bootstrap/orchestration tool**.

It handles:

- Kubernetes PKI/certificates
- kubeconfig files
- static Pod manifests for control-plane components
- bootstrap tokens
- kubelet bootstrap
- cluster configuration
- etcd bootstrap for stacked topology
- joining new control-plane nodes
- joining worker nodes
- Kubernetes version upgrades

It does **not** normally create:

- Linux VMs
- bare-metal hosts
- DNS records
- the HA load balancer
- CNI plugin
- CSI/storage system
- ingress controller
- observability stack

---

# 3. Prerequisites Before kubeadm init

On all Kubernetes nodes, prepare:

```text
Linux OS
  |
container runtime: containerd or CRI-O
  |
kubeadm
kubelet
kubectl (control-plane/admin hosts)
  |
swap disabled or appropriately configured
  |
kernel modules/sysctl
  |
DNS / hostnames / NTP
  |
firewall / required ports
```

For an HA control plane, provision the load balancer before `kubeadm init`.

Example:

```text
k8s-api.example.com -> 10.10.10.100

10.10.10.100:6443
       |
       +--> cp-1:6443
       +--> cp-2:6443
       +--> cp-3:6443
```

The LB should perform a TCP health check against the API servers.

---

# 4. kubeadm Configuration for HA

A configuration file is cleaner than a very long command.

Example:

```yaml
apiVersion: kubeadm.k8s.io/v1beta4
kind: ClusterConfiguration

kubernetesVersion: <TARGET_VERSION>

controlPlaneEndpoint: "k8s-api.example.com:6443"

networking:
  podSubnet: "10.244.0.0/16"
  serviceSubnet: "10.96.0.0/12"

---
apiVersion: kubeadm.k8s.io/v1beta4
kind: InitConfiguration

localAPIEndpoint:
  advertiseAddress: 10.10.10.11
  bindPort: 6443

nodeRegistration:
  criSocket: unix:///run/containerd/containerd.sock
```

The critical HA field is:

```yaml
controlPlaneEndpoint: "k8s-api.example.com:6443"
```

That endpoint should resolve to the load balancer/VIP, **not to one individual control-plane node**.

---

# 5. First Control Plane — kubeadm init

Typical HA command:

```bash
sudo kubeadm init \
  --config kubeadm-config.yaml \
  --upload-certs
```

A simpler non-HA/lab example can look like:

```bash
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16
```

For an HA cluster, prefer the configuration file with `controlPlaneEndpoint`.

---

# 6. What Happens Internally During kubeadm init

Interview question:

> What happens when you run kubeadm init?

The simplified sequence is:

```text
kubeadm init
    |
    +--> 1. Preflight checks
    |
    +--> 2. Generate certificates
    |
    +--> 3. Generate kubeconfig files
    |
    +--> 4. Generate control-plane static Pod manifests
    |
    +--> 5. Start kubelet / static Pods
    |
    +--> 6. Start local etcd
    |
    +--> 7. Start kube-apiserver
    |
    +--> 8. Start controller-manager
    |
    +--> 9. Start scheduler
    |
    +--> 10. Upload kubeadm configuration
    |
    +--> 11. Configure bootstrap tokens / RBAC
    |
    +--> 12. Deploy CoreDNS and kube-proxy
    |
    +--> 13. Generate join commands
```

Let's break this down.

---

## 6.1 Preflight Checks

kubeadm validates things such as:

- required binaries
- ports are available
- container runtime exists
- kubelet environment
- swap configuration
- host networking
- existing Kubernetes manifests/configuration

If a critical prerequisite fails, `kubeadm init` normally stops before changing the cluster substantially.

---

## 6.2 Certificate Generation

kubeadm builds the Kubernetes PKI under:

```text
/etc/kubernetes/pki/
```

Important certificates include:

```text
ca.crt / ca.key
apiserver.crt / apiserver.key
apiserver-kubelet-client.crt
front-proxy-ca.crt
front-proxy-client.crt
sa.key / sa.pub

etcd/
  ca.crt
  server.crt
  peer.crt
  healthcheck-client.crt
```

These certificates establish trust between:

- kubectl/clients and API server
- API server and kubelets
- API server and etcd
- etcd peers
- aggregated API servers

---

## 6.3 kubeconfig Generation

kubeadm creates kubeconfig files under:

```text
/etc/kubernetes/
```

Common files:

```text
admin.conf
controller-manager.conf
scheduler.conf
kubelet.conf
```

For kubectl administration:

```bash
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 6.4 Static Pod Manifests

kubeadm writes control-plane manifests into:

```text
/etc/kubernetes/manifests/
```

Normally:

```text
kube-apiserver.yaml
kube-controller-manager.yaml
kube-scheduler.yaml
etcd.yaml
```

The **kubelet watches this directory**.

When a manifest appears there:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
                    |
                    v
                  kubelet
                    |
                    v
              container runtime
                    |
                    v
             kube-apiserver Pod
```

That is why control-plane components can start before the Kubernetes API itself is fully available.

---

## 6.5 etcd Starts

In stacked etcd topology:

```text
CP-1
 |
 +-- kube-apiserver
 |
 +-- etcd member
```

The first `kubeadm init` creates the initial etcd member.

When more control-plane nodes join, more etcd members are added.

Typical HA result:

```text
CP-1 -> etcd-1
CP-2 -> etcd-2
CP-3 -> etcd-3
```

etcd uses quorum.

For 3 members:

```text
quorum = 2

1 etcd member can fail
cluster still has quorum
```

If 2 of the 3 etcd members fail, quorum is lost and the cluster cannot safely perform normal writes.

---

## 6.6 API Server Starts

The kube-apiserver connects to etcd and exposes the Kubernetes API on port 6443.

Initially only CP-1 exists:

```text
Load Balancer
     |
     +--> CP-1 API server   READY
     +--> CP-2             not joined yet
     +--> CP-3             not joined yet
```

As CP-2 and CP-3 join, the load balancer begins using them after their health checks succeed.

---

## 6.7 Scheduler and Controller Manager Start

The scheduler and controller manager also run as static Pods.

With multiple control-plane nodes, there are multiple instances, but they use **leader election** for controllers that must have only one active leader.

Simplified:

```text
controller-manager CP-1 -> leader
controller-manager CP-2 -> standby
controller-manager CP-3 -> standby
```

If the leader fails, another instance wins leader election.

The same principle applies to the scheduler.

---

## 6.8 CoreDNS and kube-proxy

kubeadm installs the default kubeadm-managed addons:

- CoreDNS
- kube-proxy

However, Pod networking is not functional until you install a CNI.

After `kubeadm init`, you typically install Cilium, Calico, Flannel, etc.

Example:

```text
kubeadm init
    |
control plane up
    |
CNI not installed yet
    |
nodes may remain NotReady
    |
install CNI
    |
Pod networking becomes functional
```

---

# 7. Output of kubeadm init

At the end, kubeadm prints:

1. worker join command
2. control-plane join command if `--upload-certs` is used
3. certificate key
4. admin kubeconfig instructions

Worker example:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

Additional control-plane example:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

---

# 8. What --upload-certs Does

Command:

```bash
sudo kubeadm init \
  --config kubeadm-config.yaml \
  --upload-certs
```

kubeadm encrypts the shared control-plane certificates and temporarily stores them in the cluster in the `kubeadm-certs` Secret.

The certificate key is required for another control-plane node to decrypt them.

Conceptually:

```text
CP-1 certificates
       |
       | encrypted
       v
kubeadm-certs Secret
       |
       | certificate-key
       v
CP-2 / CP-3 downloads and decrypts certs
```

Treat the certificate key as sensitive.

---

# 9. kubeadm join — Worker Node

Worker join command:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<CA_HASH>
```

---

# 10. What Happens During Worker kubeadm join

Interview explanation:

```text
kubeadm join
    |
    +--> preflight checks
    |
    +--> contact controlPlaneEndpoint
    |
    +--> validate API server CA using CA hash
    |
    +--> authenticate using bootstrap token
    |
    +--> perform TLS bootstrap
    |
    +--> generate kubelet configuration
    |
    +--> start/restart kubelet
    |
    +--> kubelet submits CSR
    |
    +--> CSR approved
    |
    +--> kubelet receives client certificate
    |
    +--> Node object registered
    |
    +--> node becomes Ready after runtime/CNI is healthy
```

---

# 11. Why Both Token and CA Hash Are Used

Example:

```bash
--token abcdef.0123456789abcdef
--discovery-token-ca-cert-hash sha256:<CA_HASH>
```

They solve different problems.

### Bootstrap token

Provides temporary bootstrap authentication.

```text
new node
   |
bootstrap token
   |
   v
API server
```

### CA certificate hash

Allows the joining node to verify that it is talking to the **correct Kubernetes control plane**, helping prevent a man-in-the-middle bootstrap attack.

```text
new node
    |
CA hash verification
    |
    v
trusted API server
```

After TLS bootstrap, the kubelet obtains its own long-term client certificate. It does not continue using the bootstrap token for normal kubelet operation.

---

# 12. TLS Bootstrap During kubeadm join

The trust establishment can be remembered as:

```text
1. Node trusts cluster
      |
   CA hash

2. Cluster temporarily trusts node
      |
 bootstrap token

3. Node creates CSR
      |
      v
 API server

4. CSR approved
      |
      v
 kubelet certificate

5. kubelet uses certificate thereafter
```

---

# 13. Joining Additional Control-Plane Nodes

Command:

```bash
sudo kubeadm join k8s-api.example.com:6443 \
  --token abcdef.0123456789abcdef \
  --discovery-token-ca-cert-hash sha256:<CA_HASH> \
  --control-plane \
  --certificate-key <CERTIFICATE_KEY>
```

The extra flags are the major difference:

```text
--control-plane
     |
     -> create API server
     -> controller-manager
     -> scheduler
     -> etcd member (stacked topology)

--certificate-key
     |
     -> download/decrypt shared control-plane certificates
```

---

# 14. What Happens During Control-Plane join

```text
kubeadm join --control-plane
       |
       +--> bootstrap against existing API endpoint
       |
       +--> download/decrypt shared certs
       |
       +--> create local control-plane kubeconfigs
       |
       +--> add local etcd member
       |
       +--> generate static Pod manifests
       |
       +--> kubelet starts:
              etcd
              kube-apiserver
              scheduler
              controller-manager
       |
       +--> LB health check sees new API server
       |
       +--> node becomes active HA control-plane member
```

After adding CP-2 and CP-3:

```text
                API LB
                  |
       +----------+----------+
       |          |          |
      CP-1       CP-2       CP-3
       |          |          |
     etcd-1     etcd-2     etcd-3
```

---

# 15. If the Join Token Expires

List tokens:

```bash
kubeadm token list
```

Generate a new worker join command:

```bash
kubeadm token create --print-join-command
```

For additional control-plane nodes, you may also need a fresh uploaded certificate key:

```bash
sudo kubeadm init phase upload-certs --upload-certs
```

This prints a new certificate key.

Then combine that with a fresh join command and:

```text
--control-plane --certificate-key <NEW_KEY>
```

---

# 16. How HA API Access Works

This is one of the most important interview topics.

Clients should **not normally use an individual control-plane IP**.

They use:

```text
https://k8s-api.example.com:6443
```

DNS:

```text
k8s-api.example.com
        |
        v
Load Balancer / VIP
10.10.10.100
```

Load balancer backend pool:

```text
10.10.10.11:6443  CP-1
10.10.10.12:6443  CP-2
10.10.10.13:6443  CP-3
```

Traffic flow:

```text
kubectl / kubelet / controller
             |
             v
   k8s-api.example.com:6443
             |
             v
       Load Balancer
             |
    +--------+--------+
    |        |        |
    v        v        v
  CP-1     CP-2     CP-3
  API      API      API
```

---

# 17. What controlPlaneEndpoint Does

In kubeadm configuration:

```yaml
controlPlaneEndpoint: "k8s-api.example.com:6443"
```

This becomes the stable endpoint used by cluster clients and joining nodes.

Without this stable endpoint, you might initially bootstrap against one control-plane node's IP, which creates a poor HA design.

Interview line:

> The API servers are individually stateless front ends to the same Kubernetes state in etcd. A TCP load balancer exposes one stable control-plane endpoint, and failed API server nodes are removed from traffic through health checks.

---

# 18. What Happens if One Control-Plane Node Fails

Example:

```text
Before:

LB
|
+--> CP-1 healthy
+--> CP-2 healthy
+--> CP-3 healthy

CP-1 fails

LB
|
+--> CP-1 unhealthy   X
+--> CP-2 healthy
+--> CP-3 healthy
```

Effects:

- API traffic continues through CP-2/CP-3
- scheduler/controller-manager leader election moves if necessary
- 3-member etcd becomes 2-member available quorum
- cluster remains operational

However:

```text
3-member etcd
quorum = 2

lose 1 -> still operational
lose 2 -> quorum lost
```

So HA API servers alone are not enough; **etcd quorum is critical**.

---

# 19. kubeadm Upgrade Strategy

Do not upgrade every node simultaneously.

Use a rolling sequence:

```text
Prechecks / backup
       |
       v
Upgrade kubeadm on CP-1
       |
kubeadm upgrade plan
       |
kubeadm upgrade apply <target>
       |
upgrade kubelet/kubectl CP-1
       |
       v
CP-2
kubeadm upgrade node
upgrade kubelet
       |
       v
CP-3
kubeadm upgrade node
upgrade kubelet
       |
       v
Workers one by one
cordon/drain
kubeadm upgrade node
upgrade kubelet
uncordon
       |
       v
Post-upgrade validation
```

---

# 20. Pre-Upgrade Checks

Before upgrading:

- verify supported Kubernetes version path
- read release notes / API removals
- confirm version-skew policy
- verify all nodes are Ready
- verify control plane is healthy
- verify etcd health
- take etcd backup
- verify CNI/CSI compatibility
- verify ingress/monitoring compatibility
- verify PDBs and workload redundancy
- ensure enough capacity for drained workloads

Useful checks:

```bash
kubectl get nodes
kubectl get pods -A
kubectl get --raw='/readyz?verbose'
```

Take an etcd snapshot according to your etcd topology/runbook before a control-plane upgrade.

---

# 21. Upgrade First Control-Plane Node

First, upgrade the `kubeadm` package on CP-1 to the target supported version.

Then:

```bash
sudo kubeadm upgrade plan
```

This checks whether the cluster can be upgraded and shows available target versions.

Apply the target:

```bash
sudo kubeadm upgrade apply <TARGET_VERSION>
```

Example shape:

```bash
sudo kubeadm upgrade apply v1.xx.y
```

Only the **first control-plane node** runs `kubeadm upgrade apply`.

---

# 22. What kubeadm upgrade apply Does

Internally:

```text
kubeadm upgrade apply
      |
      +--> preflight / cluster health checks
      |
      +--> version-skew validation
      |
      +--> verify required control-plane images
      |
      +--> update static Pod manifests
      |
      +--> kubelet restarts changed control-plane static Pods
      |
      +--> update kubeadm cluster config
      |
      +--> update kubelet config
      |
      +--> maintain bootstrap-token/RBAC config
      |
      +--> upgrade CoreDNS/kube-proxy when appropriate
      |
      +--> perform certificate renewal when applicable
```

The static Pod manifest mechanism is important:

```text
kubeadm updates
/etc/kubernetes/manifests/kube-apiserver.yaml
                  |
                  v
               kubelet
                  |
                  v
old API Pod stopped
new API Pod created
```

---

# 23. Upgrade kubelet on First Control Plane

After the kubeadm control-plane upgrade, upgrade `kubelet` and optionally `kubectl` using your Linux package manager.

Typical sequence:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Validate CP-1 before touching CP-2.

---

# 24. Upgrade Additional Control-Plane Nodes

On CP-2 and CP-3:

1. upgrade the kubeadm binary/package
2. run:

```bash
sudo kubeadm upgrade node
```

3. upgrade kubelet/kubectl packages
4. restart kubelet
5. validate node/API health

Do **not** run `kubeadm upgrade apply` on every control-plane node.

Remember:

```text
First CP:
kubeadm upgrade apply

Remaining CPs:
kubeadm upgrade node
```

---

# 25. Upgrade Worker Nodes

For each worker, one at a time:

```bash
kubectl cordon worker-1
kubectl drain worker-1 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Then on that worker:

1. upgrade kubeadm package
2. run:

```bash
sudo kubeadm upgrade node
```

3. upgrade kubelet
4. restart kubelet

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

Back on admin host:

```bash
kubectl uncordon worker-1
```

Then move to the next worker.

---

# 26. Why Drain Workers During Upgrade

```text
worker
  |
cordon
  |
no new pods scheduled
  |
drain
  |
evict movable workloads
  |
upgrade node
  |
restart kubelet
  |
uncordon
  |
scheduling resumes
```

PDBs help prevent too many replicas of an application from being disrupted at once.

Always verify spare capacity before draining a production worker.

---

# 27. Why Control Plane Remains Available During Upgrade

With 3 control-plane nodes:

```text
Upgrade CP-1
   |
LB sends API traffic to CP-2 / CP-3
   |
CP-1 returns
   |
Upgrade CP-2
   |
LB uses CP-1 / CP-3
   |
CP-2 returns
   |
Upgrade CP-3
```

The same rolling principle protects API availability.

But you must also protect etcd quorum.

Do not take down multiple stacked control-plane/etcd nodes together.

---

# 28. Upgrade Order — Easy Memory

```text
1. Backup / health checks
2. CP-1 kubeadm
3. kubeadm upgrade plan
4. kubeadm upgrade apply
5. CP-1 kubelet
6. CP-2 kubeadm -> upgrade node -> kubelet
7. CP-3 kubeadm -> upgrade node -> kubelet
8. Worker-1 drain -> upgrade node -> kubelet -> uncordon
9. Worker-N repeat
10. Validate
```

---

# 29. Post-Upgrade Validation

Check:

```bash
kubectl get nodes -o wide
kubectl get pods -A
kubectl get componentstatuses 2>/dev/null || true
kubectl get --raw='/readyz?verbose'
kubectl get --raw='/livez?verbose'
```

Validate:

- all expected nodes Ready
- kube-system Pods healthy
- API available through LB endpoint
- CoreDNS healthy
- CNI healthy
- kube-proxy/eBPF dataplane healthy
- CSI healthy
- ingress working
- workloads healthy
- no excessive evictions
- etcd quorum/health good

---

# 30. Key Interview Questions and Answers

## Q1. What is the difference between kubeadm init and kubeadm join?

`kubeadm init` bootstraps the first Kubernetes control-plane node and creates cluster-level PKI/configuration.

`kubeadm join` bootstraps a new node and joins it to an existing cluster.

Worker:

```text
kubeadm join
```

Control plane:

```text
kubeadm join --control-plane --certificate-key ...
```

---

## Q2. Does kubeadm create the load balancer?

No.

You must provision the load balancer/VIP separately.

kubeadm is then configured with:

```yaml
controlPlaneEndpoint: "k8s-api.example.com:6443"
```

---

## Q3. Does kubeadm install the CNI?

No.

After `kubeadm init`, install a CNI such as Calico or Cilium.

Until networking is configured, nodes/system Pods may not become fully Ready.

---

## Q4. What starts the API server if the API server does not exist yet?

The **kubelet**.

kubeadm writes:

```text
/etc/kubernetes/manifests/kube-apiserver.yaml
```

The kubelet watches that directory and asks the container runtime to launch the API server as a static Pod.

---

## Q5. How does a worker securely join?

```text
bootstrap token
     +
CA cert hash
     |
     v
secure discovery
     |
TLS bootstrap / CSR
     |
kubelet client certificate
```

---

## Q6. How does another control plane join?

It follows worker-style discovery/TLS bootstrap plus:

```text
--control-plane
     +
--certificate-key
```

It then creates local control-plane static Pods and, in stacked topology, adds a new etcd member.

---

## Q7. How is API HA provided?

Not by kubeadm alone.

```text
DNS/VIP
  |
Load Balancer
  |
3 API servers
```

Clients use the stable `controlPlaneEndpoint`.

---

## Q8. If one master fails, what happens?

With 3 control planes and 3 stacked etcd members:

- LB stops routing to failed API server
- remaining API servers continue serving traffic
- scheduler/controller-manager leader re-elects if needed
- etcd still has 2/3 quorum
- cluster remains available

---

## Q9. What if two of three etcd members fail?

Quorum is lost.

For a 3-member etcd cluster:

```text
majority = 2
```

Only one surviving member is not enough for normal quorum-based operation.

---

## Q10. How do you upgrade a kubeadm cluster?

```text
First CP:
kubeadm upgrade plan
kubeadm upgrade apply <version>

Other CPs:
kubeadm upgrade node

Workers:
cordon
drain
kubeadm upgrade node
upgrade/restart kubelet
uncordon
```

Upgrade sequentially and preserve etcd/control-plane quorum.

---

# 31. 30-Second Interview Pitch

> In a production kubeadm design I would normally use three control-plane nodes with stacked etcd behind a TCP load balancer. The load balancer DNS or VIP is configured as kubeadm's controlPlaneEndpoint. I bootstrap the first node using kubeadm init, which performs preflight checks, generates PKI and kubeconfigs, writes the etcd and control-plane static Pod manifests, configures bootstrap RBAC and produces worker/control-plane join commands. Additional masters use kubeadm join with the control-plane flag and certificate key, while workers use the normal join command. Worker joins use bootstrap-token discovery, CA verification and TLS bootstrap so kubelets receive their own certificates. For upgrades, I back up etcd and validate cluster health, run kubeadm upgrade apply on the first control plane, kubeadm upgrade node on the other control planes, and then drain and upgrade workers sequentially, keeping API availability and etcd quorum throughout.

---

# 32. Quick Memory Diagram

```text
FIRST CONTROL PLANE
kubeadm init
    |
PKI
    |
static Pods
    |
etcd + API + scheduler + controller
    |
join commands
    |
    +-----------------------------+
    |                             |
    v                             v
CONTROL PLANE JOIN             WORKER JOIN
--control-plane                token + CA hash
--certificate-key                    |
    |                                v
API + scheduler +                kubelet TLS
controller + etcd                 bootstrap
    |                                |
    +---------------+----------------+
                    |
                    v
             HA Kubernetes Cluster
                    |
                    v
              UPGRADE FLOW
                    |
 First CP: upgrade apply
 Other CP: upgrade node
 Workers: drain -> upgrade node -> kubelet -> uncordon
```

---

# 33. References

- Kubernetes kubeadm documentation
- kubeadm init reference
- kubeadm join reference
- Creating Highly Available Clusters with kubeadm
- kubeadm HA topology options
- Upgrading kubeadm clusters

