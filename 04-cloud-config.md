# 04 — Cloud-Specific Configuration: EKS, AKS, GKE, OpenShift — Values, DNS, Ingress

This file contains the cloud-provider-specific `values.yaml` snippets, storage backend configuration, DNS setup, and ingress configuration for each supported platform.

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/ingress.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html

---

## Ingress overview

📖 Source: https://docs.datastax.com/en/mission-control/install/ingress.html

MC uses a `regionDomain` and `wildcardDomain` model for ingress:

| Value | Example | Routes to |
|-------|---------|-----------|
| `ingress.regionDomain` | `mc.us-east.example.com` | MC UI and API |
| `ingress.wildcardDomain` | `*.mc.us-east.example.com` | Grafana (`grafana.mc...`), DB clusters (`cluster.project.mc...`) |

```yaml
ingress:
  enabled: true
  regionDomain: "mc.us-east.example.com"
  wildcardDomain: "*.mc.us-east.example.com"
```

### DNS setup (all clouds except OpenShift)
After install, get the ingress external IP or hostname:
```bash
kubectl get svc -n mission-control | grep ingress
# Note the EXTERNAL-IP or hostname
```
Create DNS records in your provider:
- `A` (or `CNAME`) record: `mc.us-east.example.com` → ingress IP/hostname
- Wildcard `A` (or `CNAME`): `*.mc.us-east.example.com` → same ingress IP/hostname

> On OpenShift: Ingress objects are automatically converted to Routes. No manual DNS setup needed.

### OIDC + Ingress: set `ui.baseUrl`
If you use OIDC authentication alongside Ingress, you **must** set:
```yaml
ui:
  baseUrl: "https://mc.us-east.example.com"
```
Without this, OIDC redirects fail because the redirect URI won't match.

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/openid-connector.html

---

## EKS (Amazon Elastic Kubernetes Service)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/installation-requirements.html

### EKS prerequisites
```bash
# 1. Set default StorageClass (recommended for Helm)
kubectl patch storageclass gp2 \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# 2. Confirm EBS CSI driver is installed
kubectl get pods -n kube-system | grep ebs-csi
```

### EKS `values.yaml` (control plane, S3 storage)

```yaml
controlPlane: true
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

ingress:
  enabled: true
  regionDomain: "mc.us-east.example.com"
  wildcardDomain: "*.mc.us-east.example.com"

ui:
  enabled: true
  baseUrl: "https://mc.us-east.example.com"   # required if using OIDC
  https:
    enabled: true

dex:
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH"
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# Loki — logs bucket (MUST be separate from metrics bucket)
loki:
  enabled: true
  loki:
    storage:
      type: s3
      bucketNames:
        chunks: mc-logs-eks-prod        # YOUR logs bucket name
      s3:
        region: us-east-1
        accessKeyId: "${AWS_ACCESS_KEY_ID}"
        secretAccessKey: "${AWS_SECRET_ACCESS_KEY}"
        s3: s3.us-east-1.amazonaws.com
        s3ForcePathStyle: false
    limits_config:
      retention_period: 7d
  backend:
    replicas: 1
    extraArgs:
      - '-config.expand-env=true'
    extraEnv:
      - name: AWS_ACCESS_KEY_ID
        valueFrom:
          secretKeyRef:
            name: loki-s3-secrets
            key: access-key-id
      - name: AWS_SECRET_ACCESS_KEY
        valueFrom:
          secretKeyRef:
            name: loki-s3-secrets
            key: secret-access-key

# Mimir — metrics bucket (MUST be separate from logs bucket)
mimir:
  mimir:
    structuredConfig:
      blocks_storage:
        backend: s3
        s3:
          bucket_name: mc-metrics-eks-prod   # YOUR metrics bucket name
          endpoint: s3.us-east-1.amazonaws.com
          access_key_id: "${AWS_ACCESS_KEY_ID}"
          secret_access_key: "${AWS_SECRET_ACCESS_KEY}"
      limits:
        ingestion_burst_size: 100000
        ingestion_rate: 50000
        max_label_names_per_series: 120
        out_of_order_time_window: 5m
```

Create S3 credential secrets before install:
```bash
kubectl create secret generic loki-s3-secrets -n mission-control \
  --from-literal=access-key-id=YOUR_KEY \
  --from-literal=secret-access-key=YOUR_SECRET

kubectl create secret generic mimir-s3-secrets -n mission-control \
  --from-literal=access-key-id=YOUR_KEY \
  --from-literal=secret-access-key=YOUR_SECRET
```

