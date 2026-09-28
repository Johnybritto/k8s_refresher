# OpenShift Air-Gapped / Disconnected Environment — Revision Notes

## 12-Step Flow

1. **Prepare an internal registry** — for example, Red Hat Quay/private enterprise registry.
2. **Prepare a connected mirror host** — install `oc` and `oc-mirror`.
3. **Authenticate** the mirror host to Red Hat registries and the internal registry.
4. **Define an ImageSetConfiguration** containing the required OpenShift release, operators, and other content.
5. **Mirror OpenShift release images** using `oc-mirror`.
6. **Mirror required operator catalogs/images** — for example GPU, storage, and other required operators.
7. **Mirror application/AI images** — application images, CUDA/PyTorch/inference images where required.
8. For a fully disconnected site, **transfer the mirrored content across the air gap** using the approved transfer mechanism.
9. **Import/push the content into the internal registry**.
10. **Configure OpenShift mirror policies and operator catalog sources** to use the internal registry.
11. **Install or upgrade OpenShift/operators** using the mirrored content only.
12. **Validate** nodes, operators, image pulls, CNI/CSI, GPU stack and applications, and verify there is no runtime internet dependency.

## Architecture

```text
Internet / Red Hat Registries
          |
     Connected Mirror Host
       oc / oc-mirror
          |
          | approved transfer
          v
================ AIR GAP ================
          |
   Internal Registry
   registry.company.com
          |
          +-----------------------------+
          |                             |
   OpenShift release              Operator/App images
          |                             |
          +-------------+---------------+
                        |
                 OpenShift Cluster
```

## Where Is the Internal Registry Configured?

The internal registry itself normally exists as an infrastructure service reachable by the disconnected OpenShift cluster, for example:

```text
registry.company.com
        |
        +-- OpenShift release images
        +-- Operator images
        +-- Application/GPU images
        |
        v
OpenShift Cluster
```

Inside OpenShift, configure the cluster to use the mirror rather than public registries.

### ImageDigestMirrorSet (IDMS)

For digest-based image mirroring, OpenShift uses `ImageDigestMirrorSet`.

Conceptual example:

```yaml
apiVersion: config.openshift.io/v1
kind: ImageDigestMirrorSet
metadata:
  name: internal-mirror
spec:
  imageDigestMirrors:
  - source: quay.io/example
    mirrors:
    - registry.company.com/example
```

Flow:

```text
Pod / Operator requests image
          |
          v
OpenShift mirror configuration
          |
          v
registry.company.com
          |
          v
Image pulled internally
```

For tag-based mirroring, `ImageTagMirrorSet` (ITMS) may also be used where appropriate.

## Operators

Disconnected operator content is mirrored into the internal registry. A `CatalogSource` points OpenShift/OLM to the mirrored operator catalog.

```text
Internal Registry
       |
Mirrored Operator Catalog
       |
   CatalogSource
       |
      OLM
       |
   Operator installation
```

The operator's required operand images must also be available internally.

## Application and Helm Images

Application workloads must use images available from approved internal registries.

```text
External image
     |
Mirror / scan / approve
     |
Internal Registry
     |
OpenShift workload
```

If a Helm chart references a public image, mirror the image internally and update the chart values/manifests to reference the internal location.

## Internal CA

If the internal registry uses a corporate/internal CA certificate, configure OpenShift to trust that CA so cluster nodes can securely pull images.

## Controlled OpenShift Upgrade

```text
Target OpenShift release
        |
Mirror target release
        |
Mirror compatible operators
        |
Validate supported upgrade path
        |
Check cluster health / recovery readiness
        |
Validate CNI / CSI / GPU compatibility
        |
Perform controlled upgrade
        |
Validate cluster and workloads
```

For GPU environments also validate:

```text
OpenShift
   |
RHCOS / kernel
   |
NVIDIA GPU Operator
   |
NVIDIA driver
   |
CUDA runtime
   |
AI workloads
```

## Interview Answer

> For a disconnected OpenShift environment, I would establish an internal registry and use a connected mirror host with oc-mirror to mirror the required OpenShift release, operator catalogs, and application or GPU images. For a fully air-gapped site, I would transfer that mirrored content through the approved offline mechanism and import it into the internal registry. Inside OpenShift, I would configure the mirror policies such as IDMS/ITMS and the required CatalogSource so nodes and OLM consume internal content. For upgrades, I would mirror the target release and compatible operators first, validate the upgrade path and dependencies, perform the controlled upgrade, and then validate the platform and workloads.

## Quick Memory Line

**Mirror OpenShift → Mirror Operators → Mirror App/GPU images → Transfer across air gap → Internal Registry → IDMS/ITMS + CatalogSource → Install/Upgrade → Validate**
