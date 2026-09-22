# AKS Cluster Build — Pre-Build, Terraform, Post-Build and DNS Q&A

> Interview refresher based on the cluster configuration and post-build scripts reviewed on 22-Sep-2026.
>
> Scope: **cluster build only**. Application onboarding is intentionally excluded.

---

## 1. End-to-End Build Flow

```text
PRE-BUILD / LANDING ZONE
        |
        v
Subscription
        |
Resource Groups
        |
Existing VNet
        |
AKS Subnet + NSG + Routes
        |
Private DNS + Identity
        |
IP Planning
(Node / Pod / Service CIDRs)
        |
Approved Kubernetes / Node Image Baseline
        |
        v
TERRAFORM BUILD
        |
        v
AKS Managed Control Plane
        |
System Node Pool
        |
User Node Pool
        |
BYOCNI + Cilium
        |
Azure Integrations
(Key Vault / Diagnostics / etc.)
        |
        v
POST-BUILD / BOOTSTRAP
        |
Validate variables
        |
Select Azure subscription
        |
Temporarily enable local accounts
        |
Cilium / kube-proxy handling
        |
Get admin kubeconfig
        |
Apply cluster RBAC
        |
Deploy/configure platform add-ons
        |
Traefik ingress
        |
Security / monitoring checks
        |
Disable local accounts
        |
        v
CLUSTER READY
```

---

# 2. Pre-Build Prerequisites

Before Terraform creates AKS, the Azure landing-zone dependencies must either already exist or be formally decided.

## 2.1 Subscription

The target Azure subscription must be identified first.

Typical prerequisites:

- Correct subscription and region
- Required Azure policies
- Resource quotas
- RBAC for the Terraform execution identity
- Enterprise networking connectivity
- Approved naming/tagging standards

The cluster-specific `.tfvars` file provides the subscription information.

---

## 2.2 Resource Groups

A typical enterprise separation is:

```text
Subscription
 |
 +-- Network Resource Group
 |     +-- VNet
 |     +-- Subnets
 |     +-- NSG / Route Table
 |
 +-- AKS Resource Group
 |     +-- AKS resource
 |
 +-- DNS Resource Group
 |     +-- Private DNS zone
 |
 +-- Identity Resource Group
       +-- User Assigned Managed Identity
```

AKS also maintains node/infrastructure resources in its AKS-managed resource group.

Examples include:

- VM Scale Sets
- NICs
- Load Balancer resources
- Managed disks
- Private IP resources

---

## 2.3 VNet and AKS Subnet

The environment consumes an existing enterprise VNet.

A dedicated AKS subnet is configured inside that VNet.

```text
Existing VNet
    |
    +-- AKS Node Subnet
            |
            +-- System node IPs
            +-- User node IPs
            +-- Internal Load Balancer connectivity
```

The reviewed configuration also includes a Key Vault service endpoint on the subnet.

---

## 2.4 IP Address Planning

Because the cluster uses **BYOCNI with Cilium**, the following ranges must be planned before creation:

- Azure VNet CIDR
- AKS node subnet CIDR
- Pod CIDR
- Service CIDR
- Kubernetes DNS service IP

Conceptually:

```text
Azure VNet / AKS Subnet
        |
        +-- Node IP addresses

Cilium Pod CIDR
        |
        +-- Pod IP addresses

Service CIDR
        |
        +-- ClusterIP addresses
              |
              +-- CoreDNS service IP
```

The ranges must not overlap with:

- Other connected VNets
- On-prem networks
- Pod CIDR
- Service CIDR
- Enterprise routed networks

---

## 2.5 NSG, Routes and Connectivity

Before building AKS, confirm the subnet has the required connectivity.

Examples:

```text
AKS Nodes
   |
   +--> Azure platform services
   +--> Container registry
   +--> Key Vault
   +--> DNS
   +--> Monitoring / logging endpoints
   +--> Required corporate services
   +--> Firewall / proxy path where applicable
```

DNS resolution alone is not enough; the network route and firewall path must also exist.

---

## 2.6 Managed Identity

The design references a **User Assigned Managed Identity**.

```text
User Assigned Managed Identity
              |
              v
             AKS
```

The identity is granted the Azure permissions required by the platform design.

Interview wording:

> We use Azure managed identity so the cluster does not need stored Azure credentials for platform-level Azure operations.

Do not state exact role assignments unless they are visible in the Terraform module or RBAC configuration.

---

## 2.7 Private DNS

Private DNS must be planned for two separate concerns:

1. **AKS API server DNS**
2. **Application / ingress DNS**

These are different paths and should not be confused.

