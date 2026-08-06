# 06 — Cluster Operations: Deploy HCD, Multi-Region, Backup/Restore, Upgrades, Migration, Day-2

📖 Source: https://docs.datastax.com/en/mission-control/install/configure-multi-region.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/backup.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/restore.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/upgrade-mc-helm.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/data-plane/storage-classes.html  
📖 Source: https://docs.datastax.com/en/mission-control/reference/workload-pinning.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/canary-deployments.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/per-node-configuration.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/replace-db-node.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/rebuild-failed-datacenter.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/add-db-nodes.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/remove-db-nodes.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/remove-db-datacenter.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/config-nosql.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/data-plane/configure-hcd-guardrails.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html

---

## 1. StorageClass: Required Configuration Before Creating Clusters

📖 Source: https://docs.datastax.com/en/mission-control/administration/data-plane/storage-classes.html

**Every StorageClass used for database PVCs must have `volumeBindingMode: WaitForFirstConsumer`.**  
Without this, volumes may bind on a different node than the pod is scheduled on, causing a deadlock.

```bash
# Verify your intended StorageClass
kubectl describe storageclass YOUR_STORAGE_CLASS | grep VolumeBindingMode
# Must show: WaitForFirstConsumer
```

Create a production StorageClass with `reclaimPolicy: Retain` (PVs survive cluster deletion):

```yaml
# storageclass-retain.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: mc-db-retain
parameters:
  type: pd-ssd                    # GKE example — change for your cloud
provisioner: pd.csi.storage.gke.io
reclaimPolicy: Retain             # PVs persist after MissionControlCluster deletion
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

```bash
kubectl apply -f storageclass-retain.yaml
```

---

## 2. Deploy Your First HCD/DSE/Cassandra Cluster

### Via the Mission Control UI (simplest)
1. Log in → **Clusters → Create Cluster**
2. Choose database type (HCD recommended), version, and cluster name
3. Configure datacenter name, number of nodes, rack topology (3 racks → 3 AZs for HA)
4. Set StorageClass, storage size, CPU and memory requests
5. Set superuser credentials (or generate — retrieve later with `kubectl get secret`)
6. Click **Create** and monitor progress in the dashboard

### Via `kubectl` / GitOps (recommended for production)

📖 Source: https://docs.datastax.com/en/mission-control/install/configure-multi-region.html

```yaml
# hcd-cluster.yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: production-hcd
  namespace: my-project             # the "project" namespace
spec:
  createIssuer: true
  k8ssandra:
    auth: true                      # keep true — default superuser created automatically
    cassandra:
      serverType: hcd
      serverVersion: "2.0.6"        # check release notes for latest
      resources:
        requests:
          cpu: "4"
          memory: "16Gi"
        limits:
          memory: "16Gi"
      config:
        jvmOptions:
          gc: G1GC
          heapSize: 8Gi             # ~1/4 of memory request
      datacenters:
        - metadata:
            name: dc1
          size: 3                   # 3 nodes minimum for production (must be multiple of rack count)
          racks:
            - name: rack1
              nodeAffinityLabels:
                topology.kubernetes.io/zone: us-east-1a
                mission-control.datastax.com/role: database
            - name: rack2
              nodeAffinityLabels:
                topology.kubernetes.io/zone: us-east-1b
                mission-control.datastax.com/role: database
            - name: rack3
              nodeAffinityLabels:
                topology.kubernetes.io/zone: us-east-1c
                mission-control.datastax.com/role: database
          storageConfig:
            cassandraDataVolumeClaimSpec:
              storageClassName: mc-db-retain    # must have WaitForFirstConsumer
              accessModes: [ReadWriteOnce]
              resources:
                requests:
                  storage: 500Gi
```

```bash
kubectl apply -f hcd-cluster.yaml
kubectl get MissionControlCluster -n my-project
kubectl get pods -n my-project -w   # watch nodes come up one by one
```

### Retrieve superuser credentials

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/authn.html

```bash
# The superuser secret name is: <CLUSTER_NAME>-superuser
# Username
kubectl get secret production-hcd-superuser -n my-project \
  -o jsonpath='{.data.username}' | base64 --decode

