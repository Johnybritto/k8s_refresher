# OVN-Kubernetes vs Calico vs Cilium — Deep Comparison

This guide compares **OVN-Kubernetes, Calico, and Cilium** for Senior Platform / Kubernetes / OpenShift architecture and interview discussions.

It covers:
- architecture
- dataplane
- routing
- overlays
- kube-proxy behavior
- network policy
- encryption
- observability
- service mesh/L7
- multi-cluster
- virtualization
- performance characteristics
- operational complexity
- enterprise/commercial offerings

---

# 1. Core Architecture Comparison

| Area | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Fundamental technology | OVN + Open vSwitch + OpenFlow | Linux routing + iptables/nftables, optional eBPF, VPP | eBPF-first dataplane |
| Core philosophy | Virtualized SDN with logical switches/routers | IP routing/firewalling first; highly configurable | Networking/security/load-balancing logic in Linux kernel using eBPF |
| Main node component | ovnkube-node, OVN controller, OVS | Felix + optional BIRD + Typha | Cilium Agent |
| Dataplane programming | OVN logical flows → OVS OpenFlow | Felix programs routing/firewall/eBPF | Cilium agent loads eBPF programs/maps |
| Switching | OVS logical switching | Linux networking / optional eBPF/VPP | eBPF + Linux routing |
| Routing | OVN logical routers | Native Linux routing; BGP is first-class | Native routing or overlay; optional BGP control plane |
| Default overlay | Geneve | Flexible: VXLAN/IPIP/no overlay | VXLAN by default, Geneve optional |
| No-overlay mode | Possible, but not the main architectural model | Strong capability | Native routing mode supported |
| Encapsulation avoidance | Not primary design point | Strong focus | Supported |
| Kubernetes Service implementation | OVN has its own Service implementation | kube-proxy with standard dataplane; eBPF can replace it | Can fully replace kube-proxy with eBPF |
| Load-balancing implementation | OVN/OVS NAT + load-balancing rules | kube-proxy or Calico eBPF/VPP | eBPF maps, socket LB, XDP options, Maglev/DSR |
| L2/L3 virtualization | Very strong | Primarily routed IP model | Primarily L3/eBPF networking |
| Windows | Supported in OpenShift hybrid networking | Supported with limitations | Primarily Linux |
| IPv6 | Yes | Yes | Yes |
| Dual stack | Yes | Yes | Yes |

---

# 2. OVN-Kubernetes Architecture

    Pod
     |
    veth
     |
    OVS Bridge (br-int)
     |
    OpenFlow rules
     |
    OVN Logical Switch / Router
     |
    Geneve Tunnel
     |
    Physical NIC
     |
    Network
     |
    Remote Node
     |
    OVS
     |
    Pod

Control plane:

    Kubernetes Objects
           |
           v
    OVN-Kubernetes Controller
           |
           v
    OVN Northbound DB
           |
           v
    OVN northd
           |
           v
    OVN Southbound DB
           |
           v
    ovn-controller
           |
           v
    OpenFlow
           |
           v
    OVS

Troubleshooting concepts frequently include:

    OVN
    OVS
    br-int
    OpenFlow
    Geneve
    NBDB
    SBDB

OVN-Kubernetes translates Kubernetes networking objects into OVN logical network objects, which eventually program OVS.

---

# 3. Calico Architecture

Traditional routed dataplane:

    Pod
     |
    veth
     |
    Linux routing
     |
    iptables / nftables policy
     |
    Host route
     |
    BGP-learned route
     |
    eth0
     |
    Physical Network
     |
    Remote Node
     |
    route
     |
    Pod

Calico can treat nodes almost like routers.

Example:

    Node
       ↓ BGP
    Route Reflector
       ↓
    Top-of-Rack Router

or:

    Node
       ↓ eBGP
    ToR

Calico is particularly strong where direct routing and BGP integration with physical infrastructure are required.

---

# 4. Cilium Architecture

    Pod
     |
    veth / netkit
     |
    eBPF program
     |
    eBPF policy
     |
    eBPF Service LB
     |
    Linux routing
     |
    NIC
     |
    Network

Example Service flow:

    Application
       |
    connect()
       |
    eBPF socket hook
       |
    Service IP
       |
    backend lookup in BPF map
       |
    Pod backend

Instead of relying on a long iptables path, Cilium can perform Service translation very early in the network stack.

---

# 5. Routing Comparison