They are covered in detail in the DNS Q&A section below.

---

## 2.8 Node Pool Design

### System Node Pool

The reviewed configuration has a dedicated system pool with:

- Autoscaling
- Minimum / maximum node counts
- Dedicated VM size
- Critical-addons-only behavior

Purpose:

```text
System Pool
   |
   +-- CoreDNS
   +-- CNI/platform components
   +-- CSI/platform components
   +-- AKS system workloads
```

### User Node Pool

A separate user node pool is used for workload capacity.

```text
User Pool
   |
   +-- Application workloads
   +-- Platform workload components as designed
```

The configuration also includes organization-specific node labels.

---

## 2.9 Approved Node Image / Snapshot

The configuration references a node pool snapshot.

Conceptually:

```text
Tested / Approved Node Image
            |
            v
     Node Pool Snapshot
            |
            v
        AKS Node Pool
```

This allows the platform team to deploy a controlled, tested node-image baseline instead of blindly consuming the latest image.

---

# 3. Terraform Build — Interview-Level Gist

The goal in an interview is **not** to reproduce the complete HCL. Explain the purpose of each layer.

## 3.1 Cluster-Specific tfvars

The `.tfvars` contains the per-cluster configuration.

Conceptually:

```hcl
subscription_id = "..."

cluster_name = "..."

virtual_network_name = "..."
virtual_network_resource_group_name = "..."

subnet = {
  address_prefixes = [...]
}

kubernetes_version = "..."

system_pool_min = ...
system_pool_max = ...

user_pool_min = ...
user_pool_max = ...

pod_cidrs     = [...]
service_cidrs = [...]
dns_service_ip = "..."

network_plugin = "byocni-cilium"

user_managed_identity_name = "..."

default_node_pool_snapshot_id = "..."

web_app_routing = null
```

Key interview point:

> The reusable Terraform module contains the implementation logic; the tfvars file contains the environment/cluster-specific values.

```text
Reusable Terraform Modules
          +
Cluster-specific tfvars
          |
          v
       AKS Cluster
```

---

## 3.2 Existing Azure Resources

The Terraform code will either use data sources or module inputs to consume existing resources such as:

- Existing VNet
- Private DNS zone
- Managed identity
- Resource groups

Conceptually:

```hcl
data "azurerm_virtual_network" "vnet" { ... }
data "azurerm_user_assigned_identity" "aks" { ... }
data "azurerm_private_dns_zone" "aks" { ... }
```

Do not memorize the syntax; understand that Terraform discovers or references pre-existing landing-zone resources.

---

## 3.3 AKS Subnet

Conceptually Terraform configures or associates:

- AKS subnet
- Address prefix
- NSG
- Route table where required
- Service endpoints where required

Example-level gist:

```hcl
resource "azurerm_subnet" "aks" {
  virtual_network_name = ...
  address_prefixes     = ...
}
```

---

## 3.4 AKS Cluster Resource

At the center is the AKS managed cluster resource.

Conceptually:

```hcl
resource "azurerm_kubernetes_cluster" "aks" {
  name               = var.cluster_name
  kubernetes_version = var.kubernetes_version

  identity {
    type = "UserAssigned"
  }

  default_node_pool {
    vm_size = ...
  }

  network_profile {
    ...
  }
}
```

Interview explanation:

> Terraform creates/configures the AKS managed control plane, default system node pool, managed identity, private networking and cluster networking profile.

---

## 3.5 Additional Node Pools

Additional pools are conceptually represented using AKS node pool resources.

```text
AKS
 |
 +-- System Node Pool
 |
 +-- User Node Pool
 |
 +-- Optional specialized pools
```

The reviewed configuration also supports Portworx-specific pools, but Portworx is disabled for the cluster being discussed.

---

# 4. Networking: BYOCNI + Cilium

The environment uses:

```text
network_plugin = byocni-cilium
```

Therefore it is not a simple default Azure CNI design.

Conceptually:

```text
             AKS
              |
      +-------+-------+
      |               |
    Node            Node
      |               |
  Cilium eBPF     Cilium eBPF
      |               |
    Pods             Pods
```

Service forwarding is intended to be handled by Cilium eBPF.

```text
Kubernetes Service
        |
        v
   Cilium eBPF
        |
        v
       Pod
```

rather than relying on the traditional:

```text
Kubernetes Service
        |
        v
     kube-proxy
        |
   iptables / IPVS
        |
        v
       Pod
```

---

# 5. Ingress: Traefik, not AKS Web App Routing

The reviewed tfvars contains:

```hcl
web_app_routing = null
```

This is intentional.