# Password
kubectl get secret production-hcd-superuser -n my-project \
  -o jsonpath='{.data.password}' | base64 --decode
```

> ⚠️ The default `cassandra` user is **disabled** by default in MC-managed clusters. Use only the `CLUSTER_NAME-superuser` account.  
> ⚠️ Keep `auth: true` — disabling auth removes the superuser protection.

When running nodetool with auth enabled:

```bash
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- \
  nodetool -u production-hcd-superuser -pw PASSWORD status
```

### Workload pinning (node affinity / placement)

📖 Source: https://docs.datastax.com/en/mission-control/reference/workload-pinning.html

To pin database pods to specific nodes using labels, use `nodeAffinityLabels` in each rack spec (shown above). This maps to `requiredDuringSchedulingIgnoredDuringExecution` node affinity rules.

---

## 3. Configure Cluster Settings (cassandraYaml / dseYaml / jvmOptions)

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/config-nosql.html

All cluster-level configuration lives under `spec.k8ssandra.cassandra.config` in the `MissionControlCluster`:

```yaml
spec:
  k8ssandra:
    cassandra:
      config:
        cassandraYaml:
          num_tokens: 16
          read_request_timeout_in_ms: 10000
          write_request_timeout_in_ms: 10000
          concurrent_reads: 32
          concurrent_writes: 32
          concurrent_counter_writes: 16
          compaction_throughput_mb_per_sec: 64
          # Any valid cassandra.yaml key is supported
        dseYaml:
          # DSE-specific settings (DSE clusters only)
          enable_health_based_routing: true
        jvmOptions:
          gc: G1GC
          heapSize: 8Gi
          # additional_jvm_opts:
          #   - "-Dcassandra.disable_auth_caches_remote_configuration=true"
```

> **Rolling config changes**: When you apply a config change at the cluster level, MC applies it rack-by-rack, node-by-node. One node is updated at a time within a rack before moving to the next.

```bash
# Verify a config change was applied
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- \
  grep read_request_timeout_in_ms /etc/cassandra/cassandra.yaml
```

### HCD Guardrails

📖 Source: https://docs.datastax.com/en/mission-control/administration/data-plane/configure-hcd-guardrails.html

Set guardrails for HCD clusters via `config.cassandraYaml.guardrails`:

```yaml
spec:
  k8ssandra:
    cassandra:
      config:
        cassandraYaml:
          guardrails:
            sai_indexes_per_table_failure_threshold: 50
            sai_indexes_per_table_warn_threshold: 30
            tables_failure_threshold: 100
            tables_warn_threshold: 80
            partition_size_warn_threshold_in_mb: 100
```

---

## 4. Per-Node Configuration (Advanced)

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/per-node-configuration.html

> ⚠️ This is an advanced operation. Use only when specific nodes require different settings (e.g., `initial_token` for token assignment, or node-specific JVM flags).

**Key constraints:**
- The ConfigMap **must exist BEFORE** the `MissionControlCluster` is created
- Keys in the ConfigMap must be named `<POD_NAME>_cassandra.yaml` or `<POD_NAME>_dse.yaml`
- Attaching or detaching a `perNodeConfigMapRef` causes a **rolling restart of ALL pods** in that DC
- `initial_token` assignment only works at cluster **creation** time, not after

```yaml
# Step 1: Create ConfigMap with per-node settings
apiVersion: v1
kind: ConfigMap
metadata:
  name: dc1-per-node-config
  namespace: my-project
data:
  production-hcd-dc1-r1-sts-0_cassandra.yaml: |
    initial_token: "-9223372036854775808"
  production-hcd-dc1-r2-sts-0_cassandra.yaml: |
    initial_token: "-3074457345618258603"
  production-hcd-dc1-r3-sts-0_cassandra.yaml: |
    initial_token: "3074457345618258602"
```

```bash
# Apply BEFORE creating the MissionControlCluster
kubectl apply -f dc1-per-node-config.yaml
```

```yaml
# Step 2: Reference in MissionControlCluster
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1
          perNodeConfigMapRef:
            name: dc1-per-node-config   # references the ConfigMap