| Capability | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Native Linux routing | Not primary architecture | Core strength | Supported |
| BGP | Via broader OpenShift/FRR integration | Very mature/core capability | BGP Control Plane |
| Node-to-node BGP | Not normal internal model | Yes | BGP mainly advertises routes externally |
| ToR peering | Via surrounding routing architecture | Excellent fit | Supported |
| Route reflectors | External design | Common design | External routing architecture |
| Overlay | Geneve | VXLAN / IPIP | VXLAN / Geneve |
| Direct routing | Less central | Strong | Strong |
| Cloud-native IP routing | Yes through platform integration | Strong | Strong |
| Service advertisement | OpenShift integrations | BGP advertisement supported | BGP Service advertisements |
| Multiple IP pools | UDN/subnet concepts | IPPool CRDs | Multiple IPAM modes |

---

# 6. kube-proxy Comparison

| Area | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| kube-proxy required? | No | Depends on dataplane | No with kube-proxy replacement |
| Service handling | OVN/OVS | kube-proxy or eBPF/VPP | eBPF |
| ClusterIP | OVN | kube-proxy/eBPF | eBPF |
| NodePort | OVN | kube-proxy/eBPF | eBPF |
| LoadBalancer | OVN | kube-proxy/eBPF | eBPF |
| DSR | Architecture dependent | eBPF/VPP capable | Strong capability |
| XDP acceleration | No equivalent architecture | Some XDP capabilities | Yes |

Important nuance:

Calico is not simply "iptables-based" anymore. It also supports an eBPF dataplane that can replace kube-proxy.

Cilium is different because eBPF is its architectural foundation rather than an optional dataplane.

---

# 7. Network Security Comparison

| Capability | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Kubernetes NetworkPolicy | Yes | Yes | Yes |
| Ingress policy | Yes | Yes | Yes |
| Egress policy | Yes | Yes | Yes |
| Cluster-wide policy | AdminNetworkPolicy / BaselineAdminNetworkPolicy | GlobalNetworkPolicy / tiers | CiliumClusterwideNetworkPolicy + Kubernetes ClusterNetworkPolicy |
| Explicit deny | Supported through policy model | Yes | Yes |
| Policy priority | ANP priorities | Strong tier/order model | Policy tiers / deny precedence |
| Identity/label-based policy | Kubernetes selectors | Strong label model | Core security-identity model |
| Host firewall | OVN ACL/infrastructure policy model | HostEndpoint policies | Cilium Host Firewall |
| FQDN policy | EgressFirewall DNS support | Yes | Yes |
| L7 HTTP policy | Not a main OVN feature | Enterprise application-layer features | Strong L7 HTTP/gRPC/DNS policy |
| HTTP method/path | External tooling | Enterprise features | Yes |
| DNS-aware enforcement | Egress firewall DNS | Enterprise DNS policy | toFQDNs + DNS proxy |
| Microsegmentation | Strong | Very strong | Very strong |
| Policy audit/pre-stage | Logging/observability | Staged policy/policy preview | Audit/policy tooling |
| VM/bare-metal policy | Primarily through OpenShift ecosystem | Enterprise host/VM protection | Enterprise expanding into VM networking |

---

# 8. Encryption

| Capability | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| WireGuard | Not primary method | Yes | Yes |
| IPsec | Yes | Available depending on dataplane | Yes |
| Transparent encryption | Yes | Yes | Yes |
| Inter-node pod traffic | Yes | Yes | Yes |
| Built into CNI/platform | Yes | Yes | Yes |

---

# 9. Observability

| Feature | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Flow visibility | Yes | Yes | Hubble |
| Service map | OpenShift Network Observability | Enterprise Flow Visualizer | Hubble UI |
| Drop reason visibility | OVN policy logging / network observability | Enterprise | Excellent native visibility |
| DNS visibility | OpenShift Network Observability | Enterprise | Hubble |
| L7 protocol visibility | More limited | Enterprise | Strong |
| Packet capture | Standard tooling / OpenShift observability | Enterprise dynamic capture | Hubble/eBPF + packet tools |
| Prometheus | Yes | Yes | Yes |
| Grafana | Yes | Yes | Yes |
| Flow history | Backend dependent | Enterprise retention | Enterprise/Hubble platform options |
| Multi-cluster visibility | OpenShift ecosystem | Enterprise | ClusterMesh + enterprise |

---

# 10. Service Mesh and L7

