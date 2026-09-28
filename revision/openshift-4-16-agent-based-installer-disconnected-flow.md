# Deploying OpenShift 4.16 via Agent-Based Installer — Disconnected Flow

> **Scope:** Deploy a new OpenShift Container Platform 4.16 cluster in a disconnected environment using the **Agent-Based Installer**.
>
> **Assumptions:** The mirror registry already exists, OpenShift release images are mirrored internally, and DNS/load-balancing/network prerequisites are available.
>
> **Not covered here:** Building the mirror registry, repository mirroring details, Squid/proxy simulation, or Operator lifecycle.

Related notes:

- `revision/openshift-disconnected-operator-upgrade-mirror-registry-runbook.md`
- `revision/openshift-and-operator-repository-mirroring-flow.md`

---

# 1. End-to-End Flow

```text
Mirror registry already available
        |
OpenShift 4.16 release already mirrored
        |
Prepare nodes + DNS + load balancer + NTP
        |
Extract openshift-install matching mirrored release
        |
Create install-config.yaml
        |
Create agent-config.yaml
        |
Generate Agent ISO
        |
Boot all masters/workers with same ISO
        |
Rendezvous master runs Assisted Service/bootstrap
        |
Agents register + hosts validated
        |
RHCOS written to disks
        |
Control plane forms
        |
Nodes reboot into permanent cluster
        |
Wait for install-complete
        |
Validate OpenShift
```

The Medium example uses KVM virtual machines to simulate on-premises disconnected infrastructure. The reusable design is the same for supported physical/virtual environments: the Agent ISO boots each host and one control-plane node becomes the rendezvous/bootstrap host.

---

# 2. What the Agent-Based Installer Does

The Agent-Based Installer provides an Assisted-Installer-style workflow locally, which makes it well suited to disconnected environments.

You generate a bootable ISO with:

```bash
openshift-install agent create image --dir <install-dir>
```

One control-plane host becomes the **rendezvous host**. It initially runs the Assisted Service/bootstrap function, coordinates installation, then reboots and becomes a normal control-plane node.

```text
install-config.yaml + agent-config.yaml
                 |
                 v
       openshift-install
                 |
                 v
          Agent boot ISO
                 |
     +-----------+-----------+
     |           |           |
 master-0    master-1    master-2
     |
     +-- rendezvous / temporary bootstrap
                 |
                 v
         Permanent OCP cluster
```

---

# 3. Example HA Topology

```text
Cluster name : ocp416
Base domain  : example.com

master-0     : 10.20.30.11
master-1     : 10.20.30.12
master-2     : 10.20.30.13
worker-0     : 10.20.30.21
worker-1     : 10.20.30.22

API VIP/LB   : 10.20.30.5
Ingress VIP  : 10.20.30.6

Mirror       : mirror.example.com:8443
DNS          : 10.20.30.2
Gateway      : 10.20.30.1
```

This is only an example addressing plan; use the organization's actual IPAM/DNS/LB design.

---

# 4. Prepare DNS

Typical DNS records:

```text
api.ocp416.example.com       -> 10.20.30.5
api-int.ocp416.example.com   -> 10.20.30.5
*.apps.ocp416.example.com    -> 10.20.30.6
```

Node records are also commonly created:

```text
master-0.ocp416.example.com -> 10.20.30.11
master-1.ocp416.example.com -> 10.20.30.12
master-2.ocp416.example.com -> 10.20.30.13
worker-0.ocp416.example.com -> 10.20.30.21
worker-1.ocp416.example.com -> 10.20.30.22
```

Validate:

```bash
dig api.ocp416.example.com
dig api-int.ocp416.example.com
dig test.apps.ocp416.example.com
dig mirror.example.com
```

Use corporate/internal DNS in production rather than relying on `/etc/hosts`.

---

# 5. Prepare Load Balancing

For user-provided/on-premises infrastructure, ensure cluster endpoints reach the correct backends.

```text
api.ocp416.example.com
        |
        v
      API LB
        |
        +--> master-0
        +--> master-1
        +--> master-2

*.apps.ocp416.example.com
        |
        v
    Ingress LB
        |
        +--> nodes running ingress/router pods
```

Important ports to plan for include:

```text
6443   Kubernetes API
22623  Machine Config Server during bootstrap/provisioning
80     HTTP ingress
443    HTTPS ingress
```

The exact VIP/backend configuration varies by platform and topology.

---

# 6. Verify Mirror Registry Prerequisites

Before generating installation media:

```bash
getent hosts mirror.example.com
curl https://mirror.example.com:8443/v2/
podman login mirror.example.com:8443
```

Confirm:

```text
DNS              OK
TLS              OK
Registry auth    OK
OCP 4.16 images  present
```

---

# 7. Use the Installer from the Mirrored Release

The installer should match the exact OpenShift release being installed.

Example:

```bash
oc adm release extract \
  -a <pull-secret-json> \
  --command=openshift-install \
  "mirror.example.com:8443/ocp4/openshift4:<4.16.x-x86_64>"
```

Then:

```bash
chmod +x openshift-install
./openshift-install version
```

This avoids using an installer binary from a different release.

---

# 8. Create the Installation Directory

```bash
mkdir -p ~/ocp416-agent
cd ~/ocp416-agent
```

Core inputs:

```text
install-config.yaml
agent-config.yaml
```

---

# 9. Create install-config.yaml

This defines **cluster-level settings**.

Example for a user-provided HA environment:

```yaml
apiVersion: v1
baseDomain: example.com

metadata:
  name: ocp416

controlPlane:
  architecture: amd64
  hyperthreading: Enabled
  name: master
  replicas: 3

compute:
- architecture: amd64
  hyperthreading: Enabled
  name: worker
  replicas: 2

networking:
  networkType: OVNKubernetes
  machineNetwork:
  - cidr: 10.20.30.0/24
  clusterNetwork:
  - cidr: 10.128.0.0/14
    hostPrefix: 23
  serviceNetwork:
  - 172.30.0.0/16

platform:
  none: {}

pullSecret: '<combined-pull-secret>'

sshKey: 'ssh-rsa AAAA...'

additionalTrustBundle: |
  -----BEGIN CERTIFICATE-----
  <mirror-registry-CA>
  -----END CERTIFICATE-----

imageContentSources:
- mirrors:
  - mirror.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - mirror.example.com:8443/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
```

**Important:** use the mirror mappings generated by your actual mirroring operation. Do not copy generic repository paths blindly.

For the disconnected install, remember:

```text
additionalTrustBundle
  = trust the mirror registry CA

imageContentSources
  = redirect OpenShift release image pulls to the internal mirror
```

For OCP 4.16, Red Hat documentation still shows `imageContentSources` for disconnected installation input; ICSP itself is deprecated, so later OpenShift releases may use newer mirror-policy mechanisms.

---

# 10. Pull Secret

The installer pull secret must include whatever authentication is required for the internal mirror registry.

```text
Red Hat pull secret
        +
Mirror-registry credentials
        |
        v
Combined pull secret
        |
        v
install-config.yaml
```

Never commit a real pull secret to Git.

---

# 11. Create agent-config.yaml

This defines **host-level configuration** such as:

- hostname
- master/worker role
- NIC/MAC
- static IP
- DNS
- default route
- installation disk
- rendezvous IP

Example:

```yaml
apiVersion: v1beta1
kind: AgentConfig

metadata:
  name: ocp416

rendezvousIP: 10.20.30.11

hosts:
- hostname: master-0
  role: master
  interfaces:
  - name: ens192
    macAddress: 52:54:00:00:00:11
  rootDeviceHints:
    deviceName: /dev/sda
  networkConfig:
    interfaces:
    - name: ens192
      type: ethernet
      state: up
      mac-address: 52:54:00:00:00:11
      ipv4:
        enabled: true
        dhcp: false
        address:
        - ip: 10.20.30.11
          prefix-length: 24
    dns-resolver:
      config:
        server:
        - 10.20.30.2
    routes:
      config:
      - destination: 0.0.0.0/0
        next-hop-address: 10.20.30.1
        next-hop-interface: ens192
        table-id: 254

- hostname: master-1
  role: master
  interfaces:
  - name: ens192
    macAddress: 52:54:00:00:00:12

- hostname: master-2
  role: master
  interfaces:
  - name: ens192
    macAddress: 52:54:00:00:00:13

- hostname: worker-0
  role: worker
  interfaces:
  - name: ens192
    macAddress: 52:54:00:00:00:21

- hostname: worker-1
  role: worker
  interfaces:
  - name: ens192
    macAddress: 52:54:00:00:00:22
```

For a real static-address deployment, normally provide the complete `networkConfig` for each node.

The MAC address is important because the Agent-Based Installer uses it to associate the host configuration with the correct machine.

---

# 12. Rendezvous Host

Example:

```yaml
rendezvousIP: 10.20.30.11
```

That address must belong to a control-plane host.