```

---

## 5. Canary Deployments (Validate Changes on One DC First)

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/canary-deployments.html

Use a canary pattern to apply a config change or version upgrade to a single datacenter first, validate it, then roll it to all datacenters.

```
Cluster-level config  ←─ base configuration (applies to all DCs)
DC-level config       ←─ merged ON TOP of cluster config for this DC only
```

**Step 1: Apply the change at the DC level only**

```yaml
spec:
  k8ssandra:
    cassandra:
      serverVersion: "2.0.5"          # current cluster-wide version
      datacenters:
        - metadata:
            name: dc1                 # canary DC
          config:                     # DC-level config MERGES with cluster-level
            cassandraYaml:
              read_request_timeout_in_ms: 15000
              # Or for version canary:
          # serverVersion: "2.0.6"   # upgrade dc1 only first
        - metadata:
            name: dc2                 # other DCs remain at cluster-level settings
```

```bash
kubectl apply -f hcd-cluster.yaml   # only dc1 gets the change
# Watch dc1 rolling restart:
kubectl get pods -n my-project -w
# Validate: nodetool, metrics, application smoke tests
```

**Step 2: After validation, promote to cluster-level**

```yaml
spec:
  k8ssandra:
    cassandra:
      serverVersion: "2.0.6"          # promote version upgrade cluster-wide
      config:
        cassandraYaml:
          read_request_timeout_in_ms: 15000
      datacenters:
        - metadata:
            name: dc1                 # remove dc-level override (operators skip dc1 — it already has the change)
        - metadata:
            name: dc2
```

> ⚠️ When you promote to cluster level, MC operators detect that `dc1` already has the config applied and **skip it**. Expect ~14 minutes per remaining datacenter for the rolling change.

---

## 6. Scale a Datacenter (Add / Remove Nodes)

### Add nodes

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/add-db-nodes.html

> ⚠️ **Critical rule**: size must always be a **multiple of the rack count**. With 3 racks, add 3, 6, or 9 nodes at a time — never an odd number.

```bash
# Scale from 3 to 6 nodes (3-rack cluster)
kubectl patch MissionControlCluster production-hcd \
  --type=json -n my-project \
  -p '[{"op":"replace","path":"/spec/k8ssandra/cassandra/datacenters/0/size","value":6}]'

# Monitor the scale-up
kubectl get CassandraDatacenter dc1 -n my-project -o yaml | grep cassandraOperatorProgress
kubectl get pods -n my-project -w
```

> ℹ️ MC automatically runs `nodetool cleanup` on the **original** nodes after new nodes join. This is required to rebalance data off the old nodes. No manual cleanup needed.

### Remove nodes

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/remove-db-nodes.html

> ⚠️ Same rule: decrease size by multiples of rack count. Do **not** remove nodes unless free space on remaining nodes is sufficient to hold the decommissioned nodes' data.

```bash
# Scale down from 6 to 3 nodes
kubectl patch MissionControlCluster production-hcd \
  --type=json -n my-project \
  -p '[{"op":"replace","path":"/spec/k8ssandra/cassandra/datacenters/0/size","value":3}]'

# Monitor: watch for ScalingDown condition
kubectl get CassandraDatacenter dc1 -n my-project -o yaml | grep -A5 conditions
```

If you see the error **`NotEnoughSpaceToScaleDown`** in the DC status, the remaining nodes don't have enough free disk to absorb the decommissioned nodes' data. You must either add more storage or add more nodes before reducing.

---

## 7. Replace a Failed Node

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/replace-db-node.html

Use `CassandraTask` with `command: replacenode` to replace a dead Cassandra node:

```yaml
# replace-node.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: replace-node
  namespace: my-project
spec:
  datacenter:
    name: production-hcd-dc1
    namespace: my-project
  jobs:
    - command: replacenode
      name: replace
      args:
        pod_name: "production-hcd-dc1-r1-sts-0"   # name of the failed pod
  restartPolicy: Never
  ttlSecondsAfterFinished: 300
```

```bash
kubectl apply -f replace-node.yaml

