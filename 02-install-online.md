# 02 — Online Installation: cert-manager, Helm, KOTS, OpenShift, Separate Cluster Resources

This file covers all **online** (internet-connected) installation paths.  
For air-gapped installs, see `03-install-airgap.md`.  
For cloud-specific `values.yaml` content (EKS/AKS/GKE/OCP), see `04-cloud-config.md`.

📖 Source: https://docs.datastax.com/en/mission-control/install/choose-an-installation-method.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

---

## Installation method comparison

| Method | Best for | Pros | Cons |
|--------|----------|------|------|
| **Helm** (recommended) | GitOps, CI/CD, fine-grained control | Version-controlled config, CLI-managed, GitOps-friendly | Requires Helm knowledge |
| **KOTS** | Guided setup, automated upgrades via UI | Web UI, guided wizard, auto-upgrade | Less customizable, web interface required |
| **Helm + separate cluster resources** | Multi-team: cluster-admin and app team separated | Cluster-scoped resources managed independently | More complex coordination |
| **OpenShift** | OCP environments | Native OCP security, Routes auto-created | OCP-specific knowledge required |

📖 Source: https://docs.datastax.com/en/mission-control/install/choose-an-installation-method.html

---

## Step A — Install cert-manager (ALL paths, required first)

📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html

cert-manager must be installed and running **before** Mission Control.

> ⚠️ The release **must** be named `cert-manager` and installed in the `cert-manager` namespace. Any other name causes `"The deployment cert-manager was not found"` errors in KOTS.

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

# Verify
kubectl get pods -n cert-manager
# All 3 pods (cert-manager, cainjector, webhook) must be Running
```

> ⚠️ The `--enable-certificate-owner-ref=true` flag is **critical**: it ensures certificate Secrets are automatically deleted when the Certificate resource is removed (prevents orphaned secrets after cluster deletion).

### OpenShift: use OperatorHub instead of Helm

For OCP 1.18.5+, cert-manager must come from the Red Hat OpenShift cert-manager Operator:

1. OCP web console → **Operators → OperatorHub**
2. Search: `cert-manager Operator for Red Hat OpenShift`
3. Select, click **Install**, follow wizard

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

> ⚠️ The OCP cert-manager Operator does **not** auto-delete cert secrets on Certificate deletion. You must clean them up manually with `oc delete secret SECRET_NAME -n NAMESPACE`.

---

## Step B — Prepare the Kubernetes cluster

```bash
# Confirm Kubernetes version
kubectl version --short

# Confirm node labels (must be done first — see 01-preflight.md)
kubectl get nodes --show-labels | grep mission-control

# Confirm StorageClass has WaitForFirstConsumer
kubectl get storageclass
kubectl describe storageclass YOUR_CLASS | grep VolumeBindingMode
# Must show: WaitForFirstConsumer
```

---

## Path 1 — Helm (Recommended)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

Your `LICENSE_ID` is used as **both** the Helm registry username and password.

```bash
# 1. Log in to the Helm OCI registry
helm registry login registry.replicated.com \
  --username 'YOUR_LICENSE_ID' \
  --password 'YOUR_LICENSE_ID'

# 2. Create your values.yaml (see 04-cloud-config.md for cloud-specific values)

# 3. Install Mission Control
helm install mc-release \
  oci://registry.replicated.com/mission-control/mission-control \
  --namespace mission-control \
  --create-namespace \
  -f values.yaml

# 4. Watch pods come up
kubectl get pods -n mission-control -w
```

> ⚠️ **Release name must NOT contain `mission-control`** — this causes internal naming conflicts. Use `mc-release`, `mc-prod`, etc.  
> ⚠️ **Data plane release name must exactly match the control plane release name.**

### Minimal `values.yaml` (control plane)

```yaml
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

### Generate a bcrypt password hash
```bash
echo YOUR_PASSWORD | htpasswd -BinC 10 admin | cut -d: -f2
```

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

## Path 2 — KOTS

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc.html

```bash
# 1. Install the KOTS kubectl plugin (version 1.105+)
curl https://kots.io/install | bash
kubectl kots version   # verify

# 2. Install Mission Control via KOTS
kubectl kots install mission-control \
  --namespace mission-control

# Answer the prompts; the CLI will provide a URL for the KOTS admin console
# RECORD the admin password you set — it cannot be recovered

# 3. If port-forward session drops, restart it:
kubectl kots admin-console -n mission-control
# Open: http://localhost:8800
```

### In the KOTS web UI

