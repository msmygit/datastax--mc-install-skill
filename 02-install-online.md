# 02 — Online Installation: cert-manager, Helm (ICR), OpenShift, Separate Cluster Resources, Migrating from Replicated

This file covers all **online** (internet-connected) installation paths.
For air-gapped installs, see `03-install-airgap.md`.
For cloud-specific `values.yaml` content (EKS/AKS/GKE/OCP), see `04-cloud-config.md`.

📖 Source: https://docs.datastax.com/en/mission-control/install/choose-an-installation-method.html
📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm-separate-cluster-resources.html
📖 Source: https://docs.datastax.com/en/mission-control/install/migrate-replicated-to-icr.html

> ⚠️ **Replicated (KOTS) is no longer supported.** As of Mission Control 1.20.1, Replicated is deprecated, and KOTS support was removed entirely in later versions. Helm from the IBM Container Registry (ICR) is the only supported install method. If the user has an existing Replicated/KOTS install, go straight to [Migrate an existing Replicated installation to ICR](#migrate-an-existing-replicated-installation-to-icr) below — do **not** reinstall from scratch.

---

## Installation method comparison

| Method | Best for | Pros | Cons |
|--------|----------|------|------|
| **Helm** (recommended) | GitOps, CI/CD, fine-grained control | Version-controlled config, CLI-managed, GitOps-friendly | Requires Helm knowledge |
| **Helm + separate cluster resources** | Multi-team: cluster-admin and app team separated | Cluster-scoped resources managed independently | More complex coordination |
| **OpenShift** | OCP environments | Native OCP security, Routes auto-created | OCP-specific knowledge required |

📖 Source: https://docs.datastax.com/en/mission-control/install/choose-an-installation-method.html

---

## Step A — Install cert-manager (ALL paths, required first)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

cert-manager must be installed and running **before** Mission Control.

```bash
# Install cert-manager CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.1/cert-manager.crds.yaml

# Install cert-manager (DataStax recommends v1.16.1)
helm repo add jetstack https://charts.jetstack.io --force-update
helm repo update
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.16.1 \
  --set 'extraArgs[0]=--enable-certificate-owner-ref=true'

# Wait for all three rollouts to complete
kubectl rollout status deployment cert-manager -n cert-manager
kubectl rollout status deployment cert-manager-cainjector -n cert-manager
kubectl rollout status deployment cert-manager-webhook -n cert-manager
```

> ⚠️ The `--enable-certificate-owner-ref=true` flag is **critical**: it ensures certificate Secrets are automatically deleted when the Certificate resource is removed (prevents orphaned secrets after cluster deletion).

### OpenShift: use OperatorHub instead of Helm

For OCP environments (1.18.5+), cert-manager must come from the Red Hat OpenShift cert-manager Operator:

1. OCP web console → **Operators → OperatorHub**
2. Search: `cert-manager Operator for Red Hat OpenShift`
3. Select, click **Install**, follow wizard

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

> ⚠️ The OCP cert-manager Operator does **not** auto-delete cert secrets on Certificate deletion. You must clean them up manually with `oc delete secret SECRET_NAME -n NAMESPACE`.

---

## Step B — Set environment variables and prepare the cluster

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

```bash
export MC_NAMESPACE=mission-control
export MC_RELEASE_NAME=mc-release            # must NOT contain the string "mission-control"
export MC_VERSION=v1.20.1                    # the MC Helm chart version to install
export PULL_SECRET_NAME=${MC_RELEASE_NAME}-registry   # default pull secret name pattern
export ENTITLEMENT_KEY=YOUR_IBM_ENTITLEMENT_KEY
export IBM_EMAIL=you@example.com
```

> Environment variables don't persist across terminal sessions — re-export them if you open a new shell.

```bash
# Create the namespace (pull secret and Helm release must live in the same namespace)
kubectl create namespace $MC_NAMESPACE
kubectl get namespace $MC_NAMESPACE

# Confirm Kubernetes version and node labels (see 01-preflight.md)
kubectl version --short
kubectl get nodes --show-labels | grep mission-control

# Confirm StorageClass has WaitForFirstConsumer
kubectl get storageclass
kubectl describe storageclass YOUR_CLASS | grep VolumeBindingMode
```

---

## Step C — Create the ICR image pull secret

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

Entitled images (`hcd`, `dse-mgmtapi`, `cql-router`, `cqlsh-pod`) are pulled from `cp.icr.io` using a Kubernetes `docker-registry` secret. The username is always the fixed value `cp`; the password is your IBM entitlement key.

```bash
kubectl create secret docker-registry $PULL_SECRET_NAME \
  --docker-server=cp.icr.io \
  --docker-username=cp \
  --docker-password=$ENTITLEMENT_KEY \
  --namespace=$MC_NAMESPACE

# Verify
kubectl get secret $PULL_SECRET_NAME -n $MC_NAMESPACE
kubectl get secret $PULL_SECRET_NAME -n $MC_NAMESPACE \
  -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
```

You only need to create this secret **once**, in the operator namespace. Mission Control automatically replicates every pull secret listed in `global.imageConfig` into project namespaces as clusters are provisioned — you don't need to create it manually in every cluster namespace.

If you use multiple clusters (e.g., separate control plane and data plane clusters), repeat this command against each cluster's `kubeconfig` context.

> ⚠️ **Authentication failures?** `--docker-username` must be exactly `cp` (lowercase); the server must be `cp.icr.io` (not `icr.io` directly); check for leading/trailing whitespace in the entitlement key; regenerate the key at https://myibm.ibm.com/products-services/containerlibrary if it's expired.

---

## Path 1 — Helm (Recommended)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

The chart is published to the OCI registry at `oci://icr.io/mission-control-helm/mission-control`. No `helm repo add` is required.

```bash
# Optional: pull the chart for offline inspection
helm pull oci://icr.io/mission-control-helm/mission-control --version $MC_VERSION

# Optional: inspect all available chart values before installing
helm show values oci://icr.io/mission-control-helm/mission-control --version $MC_VERSION
```

### Minimal `overrides.yaml`

An `overrides.yaml` is required — the chart validates several components at install time and a bare install without values fails. At minimum, configure Loki storage (or disable it) and Dex with at least one auth connector.

```yaml
# overrides.yaml
controlPlane: true
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

allowOperatorsOnDatabaseNodes: false

client:
  manageCrds: true

ui:
  enabled: true
  baseUrl: ""        # Set this if using Ingress or OIDC: https://mc.example.com
  https:
    enabled: true

dex:
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH_HERE"   # generate: echo pass | htpasswd -BinC 10 admin | cut -d: -f2
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# Loki and Mimir MUST use separate buckets — see 04-cloud-config.md for storage backend details
loki:
  enabled: true

mimir:
  enabled: true

agent:
  enabled: true

aggregator:
  enabled: true
```

If your pull secret name differs from the default (`${MC_RELEASE_NAME}-registry`), also add:

```yaml
global:
  imageConfig:
    defaults:
      pullSecrets:
        - "YOUR_SECRET_NAME"
```

### Generate a bcrypt password hash
```bash
echo YOUR_PASSWORD | htpasswd -BinC 10 admin | cut -d: -f2
```

### Install

```bash
helm install $MC_RELEASE_NAME \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace $MC_NAMESPACE \
  --create-namespace \
  -f overrides.yaml

# Watch pods come up
kubectl get pods -n $MC_NAMESPACE -w

# Check the Helm release status
helm status $MC_RELEASE_NAME -n $MC_NAMESPACE
```

> ⚠️ **Release name must NOT contain `mission-control`** — this causes internal naming conflicts. Use `mc-release`, `mc-prod`, etc.
> ⚠️ **Data plane release name must exactly match the control plane release name.**

### Post-install: access the UI
```bash
# Option 1: NodePort (default port 30880)
# Navigate to: https://NODE_IP:30880

# Option 2: Port-forward (dev/test)
kubectl port-forward svc/mc-release-mission-control-ui 8443:443 -n mission-control
# Navigate to: https://localhost:8443

# Option 3: Ingress (production) — configure in values.yaml, see 04-cloud-config.md
```

📖 Source: https://docs.datastax.com/en/mission-control/install/access-ui.html

---

## Path 2 — Helm with Separate Cluster Resources

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm-separate-cluster-resources.html
📖 Source: https://docs.datastax.com/en/mission-control/install/install-cluster-level-resources-separately.html

Use when cluster-scoped resources (CRDs, ClusterRoles) must be managed by a different team than the application deployment.

```bash
# Step 1 — cluster-admin installs cluster-scoped resources first
helm repo add mc-cluster-obj https://helm.k8ssandra.io/mission-control
helm repo update

helm install mc-cluster-obj mc-cluster-obj/mc-cluster-obj \
  -n $MC_NAMESPACE \
  --create-namespace \
  --set targetReleaseName=$MC_RELEASE_NAME \
  --set targetNamespace=$MC_NAMESPACE

# Step 2 — app team installs MC (no cluster-admin needed)
helm install $MC_RELEASE_NAME \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace $MC_NAMESPACE \
  --set global.clusterScopedResources=false \
  --set dex.rbac.createClusterScoped=false \
  --set kube-state-metrics.rbac.create=false \
  --skip-crds \
  --no-hooks \
  -f overrides.yaml
```

> ⚠️ **Upgrade order:** Always upgrade `mc-cluster-obj` first, then the main release, using the same flags.

For narrow-scope RBAC so the app team can run `helm upgrade` without cluster-admin:
📖 https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html

---

## Path 3 — OpenShift (Helm)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

### Prerequisites
- OCP 4.8+
- `oc` CLI installed
- cert-manager installed from OperatorHub (not Helm) — see Step A above

### Grant SCC permissions before install

```bash
for SA in loki mission-control mission-control-agent mission-control-aggregator \
  mission-control-cass-operator mission-control-dex \
  mission-control-k8ssandra-operator mission-control-kube-state-metrics \
  mission-control-mimir; do
  oc adm policy add-scc-to-user nonroot-v2 -z $SA -n mission-control
done
```

If your organization requires `restricted-v2` with dynamically assigned UIDs, see `05-security.md` and the OpenShift-specific overrides file referenced there.

### Configure OpenShift DNS (required for Loki/Mimir)

OpenShift uses `openshift-dns`/`dns-default` instead of `kube-dns`. Without this, Loki and Mimir silently fail to collect logs/metrics:

```yaml
loki:
  global:
    clusterDomain: cluster.local
    dnsNamespace: openshift-dns
    dnsService: dns-default
mimir:
  gateway:
    nginx:
      config:
        resolver: dns-default.openshift-dns.svc.cluster.local
  nginx:
    nginxConfig:
      resolver: "dns-default.openshift-dns.svc"
```

### Install with OCP-specific values

```bash
helm install $MC_RELEASE_NAME \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace $MC_NAMESPACE \
  --create-namespace \
  -f values-openshift.yaml \
  --timeout 15m
```

For the required OCP-specific values (DNS resolver, regionDomain, etc.), see `04-cloud-config.md`.

### Verify SCC assignment after install
```bash
oc get pod mc-release-k8ssandra-operator-PODID \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
# Should show: nonroot-v2 (or restricted-v2 if you configured that)
```

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

---

## Migrate an existing Replicated installation to ICR

📖 Source: https://docs.datastax.com/en/mission-control/install/migrate-replicated-to-icr.html

Replicated is no longer supported. If Mission Control was originally installed via `helm install` from the Replicated proxy registry, or via the KOTS admin console, you must migrate it to ICR **in place** — this switches the image pull source without reinstalling MC or losing data.

> ⚠️ **Schedule a maintenance window.** Restarting `cass-operator` (a required step) reconciles all managed database clusters and restarts database pods to pull images from `cp.icr.io`. Clusters with replication factor ≥3 tolerate this as a rolling restart; lower-RF clusters experience downtime. MC control plane pods also restart briefly during the Helm upgrade regardless of replication factor.

Two variants exist, depending on how MC was originally installed:

### Variant A — Helm-based Replicated install (no KOTS admin console)

1. **Gather:** original release name, MC namespace, all cluster namespaces, IBM entitlement key, the exact currently-installed MC version (you must migrate to the **same version** — upgrade only after migrating), and the original `values.yaml` file(s).
2. **Update the existing pull secret in place** (Replicated named it `${RELEASE_NAME}-registry`):
   ```bash
   kubectl create secret docker-registry ${RELEASE_NAME}-registry \
     --docker-server=cp.icr.io \
     --docker-username=cp \
     --docker-password=${IBM_ENTITLEMENT_KEY} \
     --namespace=${MC_NAMESPACE} \
     --dry-run=client -o yaml | kubectl apply -f -
   ```
3. **Create `helm-overrides.yaml`** redirecting all image references from the Replicated proxy to `icr.io`/`cp.icr.io` (see the full sample below).
4. **Run `helm upgrade`** with the original values file(s) followed by `helm-overrides.yaml` (later files win):
   ```bash
   helm upgrade ${RELEASE_NAME} oci://icr.io/mission-control-helm/mission-control \
     --version ${TARGET_VERSION} \
     -n ${MC_NAMESPACE} \
     -f ${ORIGINAL_VALUES_FILE} \
     -f helm-overrides.yaml \
     --server-side true
   ```
5. **Restart `cass-operator`** (below) so `CassandraDatacenter` resources pick up the new `cp.icr.io` image config.

### Variant B — KOTS-based install

KOTS owns the Helm release and renders `repl{{ }}` templates into concrete values at deploy time. Migration is three phases: **extract** the effective values, **hand off** by scaling KOTS to zero, then **upgrade** with ICR overrides.

1. **Export the values KOTS currently has deployed:**
   ```bash
   helm get values ${RELEASE_NAME} -n ${MC_NAMESPACE} -o yaml > kots-current-values.yaml
   cp kots-current-values.yaml my-values.yaml
   ```
   Review it for deployment mode, TLS settings, Dex config, observability storage, and ingress — all of it must carry forward.
2. **Update the pull secret in place** (same command as Variant A, step 2).
3. **Scale down KOTS** so it stops reconciling over your upcoming `helm upgrade`:
   ```bash
   kubectl scale deployment kotsadm kotsadm-operator -n ${KOTSADM_NAMESPACE} --replicas=0
   kubectl scale statefulset kotsadm-minio kotsadm-rqlite -n ${KOTSADM_NAMESPACE} --replicas=0
   kubectl get pods -n ${KOTSADM_NAMESPACE} | grep kotsadm   # confirm none Running
   ```
4. **Add ICR overrides to `my-values.yaml`** (same block as below) and run `helm upgrade`:
   ```bash
   helm upgrade ${RELEASE_NAME} oci://icr.io/mission-control-helm/mission-control \
     --version ${TARGET_VERSION} \
     -n ${MC_NAMESPACE} \
     -f my-values.yaml \
     --server-side true
   ```
5. **Restart `cass-operator`** (below).
6. **Remove KOTS entirely** once verified stable — this is irreversible. Delete resources directly with `kubectl`, **not** `kubectl kots remove` (that command times out because `kotsadm` is already scaled to zero):
   ```bash
   kubectl delete deployment kotsadm kotsadm-operator -n ${KOTSADM_NAMESPACE} --ignore-not-found
   kubectl delete statefulset kotsadm-minio kotsadm-rqlite -n ${KOTSADM_NAMESPACE} --ignore-not-found
   kubectl delete service kotsadm kotsadm-http -n ${KOTSADM_NAMESPACE} --ignore-not-found
   kubectl delete pvc -n ${KOTSADM_NAMESPACE} -l app=kotsadm-minio --ignore-not-found
   kubectl delete pvc -n ${KOTSADM_NAMESPACE} -l app=kotsadm-rqlite --ignore-not-found
   kubectl delete secret    -n ${KOTSADM_NAMESPACE} -l kots.io/kotsadm=true --ignore-not-found
   kubectl delete configmap -n ${KOTSADM_NAMESPACE} -l kots.io/kotsadm=true --ignore-not-found
   ```

### ICR overrides block (both variants)

```yaml
# Disable Replicated SDK and KOTS-managed pull secret generation
replicated:
  enabled: false
disablePullSecretsGeneration: true

# MC operator
image:
  registry: icr.io
  repository: datastax-mission-control/mission-control

# MC UI
ui:
  image:
    registry: icr.io
    repository: datastax-mission-control/mission-control-ui

# CRD patcher
client:
  image:
    registry: icr.io
    repository: datastax-mission-control/k8ssandra-client

global:
  imageConfig:
    defaults:
      pullSecrets:
        - PULL_SECRET_NAME   # <RELEASE_NAME>-registry
    images:
      k8ssandra-client:
        registry: icr.io
        repository: datastax-mission-control
        name: k8ssandra-client
      cql-router:                    # entitled — requires cp.icr.io pull secret
        registry: cp.icr.io
        repository: cp/ibm-ds-mission-control
      cqlsh:                         # entitled — requires cp.icr.io pull secret
        registry: cp.icr.io
        repository: cp/ibm-ds-mission-control
    types:
      hcd:                           # entitled — requires cp.icr.io pull secret
        registry: cp.icr.io
        repository: cp/ibm-ds-mission-control
        name: hcd
      dse:                           # entitled — requires cp.icr.io pull secret
        registry: cp.icr.io
        repository: cp/ibm-ds-mission-control
        name: dse-mgmtapi

# MC Dex
dex:
  image:
    repository: icr.io/datastax-mission-control/mission-control-dex
```

### Restart cass-operator (both variants)

```bash
kubectl get deployments -n ${MC_NAMESPACE} | grep cass-operator
kubectl rollout restart deployment/cass-operator -n ${MC_NAMESPACE}
kubectl rollout status deployment/cass-operator -n ${MC_NAMESPACE} --timeout=3m

# Watch and confirm database pods pull from cp.icr.io, per cluster namespace
kubectl get pods -n CLUSTER_NAMESPACE -w
kubectl get pods -n CLUSTER_NAMESPACE \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .spec.containers[*]}{.image}{"\n"}{end}{end}'

# Confirm CassandraDatacenter health
kubectl get cassandradatacenter -A
```

All images should reference `icr.io/datastax-mission-control/…` or `cp.icr.io/cp/ibm-ds-mission-control/…`. If a pod is stuck `ImagePullBackOff`, verify the pull secret contains a valid `cp.icr.io` entry.

---

## Step D — Verify Installation

```bash
# All MC pods should be Running or Completed
kubectl get pods -n mission-control

# Check MC operator logs for errors
kubectl logs -n mission-control deploy/mc-release-mission-control-operator --tail=50

# Check CRDs installed
kubectl get crd | grep datastax

# Access CLI (mcctl)
kubectl exec -it deploy/mc-release-mission-control -n mission-control -- mcctl version
```

📖 Source: https://docs.datastax.com/en/mission-control/install/access-cli.html

---

## Next Steps After Installation

- Configure cloud-specific storage and ingress → `04-cloud-config.md`
- Set up TLS, LDAP, OIDC → `05-security.md`
- Deploy your first HCD/DSE cluster → `06-cluster-ops.md`
- Multi-region setup → `06-cluster-ops.md`