# Monitor streaming progress (replacement streams data from peers)
kubectl exec -it ALIVE_POD -n my-project -c cassandra -- nodetool netstats
# Watch: streaming percentages decrease toward 0
```

**Parallel node replacement (v1.18+):**

Use `K8ssandraTask` with `maxConcurrentPods` to replace multiple nodes in the same rack simultaneously:

```yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: K8ssandraTask
metadata:
  name: replace-multiple
  namespace: my-project
spec:
  cluster:
    name: production-hcd
    namespace: my-project
  template:
    spec:
      jobs:
        - command: replacenode
          name: replace
  maxConcurrentPods: 2               # replace up to 2 nodes at once within same rack
```

---

## 8. Remove a Datacenter

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/remove-db-datacenter.html

> ⚠️ MC **blocks termination** if any user keyspace still references the datacenter in its replication settings. You **must** run ALTER KEYSPACE first.

```sql
-- Step 1: For each user keyspace, remove the DC from replication
ALTER KEYSPACE my_keyspace WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3    -- remove dc2 from this list
};

-- Repeat for all user keyspaces
-- System keyspaces: MC handles these automatically
```

```yaml
# Step 2: Remove the datacenter from MissionControlCluster
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1              # keep dc1
        # dc2 removed from list
```

```bash
kubectl apply -f hcd-cluster.yaml
# MC will verify no user keyspaces reference dc2, then decommission and delete it
kubectl get CassandraDatacenter -n my-project   # dc2 should disappear
```

---

## 9. Rebuild a Failed Datacenter (Full DC Failure)

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/rebuild-failed-datacenter.html

When an entire datacenter has failed and must be rebuilt from scratch:

**Step 1: Repair system keyspaces on the surviving DC**

```bash
kubectl exec -it ALIVE_POD -n my-project -c cassandra -- \
  nodetool repair -pr system_auth system_distributed system_traces
```

**Step 2: ALTER KEYSPACE — remove the failed DC from replication**

```sql
ALTER KEYSPACE my_keyspace WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3    -- only surviving DC
};
-- Repeat for all user keyspaces
```

**Step 3: Remove the failed DC from MissionControlCluster**

```yaml
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1        # keep surviving DC
        # dc2 removed — MC removes pods and PVCs
```

```bash
kubectl apply -f hcd-cluster.yaml
# If resources hang during cleanup, assassinate stuck nodes:
kubectl exec -it ALIVE_POD -n my-project -c cassandra -- \
  nodetool assassinate FAILED_NODE_IP
```

**Step 4: Re-add the datacenter**

```yaml
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1
        - metadata:
            name: dc2              # re-add with fresh nodes
          size: 3
          racks:
            - name: r1
            - name: r2
            - name: r3
```

```bash
kubectl apply -f hcd-cluster.yaml
# Wait for dc2 nodes to reach UN state, then restore replication:
```

```sql
ALTER KEYSPACE my_keyspace WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3,
  'dc2': 3     -- add dc2 back
};
```

**Step 5: Rebuild the new datacenter (stream data from dc1)**

```yaml
# rebuild-dc.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: K8ssandraTask
metadata:
  name: rebuild-dc2
  namespace: my-project
spec:
  cluster:
    name: production-hcd
    namespace: my-project
  template:
    spec:
      datacenter:
        name: production-hcd-dc2
        namespace: my-project
      jobs:
        - command: rebuild
          name: rebuild
          args:
            source_datacenter: "dc1"   # stream all data from dc1 to dc2
```

```bash
kubectl apply -f rebuild-dc.yaml
# Monitor via nodetool netstats on a dc2 node
kubectl exec -it DC2_POD -n my-project -c cassandra -- nodetool netstats
```

---

## 10. Backup with Medusa

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/backup.html

### Step 1: Create the Medusa credentials secret (BEFORE creating the cluster)

> ⚠️ The Medusa secret must exist in the cluster namespace **before** the `MissionControlCluster` is applied.

**AWS S3:**
```bash
kubectl create secret generic medusa-bucket-key \
  --namespace my-project \
  --from-literal=credentials="[default]
aws_access_key_id = YOUR_KEY_ID
aws_secret_access_key = YOUR_SECRET_KEY"
```

**Google Cloud Storage:**
```bash
# Create a JSON service account key file (from GCP IAM)
kubectl create secret generic medusa-bucket-key \
  --namespace my-project \
  --from-file=credentials.json=/path/to/gcs-service-account.json
