# Linux Interview Refresher — Networking, Routing, Namespaces, and cgroups

This guide focuses on Linux fundamentals that matter most for Senior Platform / DevOps / Kubernetes interviews.

The goal is to understand how Linux moves packets, isolates processes, and controls resources, because Kubernetes and containers are built on these primitives.

## 1. Linux Networking Mental Model

    Application
        ↓
    Socket / Port
        ↓
    Linux TCP/IP Stack
        ↓
    Routing Decision
        ↓
    Firewall / NAT
        ↓
    Network Interface
        ↓
    Physical / Virtual Network

For containers and Kubernetes:

    Application in Pod
           ↓
       Pod eth0
           ↓
       veth pair
           ↓
    Host / CNI networking
           ↓
    Routing / eBPF / iptables
           ↓
       Host NIC
           ↓
        Network

## 2. Interfaces and IP Addresses

Primary command:

    ip addr

Typical interfaces:

    lo
    eth0
    cni0
    vethxxxx

Example:

    eth0: 10.10.1.20/24

Meaning:

    Host IP     = 10.10.1.20
    Network     = 10.10.1.0/24

Hosts in the same subnet can normally communicate directly. Traffic outside the subnet needs a route, usually through a default gateway.

## 3. Linux Routing

Check the routing table:

    ip route

Example:

    default via 10.10.1.1 dev eth0
    10.10.1.0/24 dev eth0 src 10.10.1.20
    10.244.1.0/24 dev cni0

Interpretation:

    Destination 10.10.1.x
            ↓
        use eth0 directly

    Destination 10.244.1.x
            ↓
        use cni0

    Everything else
            ↓
        gateway 10.10.1.1

Linux uses longest-prefix match. If routes exist for /8, /16, and /24 and the destination matches all three, the /24 route wins because it is the most specific.

Very useful command:

    ip route get 10.10.5.20

It tells you the chosen interface, gateway, and source IP.

## 4. Packet Leaving a Linux Host

Example:

    Host IP      10.10.1.20
    Destination  8.8.8.8
    Gateway      10.10.1.1

Flow:

    Application
        ↓
    creates socket
        ↓
    kernel sees destination 8.8.8.8
        ↓
    routing lookup
        ↓
    default route selected
        ↓
    next hop 10.10.1.1
        ↓
    resolve gateway MAC
        ↓
    frame leaves eth0

The Ethernet destination is the gateway MAC, while the IP destination remains 8.8.8.8 until NAT changes it somewhere.

## 5. ARP / Neighbor Resolution

IPv4 ARP maps an IP address to a MAC address on the local network.

    10.10.1.1
        ↓
    aa:bb:cc:dd:ee:ff

Check:

    ip neigh

Typical flow:

    Who has 10.10.1.1?
            ↓
        ARP request
            ↓
        gateway replies
            ↓
        neighbor cache updated

If routing looks correct but a same-subnet host is unreachable, check interface state, subnet mask, neighbor table, VLAN/network, and firewall.

## 6. TCP Connection Flow

    Client                         Server
      |                              |
      | -------- SYN --------------> |
      | <------ SYN-ACK ------------ |
      | -------- ACK --------------> |
      |                              |
      |      Connection established  |

Check sockets:

    ss -lntp
    ss -ant

Important states:

    LISTEN
    SYN_SENT
    SYN_RECV
    ESTABLISHED
    TIME_WAIT
    CLOSE_WAIT

Repeated SYN_SENT normally means the client sent a SYN but did not receive the expected SYN-ACK. Investigate route, firewall, server availability, listener, LB/security rules, and return path.

## 7. Listening on 127.0.0.1 vs 0.0.0.0

If a service listens on:

    127.0.0.1:8080

only local connections can reach it.

If it listens on:

    0.0.0.0:8080

it listens on all IPv4 interfaces.

This explains many cases where localhost works but remote access fails.

## 8. DNS Resolution

Simplified Linux flow:

    Application
        ↓
    resolver
        ↓
    /etc/nsswitch.conf
        ↓
    /etc/hosts and/or DNS
        ↓
    /etc/resolv.conf

Useful commands:

    cat /etc/resolv.conf
    cat /etc/nsswitch.conf
    dig example.com
    getent hosts example.com

Troubleshooting rule:

    hostname fails
         ↓
    direct IP works
         ↓
    likely DNS/name-resolution issue

Kubernetes:

    Pod
     ↓
    Cluster DNS Service
     ↓
    CoreDNS
     ↓
    Upstream DNS

## 9. NAT

DNAT changes the destination.

    Before:
    20.30.40.50:443

    After:
    10.244.2.8:443

Used in Services, NodePort, load-balancer flows, and port forwarding.

SNAT changes the source.

    Pod 10.244.1.10
           ↓
          SNAT
           ↓
    Node 10.10.1.20

External systems then see the translated source rather than the original Pod IP.

