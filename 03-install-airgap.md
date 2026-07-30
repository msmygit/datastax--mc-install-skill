# 03 — Air-Gap / Offline Installation

This file covers fully offline (air-gapped) installations where the target cluster has **no internet access**.

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html (Airgapped section)  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html (Air-gap Helm section)  
📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

---

## Overview: two distinct phases

**Phase 1 — On an internet-connected machine:**
1. Obtain airgap-enabled license from IBM Support
2. Download the MC airgap bundle
3. Mirror all container images to your private registry

**Phase 2 — On the air-gapped cluster:**
4. Install cert-manager from mirrored images
5. Install MC via KOTS (using the airgap bundle) or Helm (using an all-overrides `values.yaml`)

---

## Phase 1A — License check

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html

> ⚠️ The airgap entitlement must be **explicitly activated** on your license by IBM Support.  
> Symptom of missing entitlement: During KOTS install the registry screen does not appear. Contact IBM Support to enable airgap on your license.

---

## Phase 1B — Download the Airgap Bundle

Run this on a machine with internet access:

```bash
# Download the airgap bundle (~6 GB or more — ensure disk space)
curl -f "https://replicated.app/embedded/mission-control/stable?airgap=true" \
  -H "Authorization: YOUR_LICENSE_ID" \
  -o mission-control-stable.tgz

# The bundle contains:
#   mission-control          MC installer binary
#   license.yaml             Your license file
#   mission-control.airgap   The image + config bundle
```

To download a specific version:
```bash
curl -f "https://replicated.app/embedded/mission-control/stable/v1.19.1?airgap=true" \
  -H "Authorization: YOUR_LICENSE_ID" \
  -o mission-control-v1.19.1.tgz
```

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html

---

## Phase 1C — Mirror Images to Your Private Registry

You have two options: **KOTS CLI bulk push** (easier) or **Skopeo** (per-image, granular control).

### Option 1: KOTS CLI bulk push (recommended for KOTS installs)

```bash
# Extract just the airgap bundle
tar xvzf mission-control-stable.tgz mission-control.airgap

# Push all images from the bundle to your private registry at once
kubectl kots admin-console push-images \
  ./mission-control.airgap \
  YOUR_REGISTRY/YOUR_NAMESPACE \
  --registry-username REGISTRY_USER \
  --registry-password REGISTRY_PASSWORD
```

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

### Option 2: Skopeo (recommended for Helm installs — precise per-image control)

📖 Source: https://docs.datastax.com/en/mission-control/administration/manage-containers.html

```bash
# Install Skopeo (package manager or: https://github.com/containers/skopeo)

# Log in to your private registry
skopeo login YOUR_PRIVATE_REGISTRY

# Copy each required image (repeat for all images listed below)
skopeo copy \
  docker://docker.io/datastax/mission-control:v1.19.1 \
  docker://YOUR_REGISTRY/YOUR_NS/mission-control:v1.19.1

# Verify a copy succeeded
skopeo inspect docker://YOUR_REGISTRY/YOUR_NS/mission-control:v1.19.1
```

### Core images to mirror (update tags per your MC version from release notes)

📖 Release notes with exact image tags: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

```
# Mission Control platform
docker.io/datastax/mission-control:<version>
docker.io/datastax/mission-control-ui:<version>
docker.io/datastax/mission-control-dex:<version>

# Operators
cr.k8ssandra.io/k8ssandra/k8ssandra-operator:<version>
docker.io/k8ssandra/cass-operator:<version>
docker.io/k8ssandra/k8ssandra-client:<version>

# Database management APIs
docker.io/k8ssandra/cass-management-api:<version>-ubi   # Cassandra 4.0, 4.1, 5.0
docker.io/datastax/dse-mgmtapi-6_8:<version>-ubi        # DSE 6.8
docker.io/datastax/dse-mgmtapi-6_9:<version>-ubi        # DSE 6.9
# HCD images are from proxy.replicated.com — contact IBM Support for access

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
docker.io/stargateio/data-api:<version>

# Object storage (if using local MinIO for testing)
quay.io/minio/minio:<version>
quay.io/minio/mc:<version>

# Replicated SDK (set replicated.enabled: false in air-gap Helm values)
docker.io/replicated/replicated-sdk:<version>
```