The cluster uses **Traefik** as the ingress controller rather than AKS Web App Routing/nginx.

Therefore the generic post-build block for the AKS Web App Routing add-on is skipped.

The intended application path is:

```text
Client
  |
DNS
  |
Azure Internal Load Balancer
  |
Traefik Service
  |
Traefik Ingress Controller
  |
Ingress / IngressRoute
  |
Kubernetes Service
  |
Pods
```

---

# 6. Post-Build / Bootstrap Steps

After Terraform creates the Azure and AKS infrastructure, the post-build script completes cluster bootstrap.

## Step 1 — Validate Inputs

The script loads values such as:

- Subscription ID
- Cluster name
- AKS resource group
- Network plugin
- Web App Routing configuration

It fails early when required values are missing.

---

## Step 2 — Select the Correct Azure Subscription

Conceptually:

```bash
az account set --subscription <subscription>
```

Purpose:

> Ensure every subsequent Azure CLI command targets the intended subscription.

---

## Step 3 — Temporarily Enable Local AKS Accounts

The script checks whether local accounts are disabled.

If disabled, it temporarily enables them so the bootstrap process can obtain admin credentials.

```text
Local accounts disabled
        |
        v
Temporarily enable
        |
        v
Bootstrap actions
        |
        v
Disable again at the end
```

---

## Step 4 — Cilium / kube-proxy Handling

The script checks whether:

```text
NETWORK_PLUGIN == byocni-cilium
```

If yes, it checks the AKS kube-proxy state.

Because the environment is designed for Cilium kube-proxy replacement, AKS-managed kube-proxy is disabled when necessary.

```text
BYOCNI + Cilium
      |
      v
Check kube-proxy
      |
 +----+----+
 |         |
enabled  disabled
 |         |
 v         |
disable    |
 +---------+
      |
      v
Continue bootstrap
```

The script may require the appropriate Azure CLI AKS extension for the kube-proxy configuration operation.

Important limitation:

> The screenshots confirm the kube-proxy handling, but they do not prove exactly where the Cilium Helm installation itself happens. It may be in Terraform, another bootstrap script or a separate pipeline stage.

---

## Step 5 — Obtain Admin kubeconfig

Conceptually:

```bash
az aks get-credentials --admin --overwrite-existing
```

Flow:

```text
Azure CLI
   |
   v
AKS API
   |
   v
Admin kubeconfig
   |
   v
kubectl
```

---

## Step 6 — Apply Enterprise Cluster RBAC

The post-build script applies the cluster-admin RBAC manifest.

Conceptually:

```text
Enterprise Admin Identity / Group
              |
              v
       Kubernetes RBAC
              |
              v
      ClusterRoleBinding
```

Remember:

```text
Authentication = Who are you?
Authorization  = What can you do?
```

---

## Step 7 — Skip AKS Web App Routing

Because:

```hcl
web_app_routing = null
```

the nginx / app-routing-system block is skipped.

This is expected, because Traefik is used instead.

---

## Step 8 — Deploy / Configure Traefik

Traefik is the ingress layer for this design.

Conceptually its platform configuration needs:

- Traefik workload
- Traefik RBAC
- IngressClass / CRDs as required
- Service of type LoadBalancer
- Internal Azure Load Balancer configuration
- TLS / certificate configuration
- Application wildcard DNS

```text
Azure Internal Load Balancer
            |
            v
      Traefik Service
            |
            v
        Traefik Pods
            |
            v
     Ingress / IngressRoute
            |
            v
       App Services
```

---

## Step 9 — Platform Add-ons

Depending on the environment, the bootstrap process may also deploy/configure:

- Monitoring agents
- Logging
- Security agents
- Key Vault integration
- CSI drivers
- Policies
- Storage components
- Portworx where enabled

Do not state a component is enabled unless it is confirmed in the cluster configuration.

---

## Step 10 — Disable Local Accounts

After bootstrap completes:

```bash
az aks update --disable-local-accounts
```

The cluster returns to its secured normal operating state.

```text
Temporary bootstrap admin
        |
        v
RBAC / platform bootstrap complete
        |
        v
Local accounts disabled
```

---

# 7. DNS — API and Application Paths

This is one of the most important interview topics.

## 7.1 Two Different DNS Concerns

```text
                 PRIVATE DNS
                     |
          +----------+----------+
          |                     |
     AKS API DNS            Application DNS
          |                     |
          v                     v
Private AKS API          *.apps wildcard
   endpoint                    |
                               v
                     Traefik Internal LB
```

These must be explained separately.

---

# 8. Q&A — How Does the AKS API DNS Work?

## Q1. Do we manually create the AKS API A record?

