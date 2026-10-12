# 03 — Air-Gap / Offline Installation (Hauler Primary, Skopeo Secondary)

This file covers fully offline (air-gapped) installations where the target cluster has **no internet access**.
Hauler is the **primary** path: it bundles all container images and the Helm chart into a single portable archive, transfers it across the air gap, and loads it into your private registry. Skopeo is retained as a secondary option for copying **individual** images only.

📖 Source: https://docs.datastax.com/en/mission-control/install/install-airgap.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html
📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

> ⚠️ **Replicated (KOTS) is no longer supported.** Support for both KOTS and Replicated was removed in Mission Control 1.20.1. Air-gapped installs now mirror images directly with Hauler (or Skopeo for individual images). If you are migrating an existing air-gapped Replicated/KOTS install, see the migration section in `02-install-online.md` — the same ICR image-redirect overrides apply, just sourced from your private registry instead of `icr.io` directly.

---

## Overview: two phases

**Phase 1 — On an internet-connected machine:** Hauler pulls all images and the chart from a `hauler-manifest.yaml` and saves a portable `mission-control.tar.zst`.

**Phase 2 — On the air-gapped cluster:** Hauler loads the archive into a store, copies images/charts to your private registry, and Helm installs Mission Control from `oci://<REGISTRY>/hauler/mission-control`.

The 1.21.0 Hauler manifest is included below. **Always use the manifest from the release notes of your target version** — image digests, versions, and the chart `repoURL` change every release.

---

## Prerequisites

📖 Source: https://docs.datastax.com/en/mission-control/install/install-airgap.html

On the **internet-connected** machine, ensure you have:

- [Hauler](https://github.com/hauler-dev/hauler) installed.
- [Docker](https://www.docker.com/) installed and running.
- An IBM entitlement key from the [IBM Container Library](https://myibm.ibm.com/products-services/containerlibrary).
- The Mission Control CLI tool, `mcctl`, installed. For a Linux tarball, contact [IBM Support](https://support.datastax.com).
- A Hauler manifest file — e.g. `hauler-manifest.yaml`. See the 1.21.0 manifest in this page, or take the one from the [release notes](https://docs.datastax.com/en/mission-control/release-notes/release-notes.html) of your target version.
- Sufficient disk space for the archive (typically 5 to 10 GB).

On the **air-gapped** machine, ensure you have:

- Hauler installed.
- `kubectl` configured to access the target Kubernetes cluster.
- `helm` installed.
- Access to a private container registry.
- Registry credentials (if authentication is required).
- `cert-manager` installed and configured in the cluster. For installation instructions, see [Install cert-manager](https://docs.datastax.com/en/mission-control/install/kubernetes.html#install-cert-manager).

---

## 1.21.0 Hauler manifest

📖 Source: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

This manifest ships with the 1.21.0 release. **Always take the manifest from the release notes of your target version** — image digests, versions, and the chart `repoURL` change every release. Note that as of 1.21.0 the `cass-config-builder` image moved to `ghcr.io` and is pinned to a SHA-256 digest, and HCD is `hcd:2.0.7-ubi`.

```yaml
apiVersion: content.hauler.cattle.io/v1
kind: Images
metadata:
  name: mission-control-images
spec:
  images:
    - name: cp.icr.io/cp/ibm-ds-mission-control/cql-router:v1.20.3
    - name: cp.icr.io/cp/ibm-ds-mission-control/cqlsh-pod:v1.21.0
    - name: cp.icr.io/cp/ibm-ds-mission-control/dse-mgmtapi:6.8.64-ubi
    - name: cp.icr.io/cp/ibm-ds-mission-control/dse-mgmtapi:6.9.23-ubi
    - name: cp.icr.io/cp/ibm-ds-mission-control/hcd:2.0.7-ubi
    - name: docker.io/grafana/grafana:12.4.9
    - name: docker.io/grafana/loki-canary:3.6.7
    - name: docker.io/grafana/loki:3.6.11
    - name: docker.io/grafana/mimir:3.1.5
    - name: docker.io/haproxytech/kubernetes-ingress:3.2.14
    - name: docker.io/k8ssandra/cass-operator:v1.32.0
    - name: docker.io/k8ssandra/k8ssandra-operator:v1.33.0
    - name: docker.io/kiwigrid/k8s-sidecar:2.10.1
    - name: docker.io/nginxinc/nginx-unprivileged:1.31-alpine3.23-slim
    - name: ghcr.io/datastax/cass-config-builder:1.0.22-ubi@sha256:60849108bc9f582714ac955a3594ed39ad39bc5e528ef421d07b1882875aeb97
    - name: icr.io/datastax-mission-control/k8ssandra-client:mission-control-1.21.0
    - name: icr.io/datastax-mission-control/mission-control-dex:v1.21.0
    - name: icr.io/datastax-mission-control/mission-control-ui:v1.21.0
    - name: icr.io/datastax-mission-control/mission-control:v1.21.0
    - name: k8ssandra/cass-management-api:4.0.20-ubi
    - name: k8ssandra/cass-management-api:4.1.11-ubi
    - name: k8ssandra/cass-management-api:5.0.8-ubi
    - name: k8ssandra/k8ssandra-client:v0.9.1
    - name: k8ssandra/medusa:0.30.1
    - name: k8ssandra/system-logger:v1.32.0
    - name: quay.io/minio/mc:RELEASE.2025-08-13T08-35-41Z
    - name: quay.io/minio/minio:RELEASE.2025-09-07T16-13-09Z
    - name: registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.20.0
    - name: stargateio/data-api:v1.0.51
    - name: thelastpickle/cassandra-reaper:5.0.1
    - name: timberio/vector:0.56.0-distroless-libc
---
apiVersion: content.hauler.cattle.io/v1
kind: Charts
metadata:
  name: mission-control-charts
spec:
  charts:
    - name: mission-control
      repoURL: oci://icr.io/mission-control-helm
      version: "1.21.0"
      add-images: true
```

---

## Create the archive (internet-connected side)

Perform these steps on a system with internet access.

1. Log in to the IBM Entitled Registry using your entitlement key:

```bash
docker login cp.icr.io -u cp -p ENTITLEMENT_KEY
```

2. Fetch all images and charts listed in the manifest:

```bash
hauler store sync \
  --store ./hauler-store \
  --filename hauler-manifest.yaml
```

Hauler downloads all images and charts referenced in the manifest and writes them to `./hauler-store`.

> ⚠️ After syncing, Hauler attempts to render the Helm chart to discover additional images. If the chart has required values (such as Loki storage configuration), you may see a warning like `failed to render chart [mission-control]: execution error ... Loki validation failed`. **This warning is expected and does not affect the store contents.**

3. Optional — verify that all artifacts were pulled successfully:

```bash
hauler store info --store ./hauler-store
```

Hauler lists all artifacts in the store.

4. Create a portable archive from the store:

```bash
hauler store save \
  --store ./hauler-store \
  --filename mission-control.tar.zst
```

Hauler writes the archive to `mission-control.tar.zst`.

5. Transfer `mission-control.tar.zst` to the air-gapped environment using your organization's approved method (for example, a USB drive or a secure file transfer to a jump host).

---

## Load the archive into the target registry (air-gapped side)

Perform these steps on the air-gapped system after transferring `mission-control.tar.zst`.

1. Restore the store from the archive:

```bash
hauler store load \
  --store ./store \
  --filename mission-control.tar.zst
```

Hauler restores the store to `./store`.

2. Confirm that all artifacts were loaded:

```bash
hauler store info --store ./store
```

3. Copy all images and charts from the store to your private registry. `REGISTRY` is your registry address and port (for example, `registry.example.com:5000`).

   Without authentication:

```bash
hauler store copy \
  --store ./store \
  --plain-http \
  registry://REGISTRY
```

   With credentials (if your registry requires authentication):

```bash
hauler store copy \
  --store ./store \
  --plain-http \
  --username USER \
  --password PASSWORD \
  registry://REGISTRY
```

Remove `--plain-http` if your registry uses TLS.

---

## Install Mission Control from the registry

📖 Source: https://docs.datastax.com/en/mission-control/install/install-airgap.html
📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html#configure-image-pull-secrets

1. Create the namespace:

```bash
kubectl create namespace mission-control
```

2. If your private registry requires authentication, create an image pull secret:

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=REGISTRY \
  --docker-username=USER \
  --docker-password=PASSWORD \
  --namespace mission-control
```

> ℹ️ Per the [security overrides](https://docs.datastax.com/en/mission-control/install/security-overrides.html#configure-image-pull-secrets) doc, since Mission Control 1.8.0 the operator automatically creates and replicates image pull secrets to all namespaces when you reference one via `global.imagePullSecrets`. The `helm-overrides.yaml` below already wires that reference up — the explicit secret above is the safe default for air-gap installs.

3. Create a file named `helm-overrides.yaml`, replacing `REGISTRY` with your registry host and port (for example, `registry.example.com:5000`):

```yaml
global:
  imageConfig:
    overrides:
      registry: "REGISTRY"
  imagePullSecrets:
    - name: registry-credentials

image:
  registry: "REGISTRY"

client:
  image:
    registry: "REGISTRY"

ui:
  image:
    registry: "REGISTRY"
    repository: "mission-control-ui"

aggregator:
  image:
    repository: "REGISTRY/timberio/vector"

k8ssandra-operator:
  image:
    registry: "REGISTRY"
  cass-operator:
    image:
      registry: "REGISTRY"

dex:
  image:
    repository: "REGISTRY/mission-control-dex"

grafana:
  enabled: true
  image:
    registry: "REGISTRY"
    repository: "grafana/grafana"
  sidecar:
    image:
      registry: "REGISTRY"
      repository: "kiwigrid/k8s-sidecar"

kube-state-metrics:
  image:
    registry: "REGISTRY"
    repository: "kube-state-metrics/kube-state-metrics"

loki:
  global:
    image:
      registry: "REGISTRY"
  image:
    registry: "REGISTRY"
    repository: "grafana/loki"
  gateway:
    image:
      registry: "REGISTRY"
      repository: "nginxinc/nginx-unprivileged"
  sidecar:
    image:
      registry: "REGISTRY"
      repository: "kiwigrid/k8s-sidecar"

mimir:
  image:
    repository: "REGISTRY/grafana/mimir"
  gateway:
    nginx:
      image:
        registry: "REGISTRY"
        repository: "nginxinc/nginx-unprivileged"
```

4. If your private registry requires authentication, log in so that Helm can pull the chart:

```bash
helm registry login REGISTRY \
  --username USER \
  --password PASSWORD
```

5. Install Mission Control from your private registry. Hauler publishes the chart under the `hauler/` path:

```bash
helm install mission-control oci://REGISTRY/hauler/mission-control \
  --version <CHART_VERSION> \
  --namespace mission-control \
  -f helm-overrides.yaml
```

Replace `REGISTRY` with your private registry address and `<CHART_VERSION>` with the chart version included in the Hauler bundle (for example, `1.21.0`).

6. Verify that all pods are running:

```bash
kubectl get pods -n mission-control
```

Wait until all pods show `Running` status.

7. Confirm that no pods pull images from outside your private registry:

```bash
kubectl get pods -n mission-control \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{range .spec.initContainers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u \
  | grep -v '^REGISTRY/'
```

Replace `REGISTRY` with your registry address. If this command produces output, an image is pulling from outside your private registry.

---

## Install a data plane from the registry

📖 Source: https://docs.datastax.com/en/mission-control/install/install-airgap.html

If you are deploying Mission Control across multiple Kubernetes clusters, each data-plane cluster must also pull images from a private registry. The Hauler archive you already transferred contains all required data-plane images — no additional bundling is needed. The private registry reachable from the data-plane cluster must contain those images before you run the data-plane install.

If the data plane cluster is in a different network zone, push the Hauler archive contents to a registry reachable from that cluster, following the same steps in [Load the archive into the target registry (air-gapped side)](#load-the-archive-into-the-target-registry-air-gapped-side).

1. Create the namespace on the data-plane cluster:

```bash
kubectl create namespace mission-control --context <DP_CONTEXT>
```

Replace `<DP_CONTEXT>` with the `kubectl` context name for the data-plane cluster.

2. If the data plane's private registry requires authentication, create an image pull secret:

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=REGISTRY \
  --docker-username=USER \
  --docker-password=PASSWORD \
  --namespace mission-control \
  --context <DP_CONTEXT>
```

3. Create a file named `dp-helm-overrides.yaml`, replacing `REGISTRY` with your data-plane registry and `CONTROL_PLANE_IP` with the IP or hostname of a control-plane node:

```yaml
# This cluster is a data plane only
controlPlane: false

# Redirect all image pulls to your private registry
global:
  imageConfig:
    overrides:
      registry: "REGISTRY"
  imagePullSecrets:
    - name: registry-credentials

image:
  registry: "REGISTRY"

client:
  image:
    registry: "REGISTRY"

aggregator:
  image:
    repository: "REGISTRY/timberio/vector"
    customConfig:
      sources:
        mimir-self-monitoring: {}
      sinks:
        mimir: {}
        loki: {}
        control_plane_aggregator:
          type: vector
          inputs: ["vector", "internal_metrics", "kube_state_metrics", "cass_operator_metrics"]
          address: "CONTROL_PLANE_IP:30600"

k8ssandra-operator:
  image:
    registry: "REGISTRY"
  cass-operator:
    image:
      registry: "REGISTRY"

# Disable control-plane-only components
ui:
  enabled: false

dex:
  enabled: false

grafana:
  enabled: false

loki:
  enabled: false

mimir:
  enabled: false

agent:
  enabled: false
```

> ⚠️ **Use the same release name (`mission-control`) as the control-plane installation.** Mismatched release names break communication between the planes.

4. If your registry uses plain HTTP, add `--insecure` to the registry login:

```bash
helm registry login REGISTRY \
  --username USER \
  --password PASSWORD \
  --insecure
```

5. Install Mission Control in data-plane mode:

```bash
helm install mission-control oci://REGISTRY/hauler/mission-control \
  --version <CHART_VERSION> \
  --namespace mission-control \
  --kube-context <DP_CONTEXT> \
  -f dp-helm-overrides.yaml
```

Replace `REGISTRY` with your private registry address, `<CHART_VERSION>` with the chart version matching the control-plane version (for example, `1.21.0`), and `<DP_CONTEXT>` with the `kubectl` context for the data-plane cluster.

6. Verify that all pods are running on the data-plane cluster:

```bash
kubectl get pods -n mission-control --context <DP_CONTEXT>
```

7. Confirm that no data-plane pods pull images from outside your private registry:

```bash
kubectl get pods -n mission-control --context <DP_CONTEXT> \
  -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{range .spec.initContainers[*]}{.image}{"\n"}{end}{end}' \
  | sort -u \
  | grep -v '^REGISTRY/'
```

8. Register the data plane with the control plane using the `mcctl` CLI:

```bash
mcctl register --source-context <DP_CONTEXT> --dest-context <CP_CONTEXT>
```

Replace `<DP_CONTEXT>` with the `kubectl` context for the data-plane cluster and `<CP_CONTEXT>` with the `kubectl` context for the control-plane cluster.

---

## Troubleshoot installation issues

📖 Source: https://docs.datastax.com/en/mission-control/install/install-airgap.html

**Hauler store sync fails:** If `hauler store sync` fails to download images, do the following:
- Verify internet connectivity.
- Check that the manifest file path is correct.
- Ensure sufficient disk space.

**Registry push fails:** If `hauler store copy` fails, do the following:
- Verify registry connectivity.
- Check registry credentials.
- Ensure the registry supports OCI artifacts.

**Helm install fails with image pull errors:** If pods fail to start with `ImagePullBackOff`, do the following:
- Verify that all images are present in your private registry.
- Check that the registry overrides in `helm-overrides.yaml` are correct.
- If your registry requires authentication, verify that the pull secret exists in the namespace where the failing pod is running:

```bash
kubectl get secret registry-credentials -n <NAMESPACE>
```

For more information, see [Configure image pull secrets](https://docs.datastax.com/en/mission-control/install/security-overrides.html#configure-image-pull-secrets).

**Pods pull from public registries:** If the verification commands show images from public registries, do the following:
- Review the `helm-overrides.yaml` file for missing or incorrect registry overrides.
- Check that all image references in the chart are properly overridden.

---

## Copy individual images with Skopeo

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

> ⚠️ Hauler is the primary and recommended air-gap path. Use Skopeo only to copy **individual** images that you did not include in the Hauler manifest, or where you cannot use Hauler.

> ⚠️ **KOTS CLI users must switch to Skopeo or Hauler.** KOTS support was removed in Mission Control 1.20.1, and the service is no longer available to earlier versions. If you previously used the KOTS CLI, switch to Skopeo (for individual images) or Hauler (for full bundles). See the migration section in `02-install-online.md` for details.

To copy an image from a public registry to your private registry, do the following:

1. Authenticate to your private registry:

```bash
skopeo login PRIVATE_REGISTRY_HOST
```

Replace `PRIVATE_REGISTRY_HOST` with the host name of the private registry.

2. Copy the container to your private registry:

```bash
skopeo copy docker://icr.io/datastax/dse-mgmtapi-6_9:6.9.23-ubi \
  docker://PRIVATE_REGISTRY_HOST/PRIVATE_REGISTRY_NAMESPACE/dse-mgmtapi-6_9:6.9.23-ubi
```

Replace the registry path with your target image registry and tag (see the 1.21.0 Hauler manifest above for the exact image list and tags).

3. Verify the copied image in your private registry:

```bash
skopeo inspect docker://PRIVATE_REGISTRY_HOST/IMAGE_NAME:TAG
```

Replace `IMAGE_NAME` and `TAG` with the image name to inspect and the tag of the image.

---

## Air-Gap Support Bundle

📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

Generate a support bundle to help troubleshoot issues when internet access is unavailable.

For an air-gapped cluster, use the `kots-support-bundle-spec.yaml` specification:

1. Download the default `kots.io` support bundle specification (`kots-support-bundle-spec.yaml`).
2. Upload the `kots-support-bundle-spec.yaml` file to the air-gapped server.
3. Create a support bundle that is saved to your local machine:

```bash
kubectl support-bundle LOCATION/kots-support-bundle-spec.yaml
```

Replace `LOCATION` with the path to the `kots-support-bundle-spec.yaml` file.

The support bundle is saved in the current directory with a timestamped filename in the format `support-bundle-YYYY-MM-DDThh_mm_ss.tar.gz`. Move it to a secure location and share it with [IBM Support](https://support.datastax.com) for assistance.

📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

---

## Next Steps

- Cloud-specific storage and ingress values → `04-cloud-config.md`
- TLS and auth → `05-security.md`
- Deploy HCD/DSE cluster → `06-cluster-ops.md`
- Migrating an existing air-gapped Replicated/KOTS install → `02-install-online.md` (Migrate an existing Replicated installation to ICR)