```

```yaml
# GCS MedusaBackup spec reference in storageProperties:
storageProperties:
  storageProvider: gcs
  bucketName: my-gcs-backup-bucket
  region: us-central1
  storageSecretRef:
    name: medusa-bucket-key
```

**Azure Blob Storage:**
```bash
kubectl create secret generic medusa-bucket-key \
  --namespace my-project \
  --from-literal=credentials="{
  \"storage_account\": \"YOUR_ACCOUNT_NAME\",
  \"storage_key\": \"YOUR_STORAGE_KEY\"
}"
```

```yaml
# Azure storageProperties:
storageProperties:
  storageProvider: azure_blobs
  bucketName: my-azure-container
  storageSecretRef:
    name: medusa-bucket-key
```

**S3-Compatible (MinIO, SeaweedFS, etc.):**
```bash
kubectl create secret generic medusa-bucket-key \
  --namespace my-project \
  --from-literal=credentials="[default]
aws_access_key_id = YOUR_KEY_ID
aws_secret_access_key = YOUR_SECRET_KEY"
```

```yaml
# S3-compatible storageProperties:
storageProperties:
  storageProvider: s3_compatible
  bucketName: my-backup-bucket
  host: seaweedfs.example.com          # your S3-compatible endpoint
  port: 8333
  secure: true
  storageSecretRef:
    name: medusa-bucket-key
```

### Step 2: Enable Medusa in the MissionControlCluster spec

```yaml
spec:
  k8ssandra:
    medusa:
      storageProperties:
        storageProvider: s3            # s3, gcs, azure_blobs, s3_compatible
        bucketName: my-backup-bucket
        region: us-east-1
        prefix: production-hcd         # optional path prefix
        maxBackupAge: 30               # days — auto-purge backups older than this
        maxBackupCount: 10             # keep at most 10 backups (older purged)
        concurrentTransfers: 4         # parallel upload streams per node
        storageSecretRef:
          name: medusa-bucket-key
```

### Step 3: Create an on-demand backup

```yaml
# backup-now.yaml
apiVersion: medusa.k8ssandra.io/v1alpha1
kind: MedusaBackupJob
metadata:
  name: backup-20250901
  namespace: my-project
spec:
  cassandraDatacenter: dc1
```

```bash
kubectl apply -f backup-now.yaml
kubectl get MedusaBackup -n my-project          # watch for COMPLETED status
kubectl describe MedusaBackup backup-20250901 -n my-project
```

### Step 4: Schedule recurring backups

```yaml
# backup-schedule.yaml
apiVersion: medusa.k8ssandra.io/v1alpha1
kind: MedusaBackupSchedule
metadata:
  name: daily-backup
  namespace: my-project
spec:
  backupSpec:
    backupType: differential          # differential or full
    cassandraDatacenter: dc1
  cronSchedule: "30 1 * * *"         # 01:30 UTC daily
  disabled: false
```

> ℹ️ **Backups continue running even if the Mission Control control plane is down.** The Medusa sidecar inside each Cassandra pod runs independently and will execute scheduled backups without CP connectivity.

---

## 11. Restore with Medusa

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/restore.html

```yaml
# restore.yaml
apiVersion: medusa.missioncontrol.datastax.com/v1beta2
kind: MedusaRestoreJob
metadata:
  name: restore-20250901
  namespace: my-project
spec:
  cassandraDatacenter: dc1
  backup: backup-20250901          # must exactly match a MedusaBackup metadata.name
```

```bash
kubectl apply -f restore.yaml

# Monitor — MC shuts down all pods, restores data, restarts
kubectl get MedusaRestoreJob restore-20250901 -o yaml -n my-project
kubectl describe MedusaRestoreJob restore-20250901 -n my-project
```

> ⚠️ Restore causes downtime for the affected datacenter while data is being restored.

---

## 12. Multi-Region Deployment

📖 Source: https://docs.datastax.com/en/mission-control/install/configure-multi-region.html

### Prerequisites

- Two or more Kubernetes clusters in separate regions
- Pod-to-pod network connectivity between clusters (VPC Peering / VNet Peering / Submariner)
- Ports 7000, 7001, 8080 open between DB pods across clusters
- Port 30600 (or ingress URL) open for Vector telemetry from data plane to control plane
- Kubernetes API server port (443 or 6443) accessible from control plane to data plane

### Verify connectivity between clusters

```bash
kubectl config get-contexts
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