## 10. iptables Conceptual Flow

    Packet arrives
        ↓
    PREROUTING
        ↓
    Routing decision
       / \
      /   \
    Local  Forwarded
      |       |
    INPUT   FORWARD
      |       |
    Process   |
      |       |
    OUTPUT    |
       \     /
      POSTROUTING
          ↓
        leaves

Key chains:

    PREROUTING
    INPUT
    FORWARD
    OUTPUT
    POSTROUTING

Typical NAT association:

    DNAT → PREROUTING
    SNAT → POSTROUTING

In kube-proxy iptables mode you may see:

    KUBE-SERVICES
    KUBE-NODEPORTS
    KUBE-SVC-xxxx
    KUBE-SEP-xxxx

## 11. Kubernetes Service Translation

Example Service:

    ClusterIP 10.96.20.100:8080

Endpoints:

    10.244.1.10:8080
    10.244.2.11:8080

Flow:

    Application
         ↓
    10.96.20.100:8080
         ↓
    kube-proxy programmed dataplane
         ↓
    Service rule
         ↓
    endpoint selected
         ↓
    DNAT
         ↓
    10.244.2.11:8080

Important interview point: kube-proxy normally programs the dataplane; the kernel handles the packet.

## 12. tcpdump — Proving Where Traffic Stops

Example:

    ss -lntp | grep 443
    tcpdump -i any port 443

If no SYN arrives, investigate routing, firewall, security group, LB, NACL, or upstream network.

If SYN arrives but there is no SYN-ACK, investigate listener, host firewall, or local networking.

If SYN-ACK leaves but the client never completes the handshake, investigate return routing, stateful firewalls, or NAT.

## 13. Asymmetric Routing

Request:

    Client
      ↓
    Router-A
      ↓
    Server

Reply:

    Server
      ↓
    Router-B
      ↓
    Client

Stateful devices may reject this. Always check both forward path and return path.

## 14. Network Namespace

A network namespace gives processes an isolated networking stack.

Each network namespace can have its own:

- interfaces
- IP addresses
- routing table
- ARP/neighbor table
- sockets
- firewall rules

Diagram:

    Host Network Namespace
    --------------------------------
    eth0
    10.10.1.20
    routing table
    sockets
    --------------------------------

    Pod Network Namespace
    --------------------------------
    eth0
    10.244.1.5
    its own routes
    its own sockets
    --------------------------------

These are separate views of networking while using the same Linux kernel.

## 15. Why Containers Need Network Namespaces

Without network namespaces:

    Container-A
    Container-B
    Host processes
          ↓
    shared interfaces and ports

With namespaces:

    Host
     |
     +-- Host network namespace
     |
     +-- Container-A namespace
     |
     +-- Container-B namespace

Each container can have its own eth0, IP, routes, and port 8080 without conflicting.

## 16. Manual Network Namespace Example

Create:

    ip netns add ns1

List:

    ip netns list

Run inside:

    ip netns exec ns1 ip addr

This proves a namespace is not a VM. It is isolation provided by the same kernel.

## 17. veth Pair

A veth pair acts like a virtual Ethernet cable.

    end-A  <================>  end-B

For a Pod:

    Pod Network Namespace
            |
           eth0
            |
         veth pair
            |
       host-side veth
            |
      Host / CNI networking

Diagram:

    Pod
    10.244.1.5
       |
      eth0
       |
     veth-A
     =======
     veth-B
       |
      Host

Packets entering one end exit the other.

## 18. Namespace + veth

    +--------------------------+
    | Pod namespace            |
    |                          |
    | App :8080                |
    |    |                     |
    | eth0 10.244.1.5          |
    +----|---------------------+
         |
       veth pair
         |
    +----|---------------------+
    | Host namespace           |
    | host-side veth           |
    | routing / CNI            |
    | eth0 10.10.1.20          |
    +----|---------------------+
         |
      Physical Network

Flow:

    Pod app
      ↓
    pod eth0
      ↓
    veth
      ↓
    host namespace
      ↓
    route
      ↓
    host NIC / another Pod

## 19. Linux Bridge

A Linux bridge behaves roughly like a software Layer-2 switch.

    Container A
        |
      veth
        |
        +------+
               |
          Linux Bridge
               |
        +------+
        |
      veth
        |
    Container B

Some container networks use a bridge. Kubernetes CNIs can instead use native routes, overlay tunnels, VXLAN, cloud routing, or eBPF.

## 20. Linux Namespace Types

Important namespaces:

    PID     process isolation
    NET     network isolation
    MNT     mount/filesystem view
    UTS     hostname/domain
    IPC     IPC resources
    USER    UID/GID mapping
    CGROUP  cgroup namespace view

PID namespace example:

    Host PID namespace
    -----------------------------
    PID 1       systemd
    PID 24871   nginx
            |
            +------------------+
                               |
    Container PID namespace    |
    ---------------------------|
    PID 1       nginx <--------+

Same process, different namespace view.

## 21. Mount Namespace