Usually, **no**.

For a private AKS cluster, the AKS provisioning process creates/manages the API-server DNS mapping in the configured private DNS zone.

Conceptually:

```text
AKS cluster creation
       |
       v
Private API endpoint created
       |
       v
Private DNS zone configured
       |
       v
API-server DNS record created
       |
       v
API FQDN --> Private endpoint IP
```

So when an administrator runs:

```bash
kubectl get nodes
```

the path is:

```text
kubectl
   |
   v
AKS API FQDN
   |
   v
Private DNS lookup
   |
   v
Private API endpoint IP
   |
   v
AKS API Server :443
```

---

## Q2. Who owns the API DNS record?

The key distinction is:

```text
AKS API DNS
    -> Created / managed as part of private AKS provisioning

Application wildcard DNS
    -> Created by the platform/DNS process
```

If an existing private DNS zone is supplied to AKS, AKS uses that zone and creates the required private API record, assuming the configured identity has the required permissions.

---

## Q3. What does Private DNS actually do?

DNS only answers:

> What IP should this name resolve to?

It does **not** provide the network route.

You still need:

- VNet connectivity
- Peering / ExpressRoute / VPN where applicable
- Routes
- Firewall/NSG rules

For example:

```text
Corporate Workstation
       |
       | DNS resolves AKS API name
       v
Private API IP
       |
       | Network route must exist
       v
AKS API Server
```

---

# 9. Q&A — How Does `*.apps` DNS Work?

## Q4. Is `*.apps` automatically created by AKS?

No.

`*.apps` is an application-domain convention. AKS itself does not automatically provide an OpenShift-style `*.apps` domain.

Because this cluster uses Traefik, the platform creates an application wildcard DNS entry pointing to the Traefik ingress load-balancer IP.

Example:

```text
*.apps.cluster01.company.internal
             |
             A
             |
             v
        10.20.30.40
```

where `10.20.30.40` represents the private frontend IP of the Azure Internal Load Balancer used by Traefik.

---

## Q5. Is the wildcard record created by the DNS team?

In a centrally managed enterprise DNS model, **yes**.

A DNS/platform team can create:

```text
*.apps.cluster01.company.internal
        -> Traefik Internal LB private IP
```

That means:

```text
orders.apps.cluster01.company.internal
payments.apps.cluster01.company.internal
billing.apps.cluster01.company.internal
```

can all resolve to the same ingress IP.

The ingress controller then determines the destination service from the requested host.

---

## Q6. Why can all applications use one IP?

Because DNS gets traffic only to the ingress endpoint.

Traefik performs Layer-7 host-based routing.

```text
orders.apps.cluster01.company.internal
                   |
                   v
              DNS lookup
                   |
                   v
        Traefik ILB IP
                   |
                   v
               Traefik
                   |
        Host = orders....
                   |
                   v
         orders-service
                   |
                   v
            orders pods
```

For another host:

```text
payments.apps.cluster01.company.internal
                    |
                    v
                 same IP
                    |
                    v
                 Traefik
                    |
         Host = payments....
                    |
                    v
          payments-service
```

One frontend IP can therefore serve many application hostnames.

---

## Q7. Should the Traefik ILB IP be static?

Preferably yes.

A predictable/reserved private IP makes the DNS mapping stable.

```text
Reserved Private IP
        |
        +--> Traefik Internal LB
        |
        +--> *.apps DNS record
```

Otherwise a dynamically changing load-balancer IP could break the DNS mapping.

---

# 10. API DNS vs Application DNS — Quick Comparison

| Item | AKS API | Applications |
|---|---|---|
| Purpose | Kubernetes control-plane access | Application traffic |
| Consumer | kubectl, CI/CD, platform automation | Users / services / clients |
| DNS name | AKS private API FQDN | `*.apps.<cluster-domain>` or chosen app domain |
| DNS record creation | AKS/private-cluster provisioning | DNS/platform process |
| Resolves to | AKS private API endpoint IP | Traefik Azure ILB private IP |
| Next hop | AKS managed API server | Traefik |
| Port | Usually 443 | Usually 80/443 |
| Routing inside cluster | Kubernetes API | Host/path routing through Traefik |

Easy memory:

```text
API DNS
   -> Private AKS API endpoint
   -> Control plane

*.apps DNS
   -> Azure Internal LB
   -> Traefik
   -> Service
   -> Pods
```

---

# 11. Complete Traffic Pictures

## AKS API Path

```text
Admin / CI / Argo / kubectl
           |
           v
      API FQDN
           |
           v
     Private DNS
           |
           v
Private AKS API Endpoint
           |
           v
Microsoft-managed AKS API Server
```

