# 03 — Air-Gap / Offline Installation

This file covers fully offline (air-gapped) installations where the target cluster has **no internet access**.

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html (Airgap Helm section)
📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

> ⚠️ **Replicated is no longer supported.** There is no more airgap bundle download from `replicated.app` and no more `kubectl kots admin-console push-images`. Air-gapped installs now mirror images directly from `icr.io` / `cp.icr.io` to your private registry with **Skopeo**, then install with Helm using registry-override values. If migrating an existing air-gapped Replicated/KOTS install, see the migration section in `02-install-online.md` — the same ICR image-redirect overrides apply, just sourced from your private registry instead of `icr.io` directly.

---

## Overview: two phases

**Phase 1 — On an internet-connected machine:**
1. Obtain your IBM entitlement key
2. Mirror all required container images from `icr.io`/`cp.icr.io` (and other public registries) to your private registry with Skopeo
3. Mirror the Mission Control Helm chart itself (OCI artifact) to your private registry, or keep pulling it from `icr.io` if your air-gapped cluster has outbound access to `icr.io` specifically

**Phase 2 — On the air-gapped cluster:**
4. Install cert-manager from mirrored images
5. Create image pull secrets for your private registry
6. Install MC via Helm using an all-overrides `values-airgap.yaml`

---

## Phase 1A — Entitlement key

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

Get your IBM entitlement key from https://myibm.ibm.com/products-services/containerlibrary — sign in with an IBMid, click **Get entitlement key** (or **Copy key**). This key authenticates against `cp.icr.io` when mirroring entitled images.

---

## Phase 1B — Mirror images with Skopeo

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

```bash
# Install Skopeo (package manager or: https://github.com/containers/skopeo)

# Log in to the source registry for entitled images
skopeo login cp.icr.io --username cp --password YOUR_ENTITLEMENT_KEY

# Log in to your private registry
skopeo login YOUR_PRIVATE_REGISTRY

# Copy each required image (repeat for all images below, per your MC version)
skopeo copy \
  docker://icr.io/datastax-mission-control/mission-control:v1.20.1 \
  docker://YOUR_REGISTRY/YOUR_NS/mission-control:v1.20.1

# Verify a copy succeeded
skopeo inspect docker://YOUR_REGISTRY/YOUR_NS/mission-control:v1.20.1
```

### Images to mirror (update tags per your MC version from release notes)

📖 Release notes with exact image tags: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html
📖 Full tag/registry list: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

```
# Mission Control platform (public — no entitlement key needed)
icr.io/datastax-mission-control/mission-control:<version>
icr.io/datastax-mission-control/mission-control-ui:<version>
icr.io/datastax-mission-control/mission-control-dex:<version>
icr.io/datastax-mission-control/k8ssandra-client:<version>

# Entitled images — pulled from cp.icr.io, requires entitlement key
cp.icr.io/cp/ibm-ds-mission-control/hcd:<version>              # HCD
cp.icr.io/cp/ibm-ds-mission-control/dse-mgmtapi:<version>      # DSE
cp.icr.io/cp/ibm-ds-mission-control/cql-router:<version>
cp.icr.io/cp/ibm-ds-mission-control/cqlsh-pod:<version>

# Operators
cr.k8ssandra.io/k8ssandra/k8ssandra-operator:<version>
docker.io/k8ssandra/cass-operator:<version>

# Other database management APIs (non-entitled OSS/DSE 6.8/6.9 paths)
docker.io/k8ssandra/cass-management-api:<version>-ubi   # Cassandra 4.0, 4.1, 5.0
docker.io/datastax/dse-mgmtapi-6_8:<version>-ubi        # DSE 6.8
docker.io/datastax/dse-mgmtapi-6_9:<version>-ubi        # DSE 6.9

# Observability
docker.io/grafana/grafana:<version>
docker.io/grafana/loki:<version>
docker.io/grafana/mimir:<version>
docker.io/timberio/vector:<version>
docker.io/kiwigrid/k8s-sidecar:<version>
registry.k8s.io/kube-state-metrics/kube-state-metrics:<version>
docker.io/nginxinc/nginx-unprivileged:<version>

# Backup / repair
docker.io/k8ssandra/medusa:<version>
docker.io/thelastpickle/cassandra-reaper:<version>

# Utility images
docker.io/k8ssandra/system-logger:<version>
docker.io/datastax/cass-config-builder:<version>

# Object storage (if using local MinIO for testing)
quay.io/minio/minio:<version>
quay.io/minio/mc:<version>
```