Host view:

    /
    ├── etc
    ├── var
    └── data

Container view:

    /
    ├── app
    ├── etc
    └── tmp

Same kernel, different mount view.

## 22. Namespace vs Virtual Machine

VM:

    Guest application
    Guest libraries
    Guest kernel
          ↓
       Hypervisor
          ↓
         Host

Container:

    Container-A      Container-B
       App              App
       libs             libs
    namespaces       namespaces
          \             /
           Shared Linux Kernel
                   |
                  Host

Namespaces isolate; they do not provide a separate kernel.

## 23. cgroups

Namespaces answer:

    What can the process see?

cgroups answer:

    How much resource can the process use?

cgroups can control/account for:

- CPU
- memory
- I/O
- process count

Diagram:

    Linux Host
    CPU 16 cores
    RAM 64 GB
         |
         +------------------+
         |                  |
      cgroup A           cgroup B
      CPU limit          CPU limit
      memory limit       memory limit
         |                  |
    Container-A        Container-B

## 24. Kubernetes and cgroups

Kubernetes configuration:

    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1
        memory: 512Mi

Flow:

    Kubernetes resource config
              ↓
            kubelet
              ↓
       container runtime
              ↓
         Linux cgroups
              ↓
        kernel enforcement

Linux ultimately enforces the runtime constraints.

## 25. CPU cgroups

If a container has a CPU limit, the kernel can throttle it when it exhausts its CPU quota.

    Application requests CPU
             ↓
        cgroup CPU quota
             ↓
         quota exhausted
             ↓
      kernel throttles task

A Pod can therefore be CPU-throttled even when the node still has some idle CPU.

## 26. CPU Request vs CPU Limit

Request:

    scheduler placement / capacity accounting

Limit:

    runtime upper bound enforced using Linux resource controls

Do not describe requests and limits as the same thing.

## 27. Memory cgroups

Example:

    memory limit = 512 MiB

Flow:

    container allocates memory
             ↓
      reaches cgroup limit
             ↓
       memory pressure
             ↓
      cgroup OOM handling
             ↓
       process killed

Kubernetes may report:

    OOMKilled

Check:

    kubectl describe pod <pod>

## 28. Container OOM vs Host OOM

Container-level example:

    Host RAM = 64 GB
    Container limit = 512 MiB

    Container exceeds 512 MiB
             ↓
       cgroup limit hit
             ↓
      container process killed

The host can still have free RAM.

Host-level OOM:

    Host memory exhausted
             ↓
       global OOM condition
             ↓
       kernel chooses process
             ↓
          process killed

Check:

    dmesg | grep -i oom
    journalctl -k | grep -i oom

## 29. cgroup Hierarchy

Modern Linux commonly uses cgroup v2.

Conceptually:

    /
    ├── system.slice
    ├── user.slice
    └── kubepods.slice
          |
          +-- pod-A
          |    |
          |    +-- container-1
          |    +-- container-2
          |
          +-- pod-B
               |
               +-- container-1

Exact paths vary by runtime, distribution, and systemd setup.

cgroups are hierarchical; child processes inherit constraints from parent groups.

## 30. systemd and cgroups

systemd also organizes services using cgroups.

    system.slice
       |
       +-- sshd.service
       +-- containerd.service
       +-- kubelet.service

cgroups are a general Linux feature, not something invented for containers.

## 31. Namespace + cgroup = Core Container Model

    +------------------- Container -------------------+
    |                                                |
    |      Namespaces                 cgroups         |
    |          |                         |            |
    |   What can it see?         What can it use?    |
    |          |                         |            |
    | PID / NET / MNT / UTS      CPU / Memory / I/O |
    +------------------------------------------------+

Strong interview answer:

A container is fundamentally a normal Linux process isolated with namespaces and constrained/accounted for with cgroups, combined with filesystem isolation and security controls.

## 32. Kubernetes Pod Networking from Linux Perspective

A Pod normally gets a network namespace.

    Pod
    +--------------------------------+
    | Network Namespace              |
    |                                |
    | container-A                    |
    | container-B                    |
    |                                |
    | Shared Pod IP 10.244.1.5       |
    | Shared network stack           |
    +---------------|----------------+
                    |
                  eth0
                    |
                 veth pair
                    |
    +---------------|----------------+
    | Node Network Namespace         |
    | routing / CNI / eBPF/iptables  |
    | eth0 10.10.1.20                |
    +---------------|----------------+
                    |
                 Network

Containers in the same Pod share the Pod network namespace, so they can communicate through localhost.

## 33. Pod-to-Pod — Same Node

Simplified:

    Pod-A
    10.244.1.5
       |
      eth0
       |
      veth
       |
    Node networking
       |
    route / bridge / eBPF
       |
      veth
       |
      eth0
       |
    Pod-B
    10.244.1.6

Exact datapath depends on the CNI.