| Capability | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Built-in service-mesh direction | No | Integrates with meshes | Yes |
| Ingress controller | Separate OpenShift Router/Ingress | Separate / enterprise gateway options | Cilium Ingress supported |
| Gateway API | Platform ecosystem | Enterprise capabilities | Major focus |
| Envoy | Not core OVN component | Can integrate | Integrated |
| Sidecarless L7 | No | Depends on solution | Yes |
| HTTP routing | External router/ingress | Gateway/application-layer features | Yes |
| gRPC policy/routing | External components | Enterprise integrations | Yes |
| mTLS/service identity | External mesh | Integration | Cilium service-mesh capabilities |

For Cilium L7:

    eBPF
      ↓
    selected flow
      ↓
    Envoy
      ↓
    HTTP / gRPC processing

Important:

Cilium does not mean "no proxy at all." L3/L4 can be handled with eBPF, while L7 processing commonly uses Envoy.

---

# 11. Multi-Cluster

| Capability | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Multi-cluster connectivity | OpenShift ecosystem | Calico Cluster Mesh / Enterprise | Cilium ClusterMesh |
| Global Service discovery | Platform dependent | Enterprise capabilities | Yes |
| Cross-cluster identity | Platform dependent | Policy/management framework | Yes |
| Central security management | OpenShift ACM/RH ecosystem | Calico Enterprise | Isovalent Enterprise |
| Hybrid/multi-cloud | Yes | Strong | Strong |

---

# 12. Multi-Network and Virtualization

OVN-Kubernetes is particularly strong in OpenShift environments that combine:

    OpenShift
       +
    OpenShift Virtualization
       +
    VMs
       +
    Pods
       +
    Multiple networks
       +
    Tenant isolation

UserDefinedNetwork provides L2/L3 segmentation models for tenants and workloads.

This makes OVN-Kubernetes a natural fit for Kubernetes + VM convergence in OpenShift.

---

# 13. Performance Characteristics

Do not claim one product is always the fastest. Actual performance depends on topology, kernel version, encapsulation, NIC offload, policy count, and Service behavior.

| Architecture | Performance characteristic |
|---|---|
| OVN-Kubernetes | Mature OVS/OpenFlow dataplane, supports hardware/DPU offload in suitable environments |
| Calico routed | Very simple efficient datapath when overlay can be avoided |
| Calico eBPF | Removes significant kube-proxy/iptables overhead |
| Calico VPP | Designed for very high-throughput networking |
| Cilium eBPF | Kernel-level policy, routing and Service LB optimizations |
| Cilium XDP | Can accelerate load balancing close to NIC ingress |

---

# 14. Operational Complexity

| Area | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Learning curve | Requires OVN/OVS/OpenFlow knowledge | Familiar to network engineers | Requires eBPF/Linux knowledge |
| Troubleshooting | ovs tools, OVN DBs, OpenFlow | ip route, BGP, Felix, calicoctl, iptables/BPF | cilium, hubble, BPF maps |
| Number of design choices | More opinionated | Very high flexibility | Moderate/high |
| OpenShift fit | Excellent | Third-party choice | Third-party choice |
| Bare-metal BGP | Possible with surrounding stack | Natural fit | Strong |
| Modern eBPF architecture | No | Optional | Core |
| Traditional network engineer familiarity | Medium | High | Medium |

---

# 15. Enterprise / Commercial Offerings

| Area | OVN-Kubernetes | Calico | Cilium |
|---|---|---|---|
| Enterprise offering | Red Hat OpenShift Networking | Calico Enterprise | Cisco/Isovalent Enterprise Platform |
| Separate paid CNI product? | No | Yes | Yes |
| Vendor | Red Hat | Tigera | Cisco / Isovalent |
| Self-managed | OpenShift manages networking | Yes | Yes |
| SaaS | Managed OpenShift offerings | Calico Cloud | Commercial/managed integrations |
| Enterprise support | Red Hat | Tigera | Cisco/Isovalent |
| Central UI | OpenShift Console | Calico Manager / Kibana-style tooling | Enterprise/Hubble tooling |
| Multi-cluster management | OpenShift ecosystem | Yes | Yes |
| Long-term flow storage | Observability backend | Enterprise | Enterprise |
| Policy recommendations | Broader platform tooling | Yes | Enterprise policy lifecycle |
| Staged policy | Different policy model | Yes | Enterprise/audit capabilities |
| Flow visualizer | Network Observability | Yes | Hubble |
| Dynamic packet capture | Platform tooling | Yes | Enterprise/eBPF tooling |
| Threat detection | Separate Red Hat security stack | Enterprise IDS/IPS/WAF capabilities | Tetragon-based Runtime Security |
| Runtime security | RHACS / broader platform | Enterprise security stack | Tetragon / Isovalent Runtime Security |
| SIEM integration | Logging/OpenShift ecosystem | Yes | SIEM Export |
| WAF | Not part of OVN itself | Enterprise | Broader Cisco security stack |
| FQDN policy | EgressFirewall | Enterprise DNS policy | Cilium policy |
| L7 policy | Not OVN's core function | Enterprise | Strong |
| Egress gateway | EgressIP/Egress services | Enterprise Egress Gateway | Enterprise add-on |
| Encryption | OpenShift OVN IPsec | WireGuard / other dataplane options | WireGuard/IPsec plus enterprise support |

