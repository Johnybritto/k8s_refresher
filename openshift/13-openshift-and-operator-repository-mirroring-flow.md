# Mirroring OpenShift & Operator Repositories — Separate Flow

> **Scope:** Mirror both the OpenShift release repository and required Operator repositories into an internal registry for a disconnected/air-gapped environment.
>
> **Assumption:** The mirror registry already exists, DNS/TLS are configured, and the mirroring host can authenticate to the source registries and the internal registry.
>
> **Not covered here:** Mirror-registry installation, proxy/Squid setup, OpenShift installation, or the Operator upgrade workflow itself.

---

# 1. High-Level Architecture

```text
                 CONNECTED SIDE

       quay.io / registry.redhat.io
                  |
                  v
          Connected Mirror Host
           oc + oc-mirror v2
                  |
          +-------+--------+
          |                |
          v                v
 OpenShift release     Operator catalog
      images            bundles/images
          |                |
          +-------+--------+
                  |
                  v
            Mirror to disk
                  |
          Approved transfer
                  |
================ AIR GAP ================
                  |
                  v
        Disconnected Mirror Host
                  |
                  v
         Internal Mirror Registry
                  |
          +-------+--------+
          |                |
          v                v
 OpenShift releases    Operator catalogs
          |                |
          +-------+--------+
                  |
                  v
         Existing OpenShift Cluster
```

---

# 2. Why Mirror Both?

There are two different content sets:

```text
OpenShift platform content
    = OpenShift release payload/images
    = used for disconnected installation or OpenShift version upgrade

Operator content
    = Operator catalog + bundles + controller/related images
    = used for Operator installation or Operator upgrade
```

If you are only upgrading an Operator on an existing OpenShift cluster, the `platform:` section is not required.

If you are preparing a complete disconnected environment for OpenShift installation or OpenShift upgrades, mirror both.

---

# 3. Prerequisites

On the connected mirroring host:

- `oc`
- `oc-mirror`
- `podman`
- Red Hat pull secret
- Access to:
  - `quay.io`
  - `registry.redhat.io`
- Enough local disk space for the mirror archive
- Credentials for the internal mirror registry when pushing on the disconnected side

Verify:

```bash
oc version
oc-mirror --help
podman login registry.redhat.io
podman login quay.io
```

Use **oc-mirror v2** for current disconnected workflows.

---

# 4. Create a Combined ImageSetConfiguration

Example:

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration

mirror:

  platform:
    architectures:
    - amd64

    channels:
    - name: stable-4.20
      minVersion: 4.20.2
      maxVersion: 4.20.4
      shortestPath: true

    graph: true

  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: loki-operator
      channels:
      - name: stable

    - name: cluster-logging
      channels:
      - name: stable
```

This one file tells oc-mirror to collect:

```text
OpenShift release payload
        +
OpenShift update graph data
        +
Selected Operator packages
        +
Operator bundles
        +
Operator related images
```

---

# 5. OpenShift Platform Section Explained

Example:

```yaml
platform:
  architectures:
  - amd64

  channels:
  - name: stable-4.20
    minVersion: 4.20.2
    maxVersion: 4.20.4
    shortestPath: true

  graph: true
```

Meaning:

### architectures

```yaml
architectures:
- amd64
```

Mirrors only the required CPU architecture.

### channel

```yaml
name: stable-4.20
```

Selects the OpenShift update channel.

### minVersion / maxVersion

```yaml
minVersion: 4.20.2
maxVersion: 4.20.4
```

Limits the OpenShift releases to the required range.

### shortestPath

```yaml
shortestPath: true
```

Asks oc-mirror to include the shortest valid update path between the selected OpenShift versions.

### graph

```yaml
graph: true
```

Mirrors graph data used by the OpenShift Update Service for disconnected update workflows.

Important: for EUS/minor-version transitions, always validate the Red Hat update graph because an intermediate OpenShift minor release might still be required.

---

# 6. Operator Section Explained

Example:

```yaml
operators:
- catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
  packages:
  - name: loki-operator
    channels:
    - name: stable
```

This selects:

```text
Red Hat Operator Index
        |
        v
Selected Operator package
        |
        v
Selected channel
        |
        v
Bundles + Operator images + related images
```

Do not mirror the entire Operator catalog unless you need it; selecting only required packages reduces storage and transfer size.

Be careful when using Operator `minVersion` / `maxVersion` filtering because aggressive filtering can truncate the Operator update graph and create multiple channel heads.

---

# 7. Check Available OpenShift Releases

Use Red Hat's OpenShift update graph / channel information to confirm:

- Current OpenShift version
- Target OpenShift version
- Supported update path
- Required intermediate versions

For interview purposes:

```text
Current OCP
   |
Check supported upgrade graph
   |
Mirror current/target/intermediate versions as required
   |
Perform disconnected upgrade later
```

---

# 8. Check Operator Packages and Channels

List Operators from a catalog:

```bash
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.20
```

Inspect one package:

```bash
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.20 \
  --package=<operator-package>