## 34. Pod-to-Pod — Different Nodes

    Pod-A
    10.244.1.5
       |
      veth
       |
    Node-1
    10.10.1.20
       |
    CNI datapath
       |
    Physical / Overlay Network
       |
    Node-2
    10.10.1.21
       |
      veth
       |
    Pod-B
    10.244.2.7

Possible mechanisms include native routing, VXLAN, cloud VNet/VPC routing, eBPF, or another overlay.

Key principle:

Linux namespaces isolate the Pod; the CNI connects those namespaces into the cluster network.

## 35. How CNI Fits

    kubelet
       ↓
    container runtime
       ↓
    Pod sandbox/network namespace
       ↓
    CNI plugin
       ↓
    interface configured
       ↓
    Pod IP assigned
       ↓
    routes/dataplane configured
       ↓
    Pod network-ready

CNI wires the Pod network namespace into the cluster network.

## 36. Networking Troubleshooting Flow

    1. Is process running?
           ↓
    2. Is port listening?
           ↓
    3. Does localhost work?
           ↓
    4. Correct IP/interface?
           ↓
    5. Correct route?
           ↓
    6. ARP/neighbor working?
           ↓
    7. Firewall/NAT correct?
           ↓
    8. Does packet arrive?
           ↓
    9. Does reply leave?
           ↓
    10. Is return route correct?

Commands:

    ps -ef
    systemctl status <service>
    ss -lntp
    curl localhost:<port>
    ip addr
    ip route
    ip route get <destination>
    ip neigh
    iptables -L -n
    iptables -t nat -L -n
    tcpdump -i any host <IP>

## 37. Kubernetes Node Networking Troubleshooting

    Pod healthy?
       ↓
    Pod IP?
       ↓
    Service endpoints?
       ↓
    Node route?
       ↓
    CNI healthy?
       ↓
    kube-proxy / eBPF dataplane?
       ↓
    Node firewall?
       ↓
    External LB/network?

Useful commands:

    kubectl get pod -o wide
    kubectl get svc
    kubectl get endpointslices
    kubectl describe pod <pod>
    ip addr
    ip route
    ss -lntp
    tcpdump -i any

For kube-proxy iptables mode:

    iptables-save | grep KUBE

For Cilium/eBPF, use the Cilium dataplane tooling rather than assuming iptables.

## 38. Interview Scenarios

### Ping works but 443 fails

Layer-3 reachability exists. Investigate Layer 4 and the application:

    ss -lntp | grep 443
    nc -vz server 443
    tcpdump -i any port 443

### IP works but hostname fails

Think DNS:

    cat /etc/resolv.conf
    dig hostname
    getent hosts hostname

### Pod can reach node but not another Pod

Think CNI, routing, NetworkPolicy, overlay, and node firewall.

### Service ClusterIP fails but direct Pod IP works

Think Service dataplane: kube-proxy/IPVS/iptables/eBPF, selectors, and EndpointSlices.

### Pod is OOMKilled while node has free memory

Think container cgroup memory limit, not necessarily host memory exhaustion.

### Node CPU is free but application is throttled

Think CPU cgroup quota / container CPU limit.

## 39. Interview Answers Worth Memorizing

### What is a namespace?

A Linux namespace isolates a process's view of system resources. Containers use namespaces such as PID, network, mount, UTS, IPC, and user namespaces so processes can have isolated process trees, networking, filesystems, hostnames, and identities while sharing the same kernel.

### What is a cgroup?

A cgroup is a Linux kernel mechanism for grouping processes and accounting for or limiting resources such as CPU, memory, I/O, and process count. Kubernetes resource limits are ultimately enforced through Linux cgroups.

### Namespace vs cgroup?

    Namespace = isolation / visibility
    cgroup    = resource control / accounting

### How does a Pod get networking?

Kubernetes creates a Pod sandbox with a network namespace. The CNI connects that namespace to node networking, commonly using virtual interfaces such as veth pairs, assigns the Pod IP, and configures routes or dataplane state.

### What does kube-proxy do?

kube-proxy watches Services and EndpointSlices and programs the node service dataplane, traditionally through iptables or IPVS. The Linux kernel then handles forwarding and DNAT. Cilium can replace kube-proxy service handling with eBPF.

### How does Linux choose a route?

The kernel performs a routing-table lookup and selects the most specific matching route. That route determines the outgoing interface, next-hop gateway, and typically the source IP. The command ip route get <destination> is one of the best ways to inspect that decision.

## 40. Must-Know Commands

Networking:

    ip addr
    ip link
    ip route
    ip route get <IP>
    ip neigh
    ss -lntp
    ss -ant
    dig
    getent hosts
    curl -v
    nc -vz
    tcpdump
    traceroute

Namespaces:

    ip netns list
    ip netns add
    ip netns exec
    lsns
    nsenter

Processes/cgroups:

    ps -ef
    systemctl status
    systemd-cgls
    systemd-cgtop
    cat /proc/<PID>/cgroup

Kernel:

    sysctl
    dmesg
    journalctl -k

