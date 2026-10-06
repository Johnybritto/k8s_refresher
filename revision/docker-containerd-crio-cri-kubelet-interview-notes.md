# Docker vs containerd vs CRI-O — CRI Relationship with kubelet

## Core idea

The easiest way to remember it:

> **Kubelet does not directly understand Docker, containerd, or CRI-O. It speaks CRI. containerd and CRI-O implement CRI.**

```text
                 Kubernetes Control Plane
                         |
                         | PodSpec
                         v
                     kubelet
                         |
                         | CRI
                         | gRPC
                         v
              +----------------------+
              | Container Runtime    |
              +----------------------+
                 |              |
            containerd         CRI-O
                 |              |
                 v              v
               runc           runc
                 |              |
                 v              v
            Linux Kernel / Containers
```

---

## What each component means

| Component | What it is | Kubernetes relationship |
|---|---|---|
| **Docker** | Full container platform: build, pull, run, CLI, daemon, etc. | Kubernetes no longer talks directly to Docker |
| **containerd** | Lightweight container runtime | Implements **CRI**, so kubelet can use it directly |
| **CRI-O** | Runtime built specifically for Kubernetes | Implements **CRI** directly |
| **CRI** | Container Runtime Interface | Standard API between kubelet and runtime |
| **runc** | Low-level OCI runtime | Creates the actual container process, namespaces and cgroups |

**Important:** CRI is **not** a runtime. It is the interface/API contract between kubelet and the container runtime.

---

## kubelet and CRI

```text
Kubelet
   |
   | "Create this pod"
   | "Start this container"
   | "Stop this container"
   | "Give me container status"
   |
   | CRI
   v
containerd / CRI-O
```

CRI is a gRPC API and conceptually exposes two major services:

```text
CRI
├── RuntimeService
│     ├── RunPodSandbox
│     ├── CreateContainer
│     ├── StartContainer
│     ├── StopContainer
│     └── RemoveContainer
│
└── ImageService
      ├── PullImage
      ├── RemoveImage
      └── ImageStatus
```

---

## What happens when Kubernetes starts a Pod?

Example: an nginx Pod is created.

```text
kubectl apply
      |
      v
API Server
      |
      v
Scheduler selects worker-node-1
      |
      v
kubelet
      |
      | CRI
      v
containerd
      |
      | OCI
      v
runc
      |
      v
Linux namespaces + cgroups
      |
      v
nginx process
```

The kubelet does not create the Linux container process itself. It asks the configured CRI-compatible runtime to do it.

---

## Where Docker fits

Historically Kubernetes supported Docker through **dockershim**:

```text
kubelet
   |
   | CRI
   v
dockershim
   |
   v
Docker Engine
   |
   v
containerd
   |
   v
runc
```

Dockershim was removed from Kubernetes starting with **Kubernetes 1.24**.

Modern Kubernetes normally uses:

```text
kubelet
   |
   | CRI
   v
containerd
   |
   v
runc
```

or:

```text
kubelet
   |
   | CRI
   v
CRI-O
   |
   v
runc
```

Docker can still be integrated through **cri-dockerd**, but it is not the normal runtime choice for modern Kubernetes worker nodes.

---

## containerd vs CRI-O

### containerd

- General-purpose container runtime.
- Supports CRI for Kubernetes.
- Commonly used in Kubernetes distributions and managed Kubernetes services.
- Uses OCI-compatible runtimes such as `runc` to create containers.

### CRI-O

- Runtime designed specifically for Kubernetes.
- Implements CRI directly.
- Uses OCI runtimes such as `runc` or `crun`.
- Commonly encountered in OpenShift environments.

---

## CRI vs OCI

Do not confuse these two.

```text
Kubernetes
    |
    | CRI
    v
containerd / CRI-O
    |
    | OCI Runtime Specification
    v
runc / crun
    |
    v
Linux Kernel
```

Remember:

> **CRI = Kubernetes ↔ container runtime**

> **OCI = container runtime ↔ low-level container execution standard**

---

## Interview answer: How does kubelet start a container?

> Once a Pod is assigned to a node, kubelet communicates with the configured container runtime through the Container Runtime Interface, or CRI. A CRI-compatible runtime such as containerd or CRI-O pulls the required image, creates the Pod sandbox and starts the containers. The runtime then uses an OCI runtime such as runc to create the actual Linux namespaces, cgroups and container processes.

---

## One-line memory trick

```text
kubelet → CRI → containerd/CRI-O → OCI → runc → Linux
```

That is the main relationship to remember for interviews.