### EKS custom StorageClass for database nodes
```yaml
# eks-storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mc-database-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
volumeBindingMode: WaitForFirstConsumer   # REQUIRED
reclaimPolicy: Retain
allowVolumeExpansion: true
```

---

## AKS (Azure Kubernetes Service)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

### AKS `values.yaml` (control plane, Azure Blob storage)

```yaml
controlPlane: true
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

ingress:
  enabled: true
  regionDomain: "mc.eastus.example.com"
  wildcardDomain: "*.mc.eastus.example.com"

ui:
  enabled: true
  baseUrl: "https://mc.eastus.example.com"
  https:
    enabled: true

dex:
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH"
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# Loki — logs container (MUST differ from metrics container)
loki:
  enabled: true
  loki:
    storage:
      type: azure
      bucketNames:
        chunks: mc-logs-aks-prod          # YOUR logs container name
      azure:
        accountName: YOUR_STORAGE_ACCOUNT
        accountKey: YOUR_STORAGE_KEY
        endpoint_suffix: blob.core.windows.net
    structuredConfig:
      storage_config:
        azure:
          container_name: mc-logs-aks-prod

# Mimir — metrics container (MUST differ from logs container)
mimir:
  mimir:
    structuredConfig:
      common:
        storage:
          backend: azure
          azure:
            account_name: YOUR_STORAGE_ACCOUNT
            account_key: YOUR_STORAGE_KEY
            endpoint_suffix: blob.core.windows.net
      blocks_storage:
        backend: azure
        azure:
          container_name: mc-metrics-aks-prod   # YOUR metrics container name
      limits:
        ingestion_burst_size: 100000
        ingestion_rate: 50000
        max_label_names_per_series: 120
        out_of_order_time_window: 5m
```

### Azure Arc alternative install path
MC can also be installed via Azure Marketplace as `DataStax HCD for Azure Arc` (no Helm required):
1. Azure Portal → Marketplace → search "DataStax HCD for Azure Arc"
2. Select plan, resource group, Arc-enabled K8s cluster
3. Configure storage account (two containers: one for metrics, one for logs)
4. Configure authentication and UI settings → Create

📖 Source: https://docs.datastax.com/en/mission-control/integrations/hcd-azure-arc.html

---

## GKE (Google Kubernetes Engine)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

### GKE `values.yaml` (control plane, GCS storage)

```yaml
controlPlane: true
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

# GKE: use native ingress (not HAProxy)
kubernetes-ingress:
  enabled: false

ingress:
  enabled: true
  regionDomain: "mc.us-central1.example.com"
  wildcardDomain: "*.mc.us-central1.example.com"

ui:
  enabled: true
  baseUrl: "https://mc.us-central1.example.com"
  https:
    enabled: true

dex:
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH"
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# Loki — GCS logs bucket
loki:
  enabled: true
  backend:
    extraEnv:
      - name: GOOGLE_APPLICATION_CREDENTIALS
        value: /etc/loki_secrets/gcp_service_account.json
    extraVolumeMounts:
      - mountPath: /etc/loki_secrets
        name: loki-secrets
    extraVolumes:
      - name: loki-secrets
        secret:
          secretName: loki-gcs-secrets
          items:
            - key: gcp_service_account.json
              path: gcp_service_account.json
  loki:
    storage:
      type: gcs
      bucketNames:
        chunks: mc-logs-gke-prod          # YOUR GCS logs bucket name
      gcs:
        insecure: false
    limits_config:
      retention_period: 7d

# Mimir — GCS metrics bucket
mimir:
  mimir:
    structuredConfig:
      blocks_storage:
        backend: gcs
        gcs:
          bucket_name: mc-metrics-gke-prod   # YOUR GCS metrics bucket name
          service_account: 'GCP_SA_JSON_CONTENT'
      limits:
        ingestion_burst_size: 100000
        ingestion_rate: 50000
        max_label_names_per_series: 120
        out_of_order_time_window: 5m
```

Create GCS credential secret:
```bash
kubectl create secret generic loki-gcs-secrets -n mission-control \
  --from-file=gcp_service_account.json=/path/to/sa-key.json
```