## 41. Final Mental Model

                         Linux Container / Pod
                                  |
                  +---------------+---------------+
                  |                               |
            Namespace Isolation             cgroup Control
                  |                               |
         PID / NET / MNT / UTS              CPU / Memory / I/O
                  |                               |
                  +---------------+---------------+
                                  |
                           Linux Process
                                  |
                               Socket
                                  |
                        Pod / Namespace eth0
                                  |
                               veth pair
                                  |
                         Node networking
                                  |
                        Routing / NAT / CNI
                                  |
                               Node NIC
                                  |
                               Network

For senior Platform/Kubernetes interviews, be strongest in:

1. How Linux chooses a route.
2. How to prove where a packet is being dropped.
3. DNAT versus SNAT.
4. Why kube-proxy does not literally receive every Service packet.
5. What a network namespace gives a Pod.
6. How veth connects a Pod namespace to node networking.
7. Namespace versus cgroup.
8. How Kubernetes CPU/memory limits map to cgroups.
9. Container OOM versus host OOM.
10. How CNI builds Pod networking from Linux primitives.


---

## 42. systemd for Kubernetes and Container Runtimes

systemd is the service manager on most enterprise Linux distributions used for Kubernetes nodes.

Typical Kubernetes services:

```text
kubelet.service
containerd.service
crio.service
sshd.service
chronyd.service
```

Useful commands:

```bash
systemctl status kubelet
systemctl status containerd
systemctl status crio

systemctl restart kubelet
systemctl restart containerd

systemctl enable kubelet
systemctl enable containerd

journalctl -u kubelet
journalctl -u containerd
journalctl -u crio

journalctl -u kubelet -f
```

If a service unit is changed:

```bash
systemctl daemon-reload
systemctl restart <service>
```

Strong interview point:

> Kubernetes control-plane and worker services ultimately depend on Linux service management. If kubelet or the container runtime is down, the node cannot manage Pods correctly.

---

## 43. Container Runtime Architecture

Modern Kubernetes uses the Container Runtime Interface (CRI) to communicate with a container runtime.

```text
Kubernetes
   |
 kubelet
   |
   | CRI / gRPC
   v
containerd / CRI-O
   |
   v
OCI runtime
runc / crun
   |
   v
Linux kernel
namespaces + cgroups + filesystem
```

The kubelet does not directly execute containers.

It asks the CRI-compatible runtime to create Pod sandboxes, pull images, start containers, and stop containers.

---

## 44. CRI vs OCI

This distinction is worth memorizing.

### CRI

CRI is the Kubernetes interface between:

```text
kubelet
   |
   v
container runtime
```

Examples of CRI-compatible runtimes:

- containerd
- CRI-O

### OCI

OCI defines standards around:

- container images
- container runtime specification

A low-level OCI runtime such as `runc` or `crun` actually creates the Linux container process.

Mental model:

```text
Kubernetes
   |
   | CRI
   v
containerd / CRI-O
   |
   | OCI
   v
runc / crun
   |
   v
Linux process
```

Interview answer:

> CRI is the interface Kubernetes uses to talk to the container runtime, while OCI defines common image and runtime standards used by the container ecosystem underneath.

---

## 45. containerd

containerd is a widely used Kubernetes container runtime.

Typical socket:

```text
/run/containerd/containerd.sock
```

Typical configuration:

```text
/etc/containerd/config.toml
```

Useful commands:

```bash
systemctl status containerd
journalctl -u containerd
crictl info
crictl ps -a
crictl pods
crictl images
```

A common kubelet runtime endpoint is conceptually:

```text
unix:///run/containerd/containerd.sock
```

### systemd cgroup driver

On modern systemd-based Kubernetes nodes, containerd commonly uses the systemd cgroup driver.

A common configuration concept is:

```text
SystemdCgroup = true
```

Why this matters:

```text
systemd manages host services
        +
kubelet/runtime use systemd cgroups
        |
        v
consistent cgroup hierarchy
```

A cgroup-driver mismatch can create node instability.

---

## 46. CRI-O

CRI-O is a lightweight container runtime built specifically around the Kubernetes CRI.

It is commonly associated with OpenShift.

Architecture:

```text
kubelet
   |
   | CRI
   v
CRI-O
   |
   v
runc / crun
   |
   v
Linux kernel
```

Typical service:

```bash
systemctl status crio
journalctl -u crio
```

You can still use `crictl` because it talks to the CRI endpoint.

Strong interview answer:

> containerd is a general-purpose container runtime widely used by Kubernetes distributions, while CRI-O is purpose-built for Kubernetes CRI and is used heavily in OpenShift.

---

## 47. crictl

`crictl` is one of the most useful tools for node-level Kubernetes runtime troubleshooting.

Useful commands:

```bash
crictl info
crictl pods
crictl ps
crictl ps -a
crictl images
crictl inspect <container-id>
crictl logs <container-id>
crictl inspectp <pod-id>
```

