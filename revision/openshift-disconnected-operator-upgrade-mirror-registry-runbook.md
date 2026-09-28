# OpenShift Disconnected Operator Upgrade Using Mirror Registry — Consolidated Runbook

> **Scope:** Upgrade an Operator on an **existing OpenShift cluster** by using a **mirror registry** in a disconnected or air-gapped environment.
>
> **Out of scope:** Installing OpenShift, mirroring the OpenShift release payload, or performing an OpenShift cluster version upgrade.

---

## 1. End-to-End Flow

```text
Prepare RHEL mirror-registry host
        |
        v
Create DNS record for registry hostname
        |
        v
Install mirror-registry (Quay-based)
        |
        v
Configure TLS + credentials
        |
        v
Make OpenShift trust/authenticate to registry
        |
        v
Install oc-mirror on mirroring host
        |
        v
Identify current Operator / channel / target version
        |
        v
Create Operator-only ImageSetConfiguration
        |
        +-------------------------------+
        |                               |
Partial/connected                 Fully air-gapped
mirror-to-mirror                 mirror-to-disk
        |                               |
        |                         transfer media
        |                               |
        |                         disk-to-mirror
        +---------------+---------------+
                        |
                        v
                Internal Mirror Registry
                        |
                        v
             Apply CatalogSource / IDMS / ITMS
                        |
                        v
                       OLM
                        |
              Subscription -> InstallPlan
                        |
                        v
                    New CSV
                        |
                        v
               Operator upgraded
                        |
                        v
             Validate managed workloads
```

---

# PART A — SET UP THE MIRROR REGISTRY

## 2. Prepare the Mirror Registry Host

Use a dedicated RHEL host reachable by the OpenShift nodes and the mirroring/bastion host.

Example:

```text
Hostname : mirror.example.com
IP       : 10.20.30.40
OS       : RHEL 8/9
Port     : 8443 (default mirror-registry HTTPS port)
Storage  : Persistent local/enterprise storage
Tools    : Podman, OpenSSL
```

For a lab or smaller environment, Red Hat's **mirror registry for Red Hat OpenShift** is suitable.

For a large production environment requiring HA, scale, backup, and enterprise lifecycle management, use an appropriately designed enterprise registry such as Red Hat Quay.

---

## 3. DNS Requirement

### Production

Create the registry hostname in the organization's **internal/corporate DNS**.

Example:

```text
mirror.example.com -> 10.20.30.40
```

All systems that use the registry should resolve the same name:

- OpenShift control-plane nodes
- OpenShift worker nodes
- Connected mirroring host
- Disconnected bastion/mirroring host
- Administrative hosts running `oc`, `oc-mirror`, or `podman`

Recommended flow:

```text
OpenShift Nodes / Mirror Host
            |
            v
     Corporate/Internal DNS
            |
            v
      mirror.example.com
            |
            v
       10.20.30.40
```

### Lab

An internal DNS server is also fine if every OpenShift node and mirroring host uses it.

Using only `/etc/hosts` is acceptable for a temporary lab, but avoid it in production because the entry has to be maintained manually on every relevant system.

---

## 4. TLS Requirement

The TLS certificate must match the registry DNS hostname.

Example SAN:

```text
DNS:mirror.example.com
```

The complete requirement is:

```text
DNS resolves mirror.example.com
          +
TLS certificate is valid for mirror.example.com
          +
OpenShift trusts the CA
          +
OpenShift has registry credentials
          |
          v
Reliable image pulls from mirror registry
```

---

## 5. Download the Mirror Registry Package

Download the current `mirror-registry.tar.gz` package from the Red Hat OpenShift downloads page.

Extract it:

```bash
tar -xvf mirror-registry.tar.gz
cd mirror-registry
```

The package contains the `mirror-registry` executable.

---

## 6. Install the Mirror Registry

### Local-host example

```bash
./mirror-registry install \
  --quayHostname mirror.example.com \
  --quayRoot /opt/quay
```

### Remote-host example

```bash
./mirror-registry install -v \
  --targetHostname mirror.example.com \
  --targetUsername <user> \
  -k ~/.ssh/id_rsa \
  --quayHostname mirror.example.com \
  --quayRoot /opt/quay
```

Important:

- `--quayHostname` must resolve by DNS.
- Do not use an IP address as the Quay hostname.
- If no external certificate is supplied, mirror-registry generates its own CA/certificate.
- An initial user named `init` is created and a generated password is displayed after installation.