---

# 16. Isovalent Enterprise Structure

Conceptually:

    Isovalent Enterprise Platform
            |
            +-- Kubernetes Networking
            |       |
            |       +-- Essentials
            |       +-- Advantage
            |
            +-- Runtime Security
            |       |
            |       +-- Essentials
            |       +-- Advantage
            |
            +-- Isovalent Load Balancer

Possible enterprise capabilities/add-ons include:

    Egress Gateway
    Encryption
    Load Balancer for Kubernetes
    SIEM Export
    Multi Cluster
    Runtime Security
    Policy lifecycle tooling

Conceptually:

    Cilium OSS
         ↓
    Networking + policy + Hubble
         ↓
    Isovalent Enterprise
         ↓
    Enterprise support
    + enhanced observability
    + policy lifecycle
    + multi-cluster
    + SIEM
    + runtime security/Tetragon
    + commercial integrations

---

# 17. Calico Enterprise Architecture

    Calico Enterprise
          |
    +-----+----------------------+------------------+
    |                            |                  |
    Networking                Security          Observability
    |                            |                  |
    Felix/eBPF                Tiered policy     Flow logs
    BGP                       DNS/FQDN           Flow Visualizer
    IPAM                      Staged policy      Packet capture
    Egress GW                 IDS/IPS/WAF        Alerts
          |
    Multi-cluster management

Calico Enterprise uses Calico networking foundations and layers on enterprise policy lifecycle, visibility, threat protection, and centralized management.

---

# 18. OpenShift / OVN Enterprise Architecture

    OpenShift
       |
    Cluster Network Operator
       |
    OVN-Kubernetes
      /       \
    OVN       OVS
     |         |
    logical    dataplane
    network
       |
    +-- NetworkPolicy
    +-- AdminNetworkPolicy
    +-- Egress controls
    +-- UserDefinedNetwork
    +-- Pod / VM networking

       +
    Network Observability Operator
       |
      eBPF
       |
    flow visibility

OVN-Kubernetes is enterprise-supported primarily through OpenShift rather than through a separate paid OVN-Kubernetes product.

---

# 19. The Three Architectural Philosophies

| OVN-Kubernetes | Calico | Cilium |
|---|---|---|
| SDN / virtual-network thinking | Router/BGP thinking | eBPF/kernel thinking |
| Logical switches | Linux routes | BPF programs |
| Logical routers | BGP | BPF maps |
| OVS | Felix | Cilium Agent |
| OpenFlow | iptables/nftables/eBPF | eBPF |
| Geneve | BGP/VXLAN/IPIP | VXLAN/Geneve/native |
| Strong OpenShift integration | Highly portable/flexible | Modern eBPF-first architecture |

---

# 20. Interview-Ready 30-Second Answer

OVN-Kubernetes is an SDN-style solution built around OVN, OVS, logical switches/routers, and commonly Geneve, and is deeply integrated into OpenShift.

Calico takes a routing-first approach and is especially strong in BGP, direct routing, and flexible policy deployment. Modern Calico also supports an eBPF dataplane.

Cilium is eBPF-first. Networking, Kubernetes Service load balancing, security, and observability are implemented deeply in the Linux kernel, with Hubble for visibility and Envoy when L7 processing is needed.

Commercially:
- OVN-Kubernetes is enterprise-supported through Red Hat OpenShift
- Calico has Calico Enterprise and Calico Cloud
- Cilium has Cisco/Isovalent Enterprise Platform

---

# 21. What Not to Say in an Interview

Do not reduce the products to:

    Calico = iptables
    Cilium = eBPF
    OVN = OVS

That is incomplete.

A better statement is:

    OVN-Kubernetes = SDN / OVS / OVN architecture

    Calico = routing-first architecture with multiple dataplanes,
             including iptables/nftables/eBPF/VPP

    Cilium = eBPF-first architecture with native Service handling,
             security, observability, and optional L7 via Envoy
