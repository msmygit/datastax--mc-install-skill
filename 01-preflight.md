# 01 — Pre-Flight: Entitlement, Hardware, Software, Node Labels, Networking & Storage

Every item in this file must be verified **before** running `helm install`.

📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html  
📖 Source: https://docs.datastax.com/en/mission-control/misc/supported-platforms.html

> ⚠️ Replicated/KOTS is no longer a supported install method (deprecated 1.20.1, removed thereafter). All installs — online and air-gapped — use Helm from the IBM Container Registry (ICR). Existing Replicated installs migrate in place; see `02-install-online.md`.  
> 📖 https://docs.datastax.com/en/mission-control/install/migrate-replicated-to-icr.html

---

## 1. IBM Entitlement Key (Mandatory for All Install Paths)

A valid IBM entitlement key is required before any installation can proceed. Unlike the old Replicated `LICENSE_ID`, the entitlement key is tied to your IBMid, not a license file.

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

### Obtain an IBMid and entitlement key
- Go to https://myibm.ibm.com/products-services/containerlibrary (IBM Container Library)
- Sign in with an IBMid (contact your IBM account team or open a ticket at https://www.ibm.com/mysupport if you don't have one)
- Click **Get entitlement key** (or **Copy key** if one already exists)
- The entitlement key is used as the **password** for the `cp.icr.io` image pull secret; the username is always the fixed value `cp`

### Evaluators / Public Preview
- A "Public Preview" license is available — contact your IBM account team or DataStax sales
- Running OSS Cassandra nodes **in addition to** HCD/DSE nodes carries an additional cost; HCD/DSE nodes themselves are included in the license

### Air-gapped entitlement
- The same IBM entitlement key is used for air-gapped installs — it authenticates against `cp.icr.io` when you mirror entitled images (`hcd`, `dse-mgmtapi`, `cql-router`, `cqlsh-pod`) to your private registry with Skopeo
- Regenerate the key at https://myibm.ibm.com/products-services/containerlibrary if it has expired or you receive container authorization errors
- Public MC images (`mission-control`, `mission-control-ui`, `mission-control-dex`, `k8ssandra-client`) are served from the non-entitled `icr.io` registry and do **not** require the entitlement key

---

## 2. Version Requirements

📖 Source: https://docs.datastax.com/en/mission-control/misc/supported-platforms.html

| Component | Required | Notes |
|-----------|----------|-------|
| Kubernetes | **1.21.0+** | **Avoid 1.35.0–1.35.3**: `MaxUnavailableStatefulSet` feature flag causes StatefulSet failures |
| OpenShift | **4.8+** | Uses `oc` CLI; cert-manager installed from OperatorHub, not Helm |
| Helm | **3.14.0–3.18.0** | This range is strict. Verify: `helm version --short`. 3.19+ is NOT supported. Must support OCI registries (built in since Helm 3.8). |
| kubectl | 1.21+ | Required for install and all day-2 operations |
| cert-manager | 1.16.1 (recommended) | Must be installed **before** MC |

### Verify Helm version before proceeding
```bash
helm version --short
# Must print v3.14.x, v3.15.x, v3.16.x, v3.17.x, or v3.18.x
```

---

## 3. Hardware Requirements

📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html

### Platform nodes (host MC UI, API, observability, operators)

| Resource | Minimum |
|----------|---------|
| Node count | 2 |
| CPU | 32 cores per node |
| RAM | 64 GB per node (for observability stack) |
| Storage | 1 TB per node |
| Node label | `mission-control.datastax.com/role=platform` |

> **Tip:** Configure `nodeSelector` in `values.yaml` to match this label so MC platform services (UI, API, observability, operators) schedule on `platform`-labeled nodes.  
> The `allowOperatorsOnDatabaseNodes: false` default means operator pods (like Reaper) also require the platform label unless you set it to `true`.

### Database nodes (host HCD/DSE/Cassandra pods)

| Resource | Minimum | Production recommended |
|----------|---------|----------------------|
| Node count | 3 (1 node = dev/test only, never production) | 3 per datacenter |
| CPU | 16 physical cores / 32 hyper-threaded | 8–16 cores |
| RAM | 64 GB | 32–128 GB |
| Storage | 1.5 TB | Size to workload |
| Node label | `mission-control.datastax.com/role=database` | |

### JVM / heap sizing rule
```
heap_size       ≈  RAM / 4        (e.g., 32 GB RAM → 8 GB heap)
memory_request  ≈  2 × heap_size  (e.g., 8 GB heap → 16 GB memory request)
```
Start at these values and tune upward if GC pressure metrics show saturation.

> **Production note:** Schedule the monitoring stack (Mimir, Loki) and database pods on **separate node pools**. Leave at least 2 GB RAM per worker node for system/networking pods.

---

## 4. Label Worker Nodes

📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html

```bash
# Label platform nodes (repeat for every platform worker)
kubectl label nodes NODE_NAME mission-control.datastax.com/role=platform

# Label database nodes (repeat for every database worker)
kubectl label nodes NODE_NAME mission-control.datastax.com/role=database

# Verify
kubectl get nodes --show-labels | grep mission-control
```

> ⚠️ **Critical:** If no node carries the `platform` label, MC platform pods will never schedule — they stay `Pending` indefinitely with no obvious error.

---

## 5. Networking Requirements

📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html

### Required ports (open between all hosts in a region)

Open TCP/UDP ports `1024–65535` between all hosts within a given control plane or data plane region.

| Port | Instance type | Purpose |
|------|--------------|---------|
| `tcp:30880` | Platform + Database | UI service |
| `tcp:30600` | Platform | Vector aggregator (log/metric shipping) |
| `tcp:9042` | Database | CQL native protocol (client connections) |
| `tcp:7000` | Database | Cassandra internode (unencrypted) |
| `tcp:7001` | Database | Cassandra internode (TLS encrypted) |
| `tcp:8080` | Database | Management API |

### Multi-cluster pod-to-pod routing (required for multi-region)

Database pods across clusters must reach each other on ports 7000/7001/8080.

📖 Source: https://docs.datastax.com/en/mission-control/install/configure-multi-region.html

Also open between control plane and data plane clusters:
- `tcp:30600` — Vector aggregator (telemetry from DP to CP)
- `tcp:443` or `tcp:6443` — Kubernetes API server (CP manages DP)

### Load balancing
- Place the UI service (port 30880) behind a load balancer across all worker nodes for HA
- **Do NOT** put a load balancer in front of database instances — Cassandra drivers handle client-side load balancing natively

### Rancher-specific
If running on Rancher, the bundled NGINX ingress controller has a **1 MB body size limit** that causes webhook failures. Fix:
```yaml
# Add annotation to your Ingress resource:
nginx.ingress.kubernetes.io/proxy-body-size: "0"
```
📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html

---

## 6. Storage Requirements

📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/data-plane/storage-classes.html

### Object storage (required for observability)

MC requires **two separate object storage buckets** — one for metrics (Mimir) and one for logs (Loki). They **cannot share the same bucket**.

| Cloud | Supported backends |
|-------|--------------------|
| AWS | S3 or S3-compatible (e.g., MinIO, SeaweedFS, ODF Object Bucket Claim) |
| GCP | Google Cloud Storage (GCS) |
| Azure | Azure Blob Storage |

### Kubernetes StorageClass (required for database PVCs)

The StorageClass used for database PVCs **must** have `volumeBindingMode: WaitForFirstConsumer`.

> ⚠️ Without this setting, volumes may bind to a node that the pod is not scheduled on, causing deadlocks and `Pending` pod status that is very difficult to debug.

```yaml
# Example StorageClass (GKE — copy from premium-rwo and add Retain):
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: premium-rwo-retain
parameters:
  type: pd-ssd
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Retain                    # keeps PVs when cluster is deleted
volumeBindingMode: WaitForFirstConsumer  # REQUIRED
allowVolumeExpansion: true
```

### Cloud-specific StorageClass recommendations

| Cloud | Recommended class | Notes |
|-------|------------------|-------|
| EKS | `gp3` (custom class) | **EKS requires a default StorageClass** — patch gp2 as default if not already done |
| AKS | `managed-premium` or `managed-csi-premium` | |
| GKE | `premium-rwo` (or custom `premium-rwo-retain`) | |
| OpenShift | `thin-csi`, `ocs-storagecluster-ceph-rbd` | Depends on OCP storage operator |

### EKS: set a default StorageClass before install
```bash
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

---

## 7. Required Kubernetes Privileges

📖 Source: https://docs.datastax.com/en/mission-control/install/kubernetes.html

The account performing the installation must be able to:
- Create and manage cluster-scoped resources (CRDs, ClusterRoles, ClusterRoleBindings)
- Create namespaces
- Manage PersistentVolumes and StorageClasses
- Full cluster-admin is typically required for a first install (see `install-mc-helm-separate-cluster-resources` for splitting cluster-scoped resources from app-team installs)

For least-privilege Helm upgrades (app team, no cluster-admin), see:  
📖 https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html

---

## 8. Pre-Flight Checklist Summary

Run through every item before proceeding to installation:

- [ ] IBMid registered and IBM entitlement key copied from https://myibm.ibm.com/products-services/containerlibrary
- [ ] Kubernetes version confirmed: 1.21.0+ and NOT 1.35.0–1.35.3
- [ ] Helm version confirmed: 3.14.0–3.18.0
- [ ] cert-manager planned (Helm chart for standard K8s; OperatorHub for OCP)
- [ ] At least 2 platform nodes labeled `mission-control.datastax.com/role=platform`
- [ ] At least 3 database nodes labeled `mission-control.datastax.com/role=database`
- [ ] Two separate object storage buckets created (one for Mimir, one for Loki)
- [ ] StorageClass with `volumeBindingMode: WaitForFirstConsumer` exists
- [ ] For EKS: default StorageClass is set
- [ ] Ports 7000, 7001, 8080, 9042, 30880, 30600 open between nodes
- [ ] For multi-region: pod-to-pod routing planned (VPC Peering, VNet Peering, Submariner, etc.)
- [ ] Release name chosen — must **not** contain the string `mission-control`

➡️ **Next step:** Load `02-install-online.md` (online install) or `03-install-airgap.md` (air-gap install)