### Install control plane (region 1)

```bash
kubectl config use-context CP_EAST
helm install mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --namespace mission-control --create-namespace \
  -f values-cp-east.yaml
```

### Install data plane (region 2+)

> ⚠️ **The data plane release name must exactly match the control plane release name.**

```bash
kubectl config use-context DP_WEST
helm install mc-release \          # SAME name as control plane
  oci://icr.io/mission-control-helm/mission-control \
  --namespace mission-control --create-namespace \
  -f values-dp-west.yaml
```

**`values-dp-west.yaml` (data plane):**

```yaml
controlPlane: false
disableCertManagerCheck: false
nodeSelector:
  mission-control.datastax.com/role: platform
allowOperatorsOnDatabaseNodes: false
client:
  manageCrds: true
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
aggregator:
  service:
    type: ClusterIP
    ports:
      - name: vector
        protocol: TCP
        port: 6000
        targetPort: 6000
  customConfig:
    sources:
      mimir-self-monitoring: {}
    sinks:
      mimir: {}
      loki: {}
      control_plane_aggregator:
        type: vector
        inputs: ["vector", "internal_metrics", "kube_state_metrics", "cass_operator_metrics"]
        address: "https://vector.mc.us-east.example.com"   # CP aggregator URL
k8ssandra-operator:
  disableCrdUpgraderJob: true
  cass-operator:
    disableCertManagerCheck: true
```

### Register the data plane with the control plane

```bash
kubectl config use-context CP_EAST
mcctl register --source-context DP_WEST --dest-context CP_EAST
kubectl get -n mission-control clientconfig
kubectl get -n mission-control secret | grep dataplane
```

### Multi-datacenter MissionControlCluster across two regions

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: multi-region-cluster
  namespace: my-project
spec:
  createIssuer: true
  encryption:
    internodeEncryption:
      enabled: true
      certs:
        createCerts: true
  k8ssandra:
    auth: true
    cassandra:
      serverType: hcd
      serverVersion: "2.0.6"
      datacenters:
        - metadata:
            name: dc-us-east
          k8sContext: CP_EAST
          size: 3
          racks:
            - name: r1
            - name: r2
            - name: r3
        - metadata:
            name: dc-eu-west
          k8sContext: DP_WEST
          size: 3
          racks:
            - name: r1
            - name: r2
            - name: r3
```

---

## 13. Upgrade Mission Control (Helm)

📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/upgrade-mc-helm.html

### Upgrade order (critical): DATA PLANES FIRST, then CONTROL PLANE

```bash
# Step 1: Upgrade each data plane cluster FIRST
kubectl config use-context DP_WEST
helm upgrade mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --namespace mission-control \
  -f values-dp-west.yaml
kubectl get pods -n mission-control -w

# Step 2: Upgrade the control plane LAST
kubectl config use-context CP_EAST
helm upgrade mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --namespace mission-control \
  -f values-cp-east.yaml
```

> Pause long-running repair operations before upgrading.  
> Avoid scheduling new backup jobs during the upgrade window.  
> The MC UI remains accessible throughout the upgrade.

---

## 14. Upgrade HCD/DSE Cluster Version (Rolling, Zero-Downtime)

```bash
# Edit serverVersion in MissionControlCluster spec
kubectl edit MissionControlCluster production-hcd -n my-project
# Change: serverVersion: "2.0.5" → "2.0.6"

# Or via canary pattern (recommended):
# 1. Apply at DC level only → validate → promote to cluster level
# See Section 5 (Canary Deployments) above