1. **Upload license file** when prompted
2. **Set admin user + password** — ⚠️ **REQUIRED since v1.9.0** — preflight fails without this
   ```bash
   # Generate hash:
   echo YOUR_PASSWORD | htpasswd -BinC 10 admin | cut -d: -f2
   ```
3. **Deployment Mode**: Control Plane (first install) or Data Plane (additional regions)
4. **Observability Storage**: Choose S3 / GCS / Azure Blob — configure **two separate buckets** (metrics ≠ logs)
5. **StorageClass**: Ensure `WaitForFirstConsumer` class is selected
6. Click **Run preflight checks** → resolve any failures → **Deploy**

> ⚠️ KOTS Admin Console (port 8800) and the Mission Control UI are **separate authentication systems**. KOTS manages the installation; MC UI manages database operations.

### EKS prerequisite for KOTS
```bash
# KOTS requires a default StorageClass before install
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Change the MC UI password via KOTS
```bash
kubectl kots admin-console -n mission-control
# Navigate to: Authentication → Static Credentials → update Password Hash field
# Then restart Dex to apply:
kubectl rollout restart deployment -n mission-control -l app.kubernetes.io/name=dex
```

---

## Path 3 — Helm with Separate Cluster Resources

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm-separate-cluster-resources.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-cluster-level-resources-separately.html

Use when cluster-scoped resources (CRDs, ClusterRoles) must be managed by a different team than the application deployment.

```bash
# Step 1 — cluster-admin installs cluster-scoped resources first
helm registry login registry.replicated.com \
  --username 'YOUR_LICENSE_ID' \
  --password 'YOUR_LICENSE_ID'

helm install mc-cluster-resources \
  oci://registry.replicated.com/mission-control/mission-control-cluster-resources \
  --namespace mission-control \
  --create-namespace

# Step 2 — app team installs MC (no cluster-admin needed)
helm install mc-release \
  oci://registry.replicated.com/mission-control/mission-control \
  --namespace mission-control \
  --set global.clusterScopedResources=false \
  --skip-crds \
  -f values.yaml
```

> ⚠️ **Upgrade order:** Always upgrade `mc-cluster-resources` first, then `mc-release`.

For narrow-scope RBAC so app team can run `helm upgrade` without cluster-admin:  
📖 https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html

---

## Path 4 — OpenShift (Helm recommended)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

### Prerequisites
- OCP 4.8+
- `oc` CLI installed
- cert-manager installed from OperatorHub (not Helm) — see Step A above

### Grant SCC permissions before install

```bash
# Grant nonroot-v2 SCC to all MC service accounts
for SA in loki mission-control mission-control-agent mission-control-aggregator \
  mission-control-cass-operator mission-control-dex \
  mission-control-k8ssandra-operator mission-control-kube-state-metrics \
  mission-control-mimir; do
  oc adm policy add-scc-to-user nonroot-v2 -z $SA -n mission-control
done
```

### Get your OCP cluster domain (needed for ingress)
```bash
oc get ingress.config.openshift.io cluster -o jsonpath='{.spec.domain}'
# Example: apps.myocp.example.com
```

### Install with OCP-specific values
```bash
helm registry login registry.replicated.com \
  --username 'YOUR_LICENSE_ID' \
  --password 'YOUR_LICENSE_ID'

helm install mc-release \
  oci://registry.replicated.com/mission-control/mission-control \
  --namespace mission-control \
  --create-namespace \
  -f values-openshift.yaml \
  --timeout 15m
```

For the required OCP-specific values (DNS resolver, no HAProxy ingress, regionDomain), see `04-cloud-config.md`.

> ⚠️ OCP silently fails to collect metrics/logs if DNS is not configured correctly for Loki and Mimir. See `04-cloud-config.md` → OpenShift section for the exact `dnsNamespace` and resolver values.

### Verify SCC assignment after install
```bash
oc get pod mc-release-k8ssandra-operator-PODID \
  -o jsonpath='{.metadata.annotations.openshift\.io/scc}'
# Should show: nonroot-v2 (or restricted-v2 if you configured that)
```

### OCP: restricted-v2 SCC (optional, higher security)
If your organization requires `restricted-v2` with dynamically assigned UIDs:
```bash
# Check your namespace UID range
oc get namespace mission-control \
  -o jsonpath='{.metadata.annotations.openshift\.io/sa\.scc\.uid-range}'

# Download and apply the restricted-v2 override file from docs, then:
helm upgrade --install mc-release \
  oci://registry.replicated.com/mission-control/mission-control \
  --namespace mission-control \
  -f values.yaml \
  -f openshift-overrides-restricted-scc.yaml \
  --timeout 15m
```

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

---

## Step C — Verify Installation

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