Typical layout:

```text
/opt/quay
   |
   +-- configuration
   +-- storage
   +-- rootCA.pem
   +-- TLS assets
```

---

## 7. Verify Registry Access

Initial test:

```bash
podman login \
  -u init \
  -p '<password>' \
  mirror.example.com:8443
```

If the local host does not trust the generated CA yet, a temporary test can use:

```bash
podman login \
  -u init \
  -p '<password>' \
  mirror.example.com:8443 \
  --tls-verify=false
```

Do **not** use `--tls-verify=false` as the permanent production design.

Registry API test:

```bash
curl -k https://mirror.example.com:8443/v2/
```

A successful response such as `{}` indicates that the registry API is reachable.

---

## 8. Trust the Mirror Registry CA on the Mirroring Host

If using the generated/internal CA:

```bash
sudo cp /opt/quay/rootCA.pem \
  /etc/pki/ca-trust/source/anchors/mirror-registry-ca.pem

sudo update-ca-trust
```

Then retry:

```bash
podman login mirror.example.com:8443
```

It should now work without `--tls-verify=false`.

---

# PART B — MAKE OPENSHIFT TRUST AND AUTHENTICATE TO THE REGISTRY

## 9. Add the Mirror Registry CA to OpenShift

If the registry uses an internal/private CA, OpenShift nodes must trust that CA.

For a registry with a port, the ConfigMap key replaces `:` with `..`.

Example:

```text
mirror.example.com:8443
        becomes
mirror.example.com..8443
```

Create the ConfigMap:

```bash
oc create configmap registry-config \
  --from-file=mirror.example.com..8443=/opt/quay/rootCA.pem \
  -n openshift-config
```

Configure OpenShift to use it:

```bash
oc patch image.config.openshift.io/cluster \
  --type=merge \
  -p '{"spec":{"additionalTrustedCA":{"name":"registry-config"}}}'
```

Verify:

```bash
oc get image.config.openshift.io/cluster -o yaml
```

---

## 10. Add Mirror Registry Credentials to the Global Pull Secret

Retrieve the current pull secret:

```bash
oc get secret/pull-secret \
  -n openshift-config \
  --template='{{index .data ".dockerconfigjson" | base64decode}}' \
  > pull-secret.json
```

Add the internal registry credentials:

```bash
oc registry login \
  --registry="mirror.example.com:8443" \
  --auth-basic="init:<password>" \
  --to=pull-secret.json
```

Update the cluster secret:

```bash
oc set data secret/pull-secret \
  -n openshift-config \
  --from-file=.dockerconfigjson=pull-secret.json
```

At this point:

```text
OpenShift
   |
   +-- DNS resolves mirror.example.com
   +-- CA trusts mirror.example.com
   +-- Pull secret authenticates to mirror.example.com
   |
   v
mirror.example.com:8443
```

---

# PART C — PREPARE THE OPERATOR UPGRADE

## 11. Install the Required CLI Tools

On the connected mirroring host, install:

- `oc`
- `oc-mirror`
- `podman`
- Red Hat pull-secret credentials
- Mirror-registry credentials

Check:

```bash
oc version
oc-mirror --help
podman login registry.redhat.io
podman login mirror.example.com:8443
```

For current workflows, use **oc-mirror v2** explicitly with `--v2`.

---

## 12. Check the Currently Installed Operator

List Subscriptions and CSVs:

```bash
oc get subscriptions -A
oc get csv -A
```

Inspect the Operator subscription:

```bash
oc get subscription <subscription-name> \
  -n <operator-namespace> \
  -o yaml
```

Capture:

```text
Operator package
Operator namespace
Current channel
Catalog source
Current CSV
InstallPlan approval mode
Target version
Target channel
```

Example:

```yaml
spec:
  channel: stable
  source: redhat-operators
  installPlanApproval: Manual

status:
  installedCSV: operator.v1.2.0
  currentCSV: operator.v1.2.0
```

Meaning:

```text
installedCSV = version currently running
currentCSV   = version OLM currently wants installed
```

---

## 13. Confirm the Supported Upgrade Path

Before mirroring:

1. Confirm the target Operator supports the current OpenShift version.
2. Confirm whether the upgrade stays in the same channel.
3. Confirm whether an intermediate Operator version/channel is required.
4. Confirm the Operator's CRD/API compatibility and any release-note prerequisites.
5. Confirm the managed application/operand compatibility.

Do not simply mirror one arbitrary target bundle if the Operator expects an upgrade graph through intermediate versions.

---

## 14. Find Operator Package and Channels

Example:

```bash
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.20
```

For one package:

```bash
oc mirror list operators \
  --catalog=registry.redhat.io/redhat/redhat-operator-index:v4.20 \
  --package=<operator-package>
```

Important: when selecting a non-default channel, include the package's **default channel** as required by the catalog/mirroring rules.

---

# PART D — CREATE AN OPERATOR-ONLY IMAGESETCONFIGURATION

## 15. ImageSetConfiguration

Because this runbook is only for an Operator upgrade, there is **no OpenShift platform payload** in the configuration.

Example:

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration

mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: <operator-package>
      channels:
      - name: <operator-channel>
```

Example:

```yaml
apiVersion: mirror.openshift.io/v2alpha1
kind: ImageSetConfiguration

mirror:
  operators:
  - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20
    packages:
    - name: loki-operator
      channels:
      - name: stable
```

Use the Red Hat Operator index corresponding to the OpenShift release/catalog you are operating.

If you need to bound versions, `minVersion` / `maxVersion` can be used, but use them carefully so you do not remove required upgrade-path bundles.

---

# PART E — MIRROR THE OPERATOR CONTENT

## 16. Scenario A — Partially Disconnected / Mirror-to-Mirror

Use this if the mirroring host can access both:

- `registry.redhat.io`
- `mirror.example.com:8443`

Flow:

```text
registry.redhat.io
        |
        v
   oc-mirror host
        |
        v
mirror.example.com:8443
```

Command:

```bash
oc-mirror \
  --config imageset-config.yaml \
  --workspace file:///data/oc-mirror-workspace \
  docker://mirror.example.com:8443 \
  --v2
```

For oc-mirror v2, `--workspace` is required for mirror-to-mirror.

---

## 17. Scenario B — Fully Air-Gapped

This is the common disconnected interview scenario.

### Connected side — mirror to disk

```text
registry.redhat.io
        |
        v
Connected oc-mirror host
        |
        v
/data/operator-images
```

Run:

```bash
oc-mirror \
  --config imageset-config.yaml \
  file:///data/operator-images \
  --v2
```

This downloads the selected:

- Operator catalog content
- Operator bundles
- Operator controller images
- Referenced related/operand images discovered through the catalog content

Preserve the generated metadata/cache/workspace. Do not casually delete it, because it is important for subsequent incremental mirroring operations.

### Transfer across the air gap

```text
/data/operator-images
        |
        | approved USB / secure file-transfer process
        v
Disconnected bastion host
```

### Disconnected side — disk to mirror

```bash
oc-mirror \
  --config imageset-config.yaml \
  --from file:///data/operator-images \
  docker://mirror.example.com:8443 \
  --v2
```

Now the Operator content is stored inside the internal mirror registry.

---

# PART F — CONNECT OPENSHIFT/OLM TO THE MIRRORED CONTENT

## 18. Locate Generated Cluster Resources

oc-mirror v2 generates cluster resources under a path similar to:

```text
<workspace-or-file-path>/
  working-dir/
    cluster-resources/
```

Typically you will find:

```text
ImageDigestMirrorSet (IDMS)
ImageTagMirrorSet    (ITMS)
CatalogSource
```

Purpose:

```text
IDMS
  = redirect digest-based image pulls to the mirror

ITMS
  = redirect tag-based image pulls where applicable

CatalogSource
  = expose the mirrored Operator catalog to OLM
```

---

## 19. Apply the Generated Resources

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

---

## 20. Verify the CatalogSource

```bash
oc get catalogsource -n openshift-marketplace
```

Inspect it:

```bash
oc describe catalogsource <catalog-name> \
  -n openshift-marketplace
```

Check marketplace pods:

```bash
oc get pods -n openshift-marketplace
```

Expected flow:

```text
Mirror Registry
      |
      v
Mirrored Operator catalog image
      |
      v
CatalogSource
      |
      v
OLM
```

---

# PART G — PERFORM THE OPERATOR UPGRADE

## 21. Confirm OLM Sees the New Version

Check the Subscription:

```bash
oc get subscription <subscription-name> \
  -n <operator-namespace> \
  -o yaml