# Monitor rolling upgrade
kubectl get cassandradatacenter dc1 -n my-project -o yaml | grep cassandraOperatorProgress
kubectl get pods -n my-project -w
```

---

## 15. Manage Clusters During a Control Plane Outage

📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html

**When the Mission Control control plane is unavailable:**
- ✅ Scheduled backups **continue** running (Medusa runs inside each pod, not through CP)
- ❌ Repairs do **NOT** run (Reaper is managed by CP)
- ✅ You can manage the data plane directly via `kubectl` against the DP cluster

### Switch kubectl context to the data plane

```bash
kubectl config use-context DP_WEST    # or CP_EAST if CP is down but DP context is different
```

### Directly edit CassandraDatacenter

```bash
kubectl edit CassandraDatacenter dc1 -n my-project
# Make your changes directly — cass-operator (running on the DP) will apply them
```

### CassandraTask commands available during CP outage

All `CassandraTask` operations are handled by `cass-operator` on the data plane and work without CP connectivity:

| Command | Purpose |
|---------|---------|
| `restart` | Rolling restart of all pods in the DC |
| `cleanup` | Remove data not belonging to this node (after topology change) |
| `rebuild` | Stream data from a source DC |
| `upgradesstables` | Upgrade SSTables to current version |
| `compaction` | Run a major compaction |
| `scrub` | Scrub SSTables (checks for corruption) |
| `move` | Move a token to a new range |
| `flush` | Flush all memtables to disk |
| `garbagecollect` | Remove deleted data from SSTables |
| `refresh` | Load new SSTables from disk (after manual copy) |

### Control CP recovery behavior with annotation

```bash
# Allow CP to overwrite any local changes you made during the outage (default on recovery)
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=always

# Allow CP to overwrite only once (then annotation is removed)
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=once

# Prevent CP from overwriting your local changes on recovery (use with care)
# Simply remove the annotation or set to a non-matching value
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec-  # remove annotation
```

> ⚠️ If you make local DP changes during a CP outage and do NOT want them overwritten when CP recovers, remove the `autoupdate-spec` annotation. By default (`always`), CP will overwrite local changes on reconnect.

---

## 16. Migrate Existing Clusters to Mission Control

📖 Source: https://docs.datastax.com/en/mission-control/migrate/migration-overview.html

### Zero-downtime: add a new MC-managed datacenter

```yaml
# migration-cluster.yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: existing-cluster-name      # MUST match the existing cluster name exactly
  namespace: my-project
spec:
  k8ssandra:
    cassandra:
      serverType: cassandra
      serverVersion: "4.1.7"        # match existing version
      clusterName: "existing-cluster-name"
      externalDatacenters:
        - name: existing-dc1        # name of your current, externally managed DC
      datacenters:
        - metadata:
            name: new-dc-mc
          k8sContext: target-cluster-context
          size: 3
          racks:
            - name: r1
          config:
            cassandraYaml:
              additionalSeeds:
                - "SEED_IP_1"
                - "SEED_IP_2"
```

```bash
kubectl apply -f migration-cluster.yaml

# Once new DC is fully up (all nodes UN):
# 1. ALTER KEYSPACE to include new DC
# 2. Run repair on the new DC to stream data
# 3. Update application connection strings
# 4. Decommission old DC nodes one by one
```

📖 Source: https://docs.datastax.com/en/mission-control/migrate/oss-cass-to-mission-control.html

---

## 17. Day-2 Operations Quick Reference

### Health checks

```bash
kubectl get MissionControlCluster -A
kubectl get CassandraDatacenter -A
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- nodetool status
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- nodetool tpstats
kubectl top pod -n mission-control
kubectl get events -n my-project --sort-by='.lastTimestamp' | tail -30
```

### Stop and start a datacenter (for maintenance)

```yaml
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1
          stopped: true    # stop — set false to restart
```

---

## See Also

- GitOps / IaC workflows: https://docs.datastax.com/en/mission-control/integrations/gitops-workflows.html
- Azure Arc install: https://docs.datastax.com/en/mission-control/integrations/hcd-azure-arc.html
- Cluster best practices: https://docs.datastax.com/en/mission-control/reference/cluster-best-practices.html
- Upgrade RBAC: https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html
- CRD reference: https://docs.datastax.com/en/mission-control/crd-reference/crd-reference-overview.html
- Service accounts reference: https://docs.datastax.com/en/mission-control/reference/service-accounts.html