> **Always check the exact image list and tags** for your specific MC version in the release notes:
> 📖 https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

> The `cqlsh-pod` image backing the in-UI CQL console is not open source and is not available on Docker Hub — it is only obtainable from `cp.icr.io` with your entitlement key.

---

## Phase 1C — Mirror the Helm chart (optional)

If the air-gapped cluster has no route to `icr.io` at all, pull and repackage the chart, or push it into your private OCI-compatible registry:

```bash
helm pull oci://icr.io/mission-control-helm/mission-control --version $MC_VERSION
# Extract, then push to your private OCI registry if it supports OCI artifacts, e.g.:
# helm push mission-control-<version>.tgz oci://YOUR_REGISTRY/YOUR_NS
```

If the cluster can reach `icr.io` (but not the entitled `cp.icr.io` image content, or other public registries), you can `helm install`/`helm upgrade` directly from `oci://icr.io/mission-control-helm/mission-control` and only override the image coordinates below.

---

## Phase 2 — Helm Air-Gap Install

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

Create a `values-airgap.yaml` that overrides every image coordinate to point to your private registry.

### Image override strategies (choose one)

**Strategy A — Global override (v1.15.0+, recommended):**
Sets `global.imageConfig.overrides.registry` to redirect all images at once.

```yaml
global:
  imageConfig:
    overrides:
      registry: "YOUR_REGISTRY:PORT"
      pullSecrets:
        - name: mc-pull-secret
    defaults:
      registry: "YOUR_REGISTRY:PORT"
      pullPolicy: IfNotPresent
    types:
      hcd:
        registry: "YOUR_REGISTRY:PORT"
        repository: "YOUR_NS"
        name: "hcd"
        suffix: "-ubi"
      cassandra:
        repository: "YOUR_NS"
        name: "cass-management-api"
        suffix: "-ubi"
      dse:
        repository: "YOUR_NS"
        name: "dse-mgmtapi-6_8"
        suffix: "-ubi"
```

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

**Strategy B — Component-by-component (all MC versions):**

```yaml
controlPlane: true
disableCertManagerCheck: true

# Main operator image
image:
  registry: YOUR_REGISTRY:PORT
  repository: YOUR_NS/mission-control
  tag: v1.20.1
  pullPolicy: IfNotPresent

# Legacy image config (pre-1.15 or when you need per-component control)
imageConfigs:
  registryOverride: YOUR_REGISTRY:PORT
  reaper:
    repository: YOUR_NS/cassandra-reaper
  medusa:
    repository: YOUR_NS/medusa

# Pull secret for all components
global:
  imagePullSecrets:
    - name: mc-pull-secret

# MC client and CRD patcher
client:
  image:
    registry: YOUR_REGISTRY:PORT
    repository: YOUR_NS/k8ssandra-client
    tag: v0.8.13

crdPatchJob:
  image:
    registry: YOUR_REGISTRY:PORT
    repository: YOUR_NS/kubectl
    tag: 1.30.1

# UI
ui:
  enabled: true
  image:
    registry: YOUR_REGISTRY:PORT
    repository: YOUR_NS/mission-control-ui
    tag: v1.20.1

# Dex
dex:
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/mission-control-dex
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH"
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# Operators
k8ssandra-operator:
  image:
    registry: YOUR_REGISTRY:PORT
  cass-operator:
    image:
      registry: YOUR_REGISTRY:PORT
    imageConfig:
      systemLogger: YOUR_REGISTRY:PORT/YOUR_NS/system-logger:v1.30.2
      configBuilder: YOUR_REGISTRY:PORT/YOUR_NS/cass-config-builder:1.0-ubi
      k8ssandraClient: YOUR_REGISTRY:PORT/YOUR_NS/k8ssandra-client:v0.8.13

# Observability
grafana:
  enabled: true
  imageRegistry: YOUR_REGISTRY:PORT
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/grafana
  sidecar:
    image:
      repository: YOUR_REGISTRY:PORT/YOUR_NS/k8s-sidecar
  downloadDashboardsImage:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/curl
  initChownData:
    image:
      repository: YOUR_REGISTRY:PORT/YOUR_NS

loki:
  global:
    image:
      registry: YOUR_REGISTRY:PORT
  sidecar:
    image:
      repository: YOUR_REGISTRY:PORT/YOUR_NS/k8s-sidecar
  minio:
    image:
      repository: YOUR_REGISTRY:PORT/YOUR_NS/minio
    mcImage:
      repository: YOUR_REGISTRY:PORT/YOUR_NS/mc

mimir:
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/mimir
  nginx:
    image:
      registry: YOUR_REGISTRY:PORT
  gateway:
    nginx:
      image:
        registry: YOUR_REGISTRY:PORT

agent:
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/vector
aggregator:
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/vector

kube-state-metrics:
  image:
    registry: YOUR_REGISTRY:PORT

# Replicated SDK no longer applies — omit the `replicated` key entirely, or set:
replicated:
  enabled: false
```