Troubleshooting flow:

```text
Pod failing
   |
kubectl describe
   |
node/runtime suspected
   |
ssh to node
   |
crictl pods / ps -a
   |
crictl inspect / logs
   |
journalctl runtime + kubelet
```

---

## 48. ctr vs crictl

With containerd you may also see `ctr`.

Important distinction:

```text
crictl
  |
  -> Kubernetes CRI troubleshooting

ctr
  |
  -> low-level containerd client
```

For Kubernetes troubleshooting, prefer `crictl` first because it reflects the CRI objects kubelet works with.

---

## 49. Pod Sandbox Concept

Before Kubernetes starts application containers in a Pod, the runtime creates a Pod sandbox.

The sandbox provides the shared Pod-level environment, especially networking.

```text
Pod
 |
 +-- sandbox
 |     |
 |     +-- network namespace
 |
 +-- app container
 |
 +-- sidecar container
```

Containers in the same Pod share the Pod's network namespace.

If sandbox creation fails, Pods may remain in:

```text
ContainerCreating
CreatePodSandboxError
```

Likely areas:

- CNI
- runtime
- network namespace creation
- registry/image pull
- runtime socket
- node filesystem

---

## 50. Container Runtime Troubleshooting

Common failure:

```text
container runtime is down
```

Check:

```bash
systemctl status containerd
journalctl -u containerd
crictl info
```

Common symptoms:

- Node NotReady
- kubelet reports runtime unavailable
- Pods do not start
- Pod sandbox creation fails

### Runtime socket issue

Check runtime endpoint and socket:

```bash
ls -l /run/containerd/containerd.sock
crictl info
```

### Image pull failure

Check:

```bash
crictl pull <image>
crictl images
journalctl -u containerd
```

Investigate:

- DNS
- proxy
- private registry auth
- CA trust
- image name/tag
- firewall

### Disk full

Check:

```bash
df -h
df -i
du -sh /var/lib/containerd/*
```

A node can fail even when CPU/memory are healthy if runtime storage is exhausted.

---

## 51. Linux Filesystem Fundamentals

Useful commands:

```bash
df -h
df -i
du -sh /var/*
lsblk
blkid
mount
findmnt
cat /etc/fstab
```

### df -h vs df -i

```text
df -h
  -> disk block usage

df -i
  -> inode usage
```

A filesystem can have free GB but still fail to create files if inodes are exhausted.

---

## 52. Mounts and /etc/fstab

List mounts:

```bash
mount
findmnt
```

Persistent mounts are normally defined in:

```text
/etc/fstab
```

Typical failure scenario:

```text
server reboot
   |
filesystem fails to mount
   |
container runtime data path missing
   |
kubelet/runtime fail
```

Always check underlying mounts when a node behaves differently after reboot.

---

## 53. XFS and ext4

Common Linux filesystems include:

```text
XFS
ext4
```

For Kubernetes/platform interviews, know at a high level:

- mount options matter
- filesystem full conditions affect kubelet/runtime
- inode exhaustion matters
- storage performance affects container startup and stateful workloads
- XFS/ext4 are common host filesystems

You do not normally need deep filesystem internals unless the role is Linux-specialist-heavy.

---

## 54. OverlayFS and Container Filesystems

Containers often use layered filesystems.

Conceptually:

```text
Image layer 1
Image layer 2
Image layer 3
     |
     +---- read-only layers
     |
Writable container layer
     |
     v
Container filesystem view
```

OverlayFS is commonly used underneath container runtimes.

Important consequence:

```text
container writable layer
!= persistent application storage
```

If the container is recreated, writable-layer data may disappear.

Persistent data should use Kubernetes volumes.

---

## 55. Linux Kernel Basics for Kubernetes

Kubernetes relies heavily on Linux kernel features.

Important areas:

- namespaces
- cgroups
- routing
- netfilter
- conntrack
- overlay networking
- filesystems
- capabilities
- seccomp
- SELinux/AppArmor
- kernel modules

Useful commands:

```bash
uname -r
lsmod
modprobe
sysctl -a
dmesg
journalctl -k
```

---

## 56. Important Kernel Modules

Common modules relevant to Kubernetes/container networking include:

```text
overlay
br_netfilter
```

Load:

```bash
modprobe overlay
modprobe br_netfilter
```

Check:

```bash
lsmod | grep overlay
lsmod | grep br_netfilter
```

Exact requirements depend on Kubernetes version and CNI.

---

## 57. Important sysctl Concepts

Useful checks:

```bash
sysctl net.ipv4.ip_forward
sysctl net.bridge.bridge-nf-call-iptables
```

IP forwarding is important when the node must route traffic between interfaces/namespaces.

Conceptually:

```text
Pod network
   |
Linux node
   |
another network
```

The kernel may need forwarding enabled.

Persistent sysctl configuration is commonly kept under:

```text
/etc/sysctl.conf
/etc/sysctl.d/
```

Apply:

```bash
sysctl --system
```

---

## 58. conntrack

Linux connection tracking maintains state for network flows.

Kubernetes Service/NAT implementations may rely on conntrack.

Useful command:

```bash
conntrack -L
```

Common issue:

```text
conntrack table full
        |
new connections fail/drop
        |
random-looking network failures
```

Check kernel logs and conntrack metrics/settings if the node experiences large-scale connection failures.

---

## 59. File Descriptor Limits

A process or node can fail because it runs out of file descriptors.

Useful commands:

```bash
ulimit -n
cat /proc/sys/fs/file-max
ls /proc/<PID>/fd | wc -l
```

Symptoms can include:

- cannot open files
- socket creation failures
- API/application instability

For high-scale Kubernetes nodes, OS-level limits matter.

---

## 60. Process and Memory Troubleshooting

Useful commands:

```bash
ps -ef
top
free -m
vmstat
pidstat
dmesg
journalctl -k
```

Typical reasoning:

```text
high load
  |
CPU?
Memory?
I/O?
blocked processes?
container cgroup?
host issue?
```

Do not assume every high load average means CPU saturation.

---

## 61. Linux I/O Troubleshooting

Useful commands:

```bash
iostat
iotop
vmstat
lsblk
df -h
```

Symptoms:

- slow image pulls
- slow container startup
- kubelet timeouts
- etcd latency
- stateful workload latency

For etcd especially, disk latency is critical.

---

## 62. OS Hardening for Kubernetes Nodes

For production/on-prem clusters, think in layers.

### Minimal OS footprint

- install only required packages
- disable unnecessary services
- remove unused software

### SSH hardening

- restrict root login
- use key-based authentication
- limit administrative access
- enforce MFA/jump-host controls where available

### Firewall

- open only required Kubernetes/runtime/CNI ports
- restrict management access
- avoid broad any/any rules

### Patch management

- regularly patch OS/kernel
- test compatibility before rollout
- use rolling node maintenance

### File permissions

Protect sensitive paths such as:

```text
/etc/kubernetes/
/var/lib/kubelet/
/etc/containerd/
/etc/crio/
/etc/ssl/
```

### Time synchronization

Use chrony/NTP.

Time drift can cause:

- certificate validation issues
- authentication problems
- etcd issues
- confusing logs

---

## 63. SELinux and AppArmor

Linux Mandatory Access Control adds another security layer beyond Unix permissions.

### SELinux

Common in RHEL/OpenShift environments.

Modes:

```text
Enforcing
Permissive
Disabled
```

Useful:

```bash
getenforce
sestatus
ausearch -m avc
```

Strong interview position:

> Do not disable SELinux as a first troubleshooting step in production. Identify the denial and fix the policy/configuration.

### AppArmor

Common on some Ubuntu-based environments.

It restricts process behavior through profiles.

---

## 64. auditd

Linux audit can capture security-relevant host activity.

Useful commands:

```bash
systemctl status auditd
ausearch
aureport
```

Audit evidence can be important in regulated environments.

Kubernetes audit logging is separate from Linux auditd; both may be used.

---

## 65. CIS Benchmarks

CIS provides security benchmark recommendations for:

- Linux OS
- Kubernetes
- container runtimes
- Kubernetes distributions

Typical areas:

- file permissions
- authentication
- API server settings
- kubelet settings
- audit policy
- encryption
- host hardening

Kubernetes-focused tool:

```text
kube-bench
```

Concept:

```text
CIS benchmark
    |
automated/manual check
    |
finding
    |
risk review
    |
remediation or documented exception
```

Do not blindly apply every recommendation without testing because some controls can conflict with platform requirements.

---

## 66. cgroup v1 vs cgroup v2

Modern Linux increasingly uses cgroup v2.

### cgroup v1

Historically, different controllers could have separate hierarchies.

### cgroup v2

Provides a unified hierarchy and improved resource-control model.

Conceptually:

```text
cgroup v2
   |
unified hierarchy
   |
system.slice
user.slice
kubepods.slice
```

Useful command:

```bash
stat -fc %T /sys/fs/cgroup/
```

On cgroup v2 systems you typically see:

```text
cgroup2fs
```

---

## 67. kubelet and Runtime cgroup Driver

The kubelet and container runtime should use compatible cgroup drivers.

Preferred modern model on systemd systems:

```text
systemd
  |
  +-- kubelet
  |
  +-- containerd / CRI-O
  |
  +-- Kubernetes Pod/container cgroups
```

A mismatch can lead to:

- resource accounting inconsistencies
- kubelet instability
- node issues

Interview line:

> On modern systemd-based Kubernetes nodes I would align kubelet and the runtime on the systemd cgroup driver and validate cgroup v2 compatibility.

---

## 68. Kernel / Runtime / Kubernetes Compatibility

Think of compatibility as a stack:

```text
Linux kernel
    |
cgroups / namespaces / modules
    |
containerd or CRI-O
    |
CRI
    |
kubelet
    |
Kubernetes version
```