> **Always check the exact image list and tags** for your specific MC version in the release notes:  
> 📖 https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

---

## Phase 2A — KOTS Air-Gap Install

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html

```bash
# Push KOTS admin images first (do this on the air-gapped machine, using the kotsadm.tar.gz bundle)
kubectl kots admin-console push-images ./kotsadm.tar.gz YOUR_REGISTRY \
  --registry-username RW_USER \
  --registry-password RW_PASS

# Install KOTS pointing to your private registry
kubectl kots install mission-control \
  --namespace mission-control \
  --kotsadm-registry YOUR_REGISTRY \
  --kotsadm-namespace YOUR_NAMESPACE \
  --registry-username RO_USER \
  --registry-password RO_PASS

# Forward the admin console
kubectl kots admin-console -n mission-control
# Open: http://localhost:8800
```

### In the KOTS web UI (air-gap specific steps)

1. Upload your `license.yaml`
2. The **registry credentials screen appears** (proof your license has airgap) — enter your registry hostname and credentials
3. Upload `mission-control.airgap` bundle — KOTS loads images from the bundle into your registry
4. Set **admin user + password** (required since v1.9.0)
5. Set **Deployment Mode**: Control Plane or Data Plane
6. Configure observability storage (two separate buckets)
7. Run preflight checks → Deploy

---

## Phase 2B — Helm Air-Gap Install

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
  tag: v1.19.1
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
    tag: v1.19.1

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
  imageRegistry: YOUR_REGISTRY:PORT
  image:
    repository: YOUR_REGISTRY:PORT/YOUR_NS/grafana
  sidecar:
    image:
      repository: YOUR_REGISTRY:PORT/YOUR_NS/k8s-sidecar

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

# Disable Replicated SDK in air-gap mode
replicated:
  enabled: false
  images:
    replicated-sdk: YOUR_REGISTRY:PORT/YOUR_NS/replicated-sdk:1.8.0
```

### Create image pull secret and install

```bash
# Create the pull secret
kubectl create namespace mission-control
kubectl create secret docker-registry mc-pull-secret \
  --namespace mission-control \
  --docker-server=YOUR_REGISTRY:PORT \
  --docker-username=REGISTRY_USER \
  --docker-password=REGISTRY_PASSWORD

# Dry-run first to validate all image references
helm install mc-release \
  oci://YOUR_REGISTRY:PORT/mission-control/mission-control \
  --namespace mission-control \
  --dry-run --debug \
  -f values-airgap.yaml 2>&1 | grep "image:"

# Install
helm install mc-release \
  oci://YOUR_REGISTRY:PORT/mission-control/mission-control \
  --namespace mission-control \
  -f values-airgap.yaml

# Verify all images pull from your registry (none should reference docker.io or replicated.com)
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
kubectl run test-pull --image=YOUR_REGISTRY:PORT/YOUR_NS/mission-control:v1.19.1 \
  --restart=Never --command -- echo ok
kubectl describe pod test-pull
kubectl delete pod test-pull
```

Common issues:
- **Mixed registry patterns**: Some sub-charts expect full image paths, others expect separate registry/repository values
- **Missing namespace**: All image paths must include the registry namespace in air-gap environments
- **Version skew**: Image tags must exactly match the expected versions for the MC release
- **Missing pull secret**: Ensure `mc-pull-secret` exists in every namespace where pods are scheduled

---

## Air-gap Support Bundle

When internet access is unavailable, generate a support bundle from a previously downloaded spec:

```bash
# On an internet-connected machine first:
curl -o support-bundle-spec.yaml https://kots.io \
  -H 'User-agent:Replicated_Troubleshoot/v1beta1'

# Copy support-bundle-spec.yaml to air-gapped server, then:
kubectl support-bundle ./support-bundle-spec.yaml
```

📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

---

## Next Steps

- Cloud-specific storage and ingress values → `04-cloud-config.md`
- TLS and auth → `05-security.md`
- Deploy HCD/DSE cluster → `06-cluster-ops.md`