### GKE StorageClass for database nodes
```bash
# Copy premium-rwo and add Retain policy
kubectl get storageclass premium-rwo -o yaml > premium-rwo.yaml
# Edit: change name to premium-rwo-retain, reclaimPolicy: Retain
kubectl apply -f premium-rwo.yaml
```

---

## OpenShift (OCP)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

### ⚠️ Critical OCP-specific gotchas

1. **No HAProxy ingress** — OCP auto-creates Routes from Ingress objects. Do NOT enable `kubernetes-ingress`.
2. **DNS resolver** — OCP uses `openshift-dns` namespace, not `kube-dns`. Without this config, Loki and Mimir **silently stop collecting** metrics and logs — no error is shown.
3. **SCC permissions** — must be granted before `helm install` or pods will fail to schedule.
4. **cert-manager** — must come from OperatorHub, not the Helm chart.
5. **Uninstall order** — remove SCC grants **before** `helm uninstall` or resources hang.

### Get your OCP cluster domain
```bash
oc get ingress.config.openshift.io cluster -o jsonpath='{.spec.domain}'
# Example output: apps.myocp.us-east.example.com
# Use this as the base for your regionDomain: mc.apps.myocp.us-east.example.com
```

### OCP `values.yaml` (control plane)

```yaml
controlPlane: true
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

# DO NOT enable HAProxy on OpenShift — OCP handles ingress natively
kubernetes-ingress:
  enabled: false

ingress:
  enabled: true
  # Use the OCP cluster domain as your base
  regionDomain: "mc.apps.myocp.us-east.example.com"
  wildcardDomain: "*.mc.apps.myocp.us-east.example.com"

ui:
  enabled: true
  baseUrl: "https://mc.apps.myocp.us-east.example.com"
  https:
    enabled: true

dex:
  config:
    enablePasswordDB: true
    staticPasswords:
      - email: admin@example.com
        hash: "BCRYPT_HASH"
        username: admin
        userID: "08a8684b-db88-4b73-90a9-3cd1661f5466"

# CRITICAL: OCP DNS configuration
# Without this, Loki and Mimir silently fail to resolve service names
loki:
  global:
    clusterDomain: cluster.local
    dnsNamespace: openshift-dns    # OCP-specific: NOT kube-system
    dnsService: dns-default        # OCP-specific: NOT kube-dns

mimir:
  gateway:
    nginx:
      config:
        resolver: dns-default.openshift-dns.svc.cluster.local
  nginx:
    nginxConfig:
      resolver: "dns-default.openshift-dns.svc"

k8ssandra-operator:
  disableCrdUpgraderJob: true
  cass-operator:
    disableCertManagerCheck: true
```

### Verify OCP DNS config
```bash
oc get svc -n openshift-dns
# Should show dns-default service
# If your cluster uses a different DNS service name, update dnsService above
```

### OCP data plane `values.yaml`
```yaml
# For a data plane on OCP:
controlPlane: false
disableCertManagerCheck: true

nodeSelector:
  mission-control.datastax.com/role: platform

kubernetes-ingress:
  enabled: false

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

# Forward telemetry to control plane
aggregator:
  customConfig:
    sinks:
      control_plane_aggregator:
        type: vector
        address: "https://vector.mc.apps.myocp-cp.example.com"
```

---

## Vector aggregator: NodePort vs Ingress (v1.18.0+)

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

Since v1.18.0, the Vector aggregator default changed from NodePort to Ingress for accepting metrics/logs from external data planes.

**Option A — Ingress (recommended for v1.18.0+):**
```yaml
aggregator:
  customConfig:
    sinks:
      control_plane_aggregator:
        type: vector
        address: https://vector.mc.us-east.example.com
```

**Option B — Revert to NodePort (legacy compatibility):**
```yaml
aggregator:
  service:
    type: NodePort
    ports:
      - name: vector
        protocol: TCP
        port: 6000
        targetPort: 6000
        nodePort: 30600
```

---

## GitOps / IaC integration

MC Helm values files are GitOps-compatible — store them in version control with Argo CD or Flux.

📖 Source: https://docs.datastax.com/en/mission-control/integrations/gitops-workflows.html

---

## Next Steps

- TLS, LDAP, OIDC → `05-security.md`
- Deploy HCD/DSE cluster, multi-region, backups → `06-cluster-ops.md`
- Troubleshooting → `07-troubleshoot.md`