### Create image pull secret and install

```bash
# Create the pull secret for your private registry
kubectl create namespace mission-control
kubectl create secret docker-registry mc-pull-secret \
  --namespace mission-control \
  --docker-server=YOUR_REGISTRY:PORT \
  --docker-username=REGISTRY_USER \
  --docker-password=REGISTRY_PASSWORD

# Dry-run first to validate all image references
helm install mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace mission-control \
  --dry-run --debug \
  -f values-airgap.yaml 2>&1 | grep "image:"

# Install (swap the OCI source for your mirrored chart registry if the cluster
# cannot reach icr.io at all — see Phase 1C)
helm install mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace mission-control \
  -f values-airgap.yaml

# Verify all images pull from your private registry (none should reference
# docker.io, icr.io, or cp.icr.io directly)
kubectl get deployments -n mission-control \
  -o jsonpath='{range .items[*]}{.spec.template.spec.containers[*].image}{"\n"}{end}'
```

### Troubleshoot image registry overrides

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

```bash
# List all container images in use
kubectl get deployments -n mission-control \
  -o jsonpath='{.items[*].spec.template.spec.containers[*].image}'

# Check init containers too (some components use them)
kubectl get deployments -n mission-control \
  -o jsonpath='{.items[*].spec.template.spec.initContainers[*].image}'

# Check pull events for auth errors
kubectl get events -n mission-control | grep -i "pull\|image\|auth"

# Test a registry connection from within the cluster
kubectl run test-pull --image=YOUR_REGISTRY:PORT/YOUR_NS/mission-control:v1.20.1 \
  --restart=Never --command -- echo ok
kubectl describe pod test-pull
kubectl delete pod test-pull
```

Common issues:
- **Mixed registry patterns**: Some sub-charts expect full image paths, others expect separate registry/repository values
- **Missing namespace**: All image paths must include the registry namespace in air-gap environments
- **Version skew**: Image tags must exactly match the expected versions for the MC release
- **Missing pull secret**: Ensure `mc-pull-secret` exists in every namespace where pods are scheduled
- **`cqlsh-pod` image missing**: This image is not open source and only available from `cp.icr.io` with the entitlement key — mirror it explicitly, it will not appear in generic Docker Hub scans

---

## Air-Gap Support Bundle

Generate a support bundle for troubleshooting when internet access is unavailable:

```bash
# On an internet-connected machine first: download the spec
curl -o support-bundle-spec.yaml https://kots.io

# Copy support-bundle-spec.yaml to the air-gapped server, then:
kubectl support-bundle ./support-bundle-spec.yaml
```

📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

---

## Next Steps

- Cloud-specific storage and ingress values → `04-cloud-config.md`
- TLS and auth → `05-security.md`
- Deploy HCD/DSE cluster → `06-cluster-ops.md`
- Migrating an existing air-gapped Replicated/KOTS install → `02-install-online.md` (Migrate an existing Replicated installation to ICR)