```text
master-0 / 10.20.30.11
        |
        +-- runs Assisted Service
        +-- coordinates discovery
        +-- performs temporary bootstrap role
        +-- installs/reboots
        +-- joins as normal master
```

This removes the need to manually build and later delete a separate bootstrap VM as in a traditional UPI workflow.

---

# 13. Preflight Checks

Before creating the ISO, verify:

```text
Cluster name matches in both YAML files
Machine CIDR matches node subnet
Node IPs are unique
MAC addresses are correct
Master/worker roles are explicit
Rendezvous IP belongs to a master
DNS resolves API, apps and mirror
Gateway/routing works
NTP/time is correct
Mirror registry CA is correct
Pull secret contains mirror authentication
Mirrored release exists
Root device hints point to the correct disks
```

Optional YAML validation:

```bash
yamllint install-config.yaml
yamllint agent-config.yaml
```

---

# 14. Generate the Agent ISO

```bash
openshift-install agent create image \
  --dir ~/ocp416-agent
```

Typical result:

```text
agent.x86_64.iso
```

```text
install-config.yaml
       +
agent-config.yaml
       |
       v
openshift-install agent create image
       |
       v
agent.x86_64.iso
```

---

# 15. Boot Every Target Host

Attach the Agent ISO to all masters and workers.

Examples:

```text
KVM        -> attach ISO to VM
VMware     -> mount ISO as virtual CD/DVD
Bare metal -> BMC virtual media / physical media
PXE        -> alternatively generate/use Agent PXE assets
```

Every host boots the same image.

---

# 16. Agent Discovery

After boot:

```text
Agent ISO boots
      |
Network config applied
      |
Discovery agent starts
      |
Host reaches rendezvousIP
      |
Host registers with Assisted Service
      |
Hardware/network validations run
```

Common early failures:

- wrong NIC/MAC
- bad IP/gateway
- DNS failure
- VLAN mismatch
- registry TLS failure
- bad pull-secret authentication
- missing release image
- wrong installation disk

---

# 17. Internal Installation Sequence

```text
All required hosts register
        |
Assisted Service validates inventory
        |
Rendezvous host begins bootstrap
        |
RHCOS written to target disks
        |
Control-plane nodes reboot
        |
etcd/API/control plane form
        |
Workers reboot and join
        |
Rendezvous host reboots
        |
Temporary bootstrap role disappears
        |
Permanent OpenShift cluster
```

---

# 18. Monitor Bootstrap

```bash
openshift-install \
  --dir ~/ocp416-agent \
  agent wait-for bootstrap-complete \
  --log-level=info
```

Expected:

```text
INFO cluster bootstrap is complete
```

For troubleshooting:

```bash
openshift-install \
  --dir ~/ocp416-agent \
  agent wait-for bootstrap-complete \
  --log-level=debug
```

---

# 19. Wait for Installation Completion

```bash
openshift-install \
  --dir ~/ocp416-agent \
  agent wait-for install-complete
```

Expected:

```text
INFO Cluster is installed
INFO Install complete!
```

The installation directory contains credentials such as:

```text
auth/kubeconfig
auth/kubeadmin-password
```

---

# 20. Access and Validate

```bash
export KUBECONFIG=~/ocp416-agent/auth/kubeconfig

oc get nodes
oc get clusterversion
oc get clusteroperators
```

Validate:

```text
All expected nodes are Ready
ClusterVersion is Available
ClusterOperators are Available
No unexpected Degraded=True
API resolves/works
Console route resolves/works
Ingress wildcard DNS works
Internal registry pulls work
```

Console example:

```text
https://console-openshift-console.apps.ocp416.example.com
```

---

# 21. Validate the Disconnected Design

Look for image-pull problems:

```bash
oc get pods -A
oc get events -A | grep -i -E 'pull|image|registry|x509|unauthorized'
```

You should not see unexpected:

```text
ImagePullBackOff
ErrImagePull
x509 errors
unauthorized errors
runtime dependency on inaccessible public registries
```

---

# 22. Troubleshooting: Release Image Pull Failure

Symptoms:

```text
Release image URL pull check failed
x509: certificate signed by unknown authority
unauthorized
manifest unknown
```

Check:

1. `mirror.example.com` resolves.
2. Registry TLS SAN matches its DNS hostname.
3. Registry CA is present in `additionalTrustBundle`.
4. Combined pull secret contains mirror credentials.
5. Correct 4.16 release exists in the mirror registry.
6. `imageContentSources` uses the actual mirrored repository mappings.
7. Firewall permits node-to-registry traffic.