```

Example state during upgrade discovery:

```yaml
status:
  installedCSV: operator.v1.2.0
  currentCSV: operator.v1.3.0
```

Interpretation:

```text
Currently installed = 1.2.0
OLM target          = 1.3.0
```

---

## 22. Change the Subscription Channel Only If Required

If the documented upgrade path requires moving channels:

```bash
oc patch subscription <subscription-name> \
  -n <operator-namespace> \
  --type merge \
  -p '{"spec":{"channel":"<target-channel>"}}'
```

Example:

```text
stable-1.2 -> stable-1.3
```

If the old and new versions are both in the same `stable` channel, no channel change is needed.

---

## 23. InstallPlan

For a controlled production upgrade, `Manual` approval gives you a review gate.

Check:

```bash
oc get installplan -n <operator-namespace>
```

Example:

```text
NAME            APPROVAL   APPROVED
install-abc12   Manual     false
```

Review:

```bash
oc describe installplan install-abc12 \
  -n <operator-namespace>
```

Approve:

```bash
oc patch installplan install-abc12 \
  -n <operator-namespace> \
  --type merge \
  -p '{"spec":{"approved":true}}'
```

If `installPlanApproval: Automatic`, OLM approves it automatically.

---

## 24. What OLM Does

You do **not** manually edit the Operator Deployment image.

The flow is:

```text
Mirrored Catalog
      |
      v
CatalogSource refreshed
      |
      v
Subscription sees newer bundle/CSV
      |
      v
InstallPlan created
      |
      v
InstallPlan approved
      |
      v
New CSV installed
      |
      +-- CRDs / APIs updated as defined
      +-- RBAC updated
      +-- Webhooks updated
      +-- Operator Deployment updated
      +-- New controller image pulled from mirror
      |
      v
Old CSV replaced according to OLM upgrade path
```

Component meanings:

```text
mirror-registry = stores images/catalog content

oc-mirror       = copies the required catalog, bundles and images

CatalogSource   = tells OLM where the mirrored catalog is

Subscription    = defines Operator package/channel and approval policy

InstallPlan     = concrete OLM installation/upgrade plan

CSV             = installed Operator version + install strategy metadata

OLM             = orchestrates the Operator lifecycle/upgrade
```

---

# PART H — VALIDATION

## 25. Validate the Upgrade

Watch the Subscription:

```bash
oc get subscription <subscription-name> \
  -n <operator-namespace> \
  -w
```

Check CSV:

```bash
oc get csv -n <operator-namespace>
```

Expected:

```text
operator.v1.3.0   Succeeded
```

Check Subscription again:

```bash
oc get subscription <subscription-name> \
  -n <operator-namespace> \
  -o yaml
```

Successful final state:

```yaml
status:
  installedCSV: operator.v1.3.0
  currentCSV: operator.v1.3.0
```

Check Operator pods:

```bash
oc get pods -n <operator-namespace>
```

Check logs:

```bash
oc logs deployment/<operator-controller> \
  -n <operator-namespace>
```

Check recent events:

```bash
oc get events \
  -n <operator-namespace> \
  --sort-by=.lastTimestamp