Before upgrades, validate:

- supported Linux distribution/kernel
- cgroup version
- runtime version support
- kubelet/Kubernetes version skew
- CNI compatibility
- CSI compatibility
- GPU driver compatibility where relevant

Do not upgrade one layer blindly without checking the others.

---

## 69. Common Runtime/Node Interview Scenarios

### kubelet is running but node is NotReady

Check:

```text
kubelet logs
runtime health
CNI health
disk pressure
memory pressure
PID pressure
network
```

Commands:

```bash
systemctl status kubelet
journalctl -u kubelet
crictl info
kubectl describe node <node>
```

### Runtime is down

```bash
systemctl status containerd
journalctl -u containerd
```

Expected impact:

```text
existing containers may temporarily continue
but kubelet cannot reliably create/manage new containers
node becomes unhealthy/NotReady
```

### Node disk pressure

```bash
df -h
df -i
du -sh /var/lib/containerd/*
du -sh /var/lib/kubelet/*
```

Possible outcome:

- DiskPressure
- image garbage collection
- Pod evictions
- runtime failures

### Image pull works with podman but not Kubernetes

Check:

- runtime registry configuration
- runtime CA trust
- imagePullSecrets
- ServiceAccount
- runtime proxy config
- DNS on node
- `crictl pull`

---

## 70. Senior-Level Linux + Runtime Troubleshooting Flow

```text
1. Is node reachable?
       |
2. OS healthy?
       |
3. CPU/memory/disk/inodes?
       |
4. kubelet healthy?
       |
5. container runtime healthy?
       |
6. CRI responding?
       |
7. CNI healthy?
       |
8. DNS/network routes?
       |
9. registry/image access?
       |
10. cgroup/kernel/security issue?
```

Useful commands:

```bash
uptime
free -m
df -h
df -i
systemctl status kubelet
systemctl status containerd
journalctl -u kubelet
journalctl -u containerd
crictl info
crictl ps -a
ip addr
ip route
dmesg
journalctl -k
```

---

## 71. Must-Know Linux and Runtime Commands

### Services

```bash
systemctl status kubelet
systemctl status containerd
systemctl status crio
journalctl -u kubelet
journalctl -u containerd
```

### Runtime

```bash
crictl info
crictl pods
crictl ps -a
crictl images
crictl inspect
crictl logs
```

### Filesystem

```bash
df -h
df -i
du -sh
lsblk
findmnt
mount
```

### Kernel

```bash
uname -r
lsmod
modprobe
sysctl
dmesg
journalctl -k
```

### Process/resource

```bash
ps -ef
top
free -m
vmstat
iostat
pidstat
```

---

## 72. Container Runtime Interview Answers Worth Memorizing

### What is CRI?

CRI is the gRPC interface between kubelet and a container runtime such as containerd or CRI-O.

### What is OCI?

OCI defines standard container image and runtime specifications. Low-level runtimes such as runc implement the runtime specification.

### containerd vs CRI-O?

Both can satisfy the Kubernetes CRI requirement. containerd is a broader industry container runtime, while CRI-O is designed specifically for Kubernetes and is widely used in OpenShift.

### What happens if containerd goes down?

The kubelet loses communication with the runtime. Existing containers may continue running for some time, but the node cannot reliably create, delete, restart, or inspect containers, and the node can transition to an unhealthy/NotReady state.

### Why use crictl?

Because crictl talks directly to the CRI endpoint and is the preferred node-level troubleshooting tool for Kubernetes Pods/containers independent of whether the runtime is containerd or CRI-O.

### Why does cgroup driver matter?

The kubelet and runtime both manage resource hierarchy. On systemd-based nodes, aligning them on the systemd cgroup driver gives consistent resource management and avoids instability.

---

## 73. Updated Final Mental Model

```text
                    Kubernetes Pod
                         |
                kubelet on Linux node
                         |
                       CRI
                         |
             +-----------+-----------+
             |                       |
         containerd                 CRI-O
             |                       |
             +-----------+-----------+
                         |
                    runc / crun
                         |
                    Linux kernel
                         |
       +-----------------+------------------+
       |                 |                  |
   namespaces          cgroups          filesystem
       |                 |                  |
 isolation/view     resource control      OverlayFS
       |
network namespace
       |
veth / CNI
       |
routing / eBPF / iptables
       |
node NIC
```

For Point 3 — **Linux and Container Runtime** — be strongest in:

1. Linux networking and routing.
2. Namespaces and cgroups.
3. containerd / CRI-O architecture.
4. CRI vs OCI.
5. kubelet-to-runtime interaction.
6. crictl troubleshooting.
7. systemd and journalctl.
8. disk/inode/filesystem troubleshooting.
9. kernel modules and sysctl.
10. cgroup v2 and systemd cgroup driver.
11. OS hardening / SELinux / CIS.
12. kernel-runtime-Kubernetes compatibility.