```

Confirm:

- Package name
- Default channel
- Required channel
- Target versions
- Dependencies

For Operator filtering, include the default channel where required.

---

# 9. Fully Disconnected Flow — Mirror to Disk

On the connected host:

```bash
oc mirror \
  -c imageset-config.yaml \
  file:///data/oc-mirror-content \
  --v2
```

This collects:

```text
OpenShift release images
        +
OpenShift graph data
        +
Operator catalog images
        +
Operator bundles
        +
Operator controller images
        +
Related images
```

Verify the archive exists:

```bash
ls -lh /data/oc-mirror-content
```

---

# 10. Transfer Across the Air Gap

Move the generated content using the organization's approved mechanism:

```text
/data/oc-mirror-content
         |
         | USB / secure approved transfer
         v
Disconnected mirror host
```

oc-mirror does not define the physical transfer mechanism.

---

# 11. Disk to Internal Mirror Registry

On the disconnected side:

```bash
oc mirror \
  -c imageset-config.yaml \
  --from file:///data/oc-mirror-content \
  docker://mirror.example.com:8443 \
  --v2
```

Result:

```text
mirror.example.com:8443
       |
       +-- OpenShift release repositories
       |
       +-- OpenShift graph data
       |
       +-- Operator catalog repositories
       |
       +-- Operator bundles/images
```

---

# 12. Partially Disconnected Flow — Mirror to Mirror

If the mirroring host can access both the internet and the internal registry:

```text
quay.io / registry.redhat.io
              |
              v
        oc-mirror host
              |
              v
    mirror.example.com:8443
```

Command:

```bash
oc mirror \
  -c imageset-config.yaml \
  --workspace file:///data/oc-mirror-workspace \
  docker://mirror.example.com:8443 \
  --v2
```

For oc-mirror v2 mirror-to-mirror, maintain the workspace.

---

# 13. Generated Cluster Resources

After mirroring, oc-mirror v2 generates resources such as:

```text
working-dir/
  cluster-resources/
      |
      +-- ImageDigestMirrorSet
      +-- ImageTagMirrorSet
      +-- CatalogSource
      +-- UpdateService manifest when graph data is enabled
```

These resources connect OpenShift to the mirrored content.

---

# 14. Apply Mirror Resources

Example:

```bash
cd <workspace>/working-dir/cluster-resources

oc apply -f .
```

Verify:

```bash
oc get imagedigestmirrorset
oc get imagetagmirrorset
oc get catalogsource -n openshift-marketplace
```

Conceptually:

```text
OpenShift asks for external image
             |
             v
      IDMS / ITMS rules
             |
             v
    Internal Mirror Registry
```

---

# 15. How OpenShift Uses the Mirrored Release

OpenShift release content is used later for:

```text
Disconnected OpenShift installation
             or
Disconnected OpenShift version upgrade
```

The mirrored platform content is not what upgrades an individual Operator.

---

# 16. How OLM Uses the Mirrored Operator Catalog

```text
Internal Mirror Registry
          |
          v
   Mirrored catalog image
          |
          v
     CatalogSource
          |
          v
         OLM
          |
          v
Subscription -> InstallPlan -> CSV
```

That is the Operator lifecycle path.

---

# 17. Combined End-to-End Flow

```text
1. Decide target OpenShift release(s)
2. Decide required Operators
3. Validate OpenShift upgrade graph
4. Validate Operator channels/update paths
5. Create combined ImageSetConfiguration
6. Mirror OpenShift release payload
7. Mirror selected Operator packages
8. Mirror graph data if disconnected upgrades are required
9. Mirror to disk
10. Transfer across the air gap
11. Push disk content to internal registry
12. Apply generated IDMS/ITMS
13. Apply generated CatalogSource
14. Configure/use UpdateService if required
15. Validate OpenShift and Operator content from the mirror
```

---

# 18. Interview Answer

> I first identify which OpenShift releases and Operator packages are required. I then create a single oc-mirror v2 ImageSetConfiguration containing a platform section for the OpenShift release channel and versions, and an operators section for the required Red Hat Operator packages and channels. In a fully disconnected environment, I run oc-mirror on a connected host to create an archive, transfer that archive across the air gap, and run oc-mirror again to push the content into the internal mirror registry. oc-mirror generates IDMS, ITMS and CatalogSource resources, which I apply to the cluster so image pulls and OLM use the mirrored content. If I am also performing disconnected OpenShift upgrades, I mirror the update graph data and configure the OpenShift Update Service as required.

---

# 19. Important Difference to Remember

```text
platform:
   -> OpenShift release repositories
   -> OpenShift install / cluster upgrade

operators:
   -> Operator catalog + bundles + images
   -> Operator install / Operator upgrade
```

For an Operator-only upgrade on an already-running OpenShift cluster:

```text
operators:  REQUIRED
platform:   NOT REQUIRED
```

For a complete disconnected OpenShift lifecycle:

```text
platform:   REQUIRED
operators:  REQUIRED for Operators you intend to use
```

---

# 20. Quick Memory Line

**Select OCP versions + Operators -> ImageSetConfiguration -> oc-mirror -> disk -> air gap -> mirror registry -> IDMS/ITMS + CatalogSource -> OpenShift/OLM consume internal content**