```

Finally validate:

- Operator custom resources
- Managed operands/workloads
- CRD/API compatibility
- Controller reconciliation
- Application functionality
- No image pull is trying to reach the public registry

Do **not** stop validation at only `CSV=Succeeded`.

---

# PART I — QUICK TROUBLESHOOTING

## 26. ImagePullBackOff / x509 Errors

### Symptoms

```text
ImagePullBackOff
x509: certificate signed by unknown authority
unauthorized: authentication required
```

Check:

```bash
oc describe pod <pod> -n <namespace>
```

Validate:

1. DNS resolves `mirror.example.com`
2. TLS SAN contains `mirror.example.com`
3. OpenShift `additionalTrustedCA` contains the registry CA
4. Global pull secret contains registry credentials
5. Firewall allows registry port 8443
6. Image exists in the mirror registry

---

## 27. CatalogSource Not Ready

Check:

```bash
oc get catalogsource -n openshift-marketplace
oc describe catalogsource <name> -n openshift-marketplace
oc get pods -n openshift-marketplace
```

Likely causes:

- Catalog image was not mirrored
- Wrong registry hostname/path
- Registry CA not trusted
- Authentication failure
- DNS/network connectivity issue
- Generated CatalogSource not applied

---

## 28. No InstallPlan Created

Check:

```bash
oc get subscription <name> -n <namespace> -o yaml
oc get csv -n <namespace>
oc get catalogsource -n openshift-marketplace
```

Likely causes:

- Target version is not present in the mirrored catalog
- Wrong Subscription channel
- Required default channel was omitted during mirroring
- Unsupported/missing upgrade edge between CSVs
- CatalogSource has not refreshed
- Dependency Operator/bundle was not mirrored

---

## 29. InstallPlan Exists but Upgrade Does Not Start

If approval is manual:

```bash
oc get installplan -n <namespace>
```

Check whether:

```text
APPROVAL = Manual
APPROVED = false
```

If so, review and approve it.

---

## 30. New CSV Fails

```bash
oc describe csv <new-csv> -n <namespace>
oc get events -n <namespace> --sort-by=.lastTimestamp
oc logs deployment/<operator-controller> -n <namespace>
```

Look for:

- Missing image
- Missing dependency
- CRD conversion/webhook errors
- RBAC errors
- Unsupported upgrade path
- Operator-specific prerequisites
- Operand compatibility issues

---

# PART J — 15-STEP INTERVIEW ANSWER

1. Prepare a RHEL host for the mirror registry.
2. Create an internal/corporate DNS record such as `mirror.example.com`.
3. Install Red Hat mirror-registry/Quay and configure persistent storage.
4. Configure TLS; the certificate SAN must match the registry DNS name.
5. Add the registry CA to OpenShift using `additionalTrustedCA`.
6. Add mirror-registry credentials to the OpenShift global pull secret.
7. Install `oc` and `oc-mirror` on the mirroring host.
8. Check the existing Operator's Subscription, CSV, channel and supported target version.
9. Create an Operator-only `ImageSetConfiguration`.
10. Use oc-mirror v2 to mirror the required catalog, bundles and related images.
11. For a fully air-gapped site, mirror to disk, transfer the content, then load it into the internal registry.
12. Apply the generated `CatalogSource`, `IDMS` and `ITMS`.
13. OLM detects the newer Operator version and creates an `InstallPlan`.
14. Review/approve the InstallPlan; OLM installs the new CSV and updates the Operator.
15. Validate the CSV, Operator pods, logs, CRs, operands and application functionality.

---

# 31. 30-Second Interview Pitch

> We already have the OpenShift cluster, so I am not installing or upgrading OpenShift. I first provide an internal mirror registry with proper corporate DNS, TLS, storage and credentials. I make the existing OpenShift cluster trust that registry CA and add its credentials to the global pull secret. I then identify the currently installed Operator CSV, channel and supported target upgrade path. Using an Operator-only ImageSetConfiguration and oc-mirror v2, I mirror the Operator catalog, bundles and related images. In a fully disconnected environment I mirror to disk, transfer the archive across the air gap and push it into the internal registry. I apply the generated CatalogSource, IDMS and ITMS resources. OLM then discovers the new Operator through the mirrored catalog, creates an InstallPlan, I review and approve it, and OLM installs the new CSV. Finally I validate the Operator pods, custom resources, managed workloads and confirm there are no public-registry dependencies.

---

# 32. Quick Memory Flow

```text
DNS
 -> Mirror Registry
 -> TLS / CA / Pull Secret
 -> oc-mirror
 -> ImageSetConfiguration
 -> Mirror catalog + bundles + images
 -> Transfer across air gap
 -> Internal Registry
 -> IDMS / ITMS / CatalogSource
 -> Subscription
 -> InstallPlan
 -> CSV
 -> Validate
```

---

## Reference Notes

- Use **oc-mirror v2** with `mirror.openshift.io/v2alpha1`.
- Mirror-to-mirror in v2 requires a workspace.
- Mirror-to-disk and disk-to-mirror are the standard fully disconnected workflow.
- Keep oc-mirror workspace/cache/metadata on reliable storage for future incremental updates.
- If the mirror registry uses a port, the OpenShift CA ConfigMap key replaces `:` with `..`.
- The registry hostname must resolve through DNS and its TLS certificate must match that hostname.
- For Operator-only work, do **not** add an OpenShift `platform:` section to the ImageSetConfiguration.