---

# 23. Troubleshooting: Host Does Not Register

Check:

```bash
ip addr
ip route
cat /etc/resolv.conf
```

Likely causes:

- MAC address in `agent-config.yaml` does not match
- incorrect static IP
- wrong gateway
- VLAN/network mismatch
- DNS cannot resolve required names
- node cannot reach rendezvous host

---

# 24. Troubleshooting: Bootstrap Stuck

Run:

```bash
openshift-install \
  --dir ~/ocp416-agent \
  agent wait-for bootstrap-complete \
  --log-level=debug
```

Gather diagnostics from the rendezvous host:

```bash
ssh core@10.20.30.11 \
  agent-gather -O > agent-gather.tar.xz
```

Investigate:

```text
DNS
API/LB reachability
etcd/control-plane connectivity
mirror pulls
time synchronization
RHCOS disk installation
host reachability
```

---

# 25. Troubleshooting: Bootstrap Completes but Install Fails

```bash
export KUBECONFIG=~/ocp416-agent/auth/kubeconfig

oc get nodes
oc get co
oc get pods -A
oc adm must-gather
```

---

# 26. Traditional UPI vs Agent-Based Installer

Traditional UPI:

```text
Create bootstrap VM
 -> generate/serve Ignition
 -> boot bootstrap
 -> boot masters/workers
 -> bootstrap completes
 -> remove bootstrap VM
```

Agent-Based Installer:

```text
install-config + agent-config
 -> generate Agent ISO
 -> boot all hosts
 -> one master becomes rendezvous/bootstrap
 -> installer orchestrates provisioning
 -> rendezvous host becomes normal master
```

---

# 27. Key Files for Interview

```text
install-config.yaml
    = cluster-level configuration

agent-config.yaml
    = host inventory/network/disk/roles

agent.x86_64.iso
    = bootable discovery/install image

rendezvousIP
    = master IP that temporarily coordinates bootstrap

auth/kubeconfig
    = admin access after installation
```

---

# 28. 15-Step Interview Answer

1. Mirror the required OpenShift 4.16 release into the internal registry.
2. Verify mirror-registry DNS, TLS, CA and credentials.
3. Prepare target control-plane and worker hosts.
4. Configure API, API-internal and wildcard apps DNS.
5. Prepare API/MachineConfig/Ingress load-balancing paths as required.
6. Extract `openshift-install` from the exact mirrored release.
7. Create `install-config.yaml` with cluster/network settings.
8. Add the mirror CA and mirror mappings to `install-config.yaml`.
9. Create `agent-config.yaml` with roles, MACs, IPs, DNS, routes and disk hints.
10. Select a master node as the rendezvous host.
11. Generate the Agent ISO.
12. Boot every master and worker using that ISO.
13. Agents register with the Assisted Service and the rendezvous host bootstraps the cluster.
14. Track `bootstrap-complete` and `install-complete`.
15. Export kubeconfig and validate nodes, ClusterVersion, ClusterOperators, routes and disconnected image pulls.

---

# 29. 30-Second Interview Pitch

> In a disconnected Agent-Based OpenShift installation, I first ensure the OpenShift release is mirrored into our internal registry and validate DNS, TLS and registry authentication. I prepare the API and application DNS/load-balancing paths, then extract the openshift-install binary from the exact mirrored release. I use install-config.yaml for cluster-level settings including the mirror CA and mirror mappings, and agent-config.yaml for node-level configuration such as roles, MAC addresses, static IPs, routes, DNS and installation disks. One master is selected as the rendezvous host. I generate the Agent ISO and boot every cluster host from it. The agents register with the Assisted Service on the rendezvous host, RHCOS is installed, the control plane bootstraps, and the nodes reboot into the permanent cluster. I then wait for install-complete and validate the cluster and internal image pulls.

---

# 30. Quick Memory Flow

```text
Mirror OCP Release
 -> DNS/LB
 -> Matching openshift-install
 -> install-config.yaml
 -> agent-config.yaml
 -> Agent ISO
 -> Boot nodes
 -> Rendezvous/Assisted Service
 -> RHCOS
 -> Bootstrap
 -> Nodes reboot
 -> OpenShift
 -> Validate
```

---

# 31. Relationship to Our Other Runbooks

```text
Mirror Registry Setup
        |
        v
Mirror OpenShift + Operator Repositories
        |
        v
THIS FLOW: Agent-Based OpenShift Deployment
        |
        v
Operator installation/upgrade lifecycle
```