## Application Path

```text
User / Application Client
           |
           v
orders.apps.cluster01.company.internal
           |
           v
     Private DNS
           |
           v
Traefik Internal LB Private IP
           |
           v
Azure Internal Load Balancer
           |
           v
Traefik Service / Traefik Pods
           |
           v
Ingress / IngressRoute
           |
           v
Kubernetes Service
           |
           v
Application Pods
```

---

# 12. Interview Answer — AKS Build in ~90 Seconds

> Before we build AKS, the Azure landing-zone dependencies must be ready. That includes the subscription, resource groups, existing VNet, AKS subnet, NSG and routing, private DNS, user-assigned managed identity, IP planning for nodes, pods and services, and our approved Kubernetes/node-image baseline.
>
> The cluster-specific values are stored in tfvars, while reusable Terraform modules contain the common implementation. Terraform provisions the AKS managed control plane, system node pool, user node pool, identity and private networking integration.
>
> Our networking uses BYOCNI with Cilium, with separate pod and service CIDRs. Cilium is used as the dataplane and the post-build flow verifies/disables AKS-managed kube-proxy where kube-proxy replacement is required.
>
> Once Terraform finishes, the post-build script validates the cluster settings, selects the Azure subscription, temporarily enables local admin access, obtains admin kubeconfig and applies enterprise Kubernetes RBAC. AKS Web App Routing is intentionally null because we use Traefik rather than the nginx Web App Routing add-on. Traefik is exposed through an internal Azure Load Balancer and the application wildcard DNS points to that private load-balancer IP.
>
> Finally, monitoring/security/platform components are configured as required and local AKS accounts are disabled again.

---

# 13. Fast Interview Q&A

### Q: Why separate system and user node pools?
System components and application workloads have different lifecycle, sizing and scaling requirements. Dedicated pools improve isolation and operational control.

### Q: Why use autoscaling?
To scale node capacity based on unschedulable pod demand while enforcing minimum and maximum capacity boundaries.

### Q: Why use a node snapshot?
To use a tested/approved node image baseline instead of blindly taking the newest available image.

### Q: Why BYOCNI with Cilium?
It allows the platform to use Cilium for pod networking and eBPF-based dataplane capabilities rather than relying entirely on the standard AKS networking dataplane.

### Q: Why disable kube-proxy?
When Cilium is configured for kube-proxy replacement, Cilium eBPF performs Kubernetes service forwarding, making the traditional kube-proxy path unnecessary.

### Q: Why is `web_app_routing = null`?
Because the cluster uses Traefik as the ingress controller instead of the AKS Web App Routing/nginx add-on.

### Q: What does application DNS point to?
The wildcard application DNS record points to the private frontend IP of the Azure Internal Load Balancer used by Traefik.

### Q: What does the AKS API DNS point to?
The AKS private API FQDN resolves through private DNS to the private AKS API endpoint.

### Q: Does the DNS team manually create the AKS API record?
Normally no. The API DNS record is created/managed as part of private AKS provisioning in the configured private DNS zone.

### Q: Does DNS provide connectivity?
No. DNS resolves a name to an IP; routing, peering, VPN/ExpressRoute, NSGs and firewalls provide the actual network path.

### Q: Why temporarily enable local accounts during bootstrap?
The reviewed script requires admin kubeconfig for bootstrap operations. It temporarily enables local accounts, performs required configuration, and disables them again when bootstrap is complete.

### Q: What happens after Terraform?
The post-build/bootstrap stage completes network-specific configuration, RBAC, ingress/platform add-ons, monitoring/security checks and returns the cluster to its intended secured state.

---

# 14. One Diagram to Remember

```text
                         PRE-BUILD
                            |
       Subscription / RG / VNet / Subnet / DNS / Identity
                            |
                  CIDR + Node Image Planning
                            |
                            v
                        TERRAFORM
                            |
                        AKS Cluster
                            |
              +-------------+-------------+
              |                           |
       System Node Pool             User Node Pool
              |                           |
              +-------------+-------------+
                            |
                     BYOCNI / Cilium
                            |
                            v
                        POST-BUILD
                            |
               kube-proxy / Cilium checks
                            |
                    kubeconfig + RBAC
                            |
                         Traefik
                            |
                  Azure Internal LB
                            |
                    *.apps wildcard
                            |
                            v
                       CLUSTER READY


CONTROL PLANE DNS:
API FQDN -> Private DNS -> AKS Private API Endpoint

APPLICATION DNS:
*.apps -> Private DNS -> Traefik ILB -> Traefik -> Service -> Pods
```
