# 06 — Cluster Operations: Deploy HCD, Multi-Region, Backup/Restore, Upgrades, Migration, Day-2

📖 Source: https://docs.datastax.com/en/mission-control/install/configure-multi-region.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/backup.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/restore.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/backup-restore-compatibility.html
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
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/rolling-restart.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/upgrade-sstables.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/upgrade-nosql-cluster.html
📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html
📖 Source: https://docs.datastax.com/en/mission-control/migrate/oss-cass-to-mission-control.html

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
      serverVersion: "2.0.8"        # check release notes for latest
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
          # serverVersion: "2.0.8"   # upgrade dc1 only first
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
      serverVersion: "2.0.8"          # promote version upgrade cluster-wide
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

## 5A. Rolling Restart

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/rolling-restart.html

Restart all nodes in a cluster in a rolling fashion with zero downtime. Use after adding or terminating nodes, or to apply configuration changes that require a restart.

Mission Control iterates one rack at a time, restarting pods one at a time within each rack. Starting with v1.18.0, parallel rack restarts are supported for faster operations.

> ℹ️ A rolling restart has no significant impact on performance or stability as long as the replication factor (RF) is 3 or higher. With RF ≥ 3, one node can be down while maintaining quorum reads and writes.

### Via the UI

1. In MC UI → **Home → Clusters**, click your cluster namespace.
2. In the **Nodes** section of the **Overview** tab, select the target datacenter row checkbox.
3. Click ⋮ **More Options** → **Restart**.
4. To restart all DCs simultaneously: select the **Name** checkbox (selects all DCs) → **Bulk Actions** → confirm **Action type: Restart** → **Run**.

### Via `kubectl` — CassandraTask (single DC)

```yaml
# restart-task.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: restart-dc1
spec:
  datacenter:
    name: dc1
    namespace: my-project
  jobs:
    - name: restart-dc1
      command: restart
```

```bash
kubectl apply -f restart-task.yaml
# Monitor status
kubectl get cassandratask restart-dc1 | yq .status
```

> ℹ️ `CassandraTask` targets one datacenter. Use `K8ssandraTask` to restart multiple DCs in a single manifest.

### Via `kubectl` — K8ssandraTask (multiple DCs)

```yaml
# restart-multi.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: K8ssandraTask
metadata:
  name: restart-dc
spec:
  cluster:
    name: production-hcd
    namespace: my-project
  datacenters:
    - dc1
    - dc2
  jobs:
    - name: restart-dcs
      command: restart
```

```bash
kubectl apply -f restart-multi.yaml
kubectl get k8ssandratask restart-dc | yq .status
```

> ⚠️ Submit the task object to the Kubernetes cluster where the specified datacenter is deployed.

### Parallel rack restarts (v1.18.0+)

By default, nodes restart sequentially within each rack. Set `maxConcurrentPods: 0` (all nodes in a rack restart at once) or a positive integer (limit concurrency) to speed up the operation.

```yaml
spec:
  cluster:
    name: production-hcd
    namespace: my-project
  datacenters:
    - dc1
  maxConcurrentPods: 0    # 0 = restart all pods in the rack simultaneously
  jobs:
    - name: restart-dc1
      command: restart
```

> ⚠️ Parallel restarts temporarily reduce availability within the affected rack. Ensure RF ≥ 3 before enabling.

### Restart a specific rack only (during CP outage)

```yaml
spec:
  datacenter:
    name: dc1
    namespace: my-project
  jobs:
    - name: restart-rack1
      command: restart
      args:
        - rack: rack1
```

---

## 5B. Upgrade SSTables

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/upgrade-sstables.html

Run this after a server version upgrade or when changing compression options. If SSTables are already at the current version, the command returns immediately with no action.

Mission Control iterates one rack at a time and triggers `nodetool upgradesstables` on one pod at a time. It reports task progress and status.

> ⚠️ **Performance note**: During an SSTable upgrade, additional disk pressure occurs while SSTables are rewritten (if required). Read performance improves after the format upgrade is complete.

### Via the UI

1. MC UI → **Home → Clusters** → your cluster namespace.
2. In the **Nodes** section of the **Overview** tab, select the target DC checkbox.
3. Click ⋮ **More Options** → **Upgrade sstables**.
4. For all DCs: select the **Name** checkbox → **Bulk Actions** → **Action type: Upgrade sstables** → **Run**.

### Via `kubectl`

```yaml
# upgradesstables-dc1.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: upgradesstables-dc1
spec:
  datacenter:
    name: dc1
    namespace: my-project          # adjust to your namespace
  jobs:
    - name: upgradesstables-dc1
      command: upgradesstables
      args:
        keyspace_name: my_keyspace  # optional — omit to upgrade ALL keyspaces
```

```bash
# Submit to the Kubernetes cluster where the data plane and datacenter are deployed.
# The namespace (-n) MUST be mission-control.
kubectl apply -f upgradesstables-dc1.yaml -n mission-control

# Monitor status
kubectl get cassandratask upgradesstables-dc1 | yq .status
```

**Key options:**
- `spec.jobs[0].command`: must be `upgradesstables`
- `spec.jobs[0].args.keyspace_name`: restrict to one keyspace; omit to upgrade all keyspaces

**Status output example** (task complete, 3 nodes upgraded):

```yaml
status:
  completionTime: "2025-08-28T18:07:08Z"
  conditions:
    - status: "True"
      type: Complete
  startTime: "2025-08-28T18:06:57Z"
  succeeded: 3
```

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

> ⚠️ MC **blocks termination** if any user keyspace still references the datacenter in its replication settings. You **must** remove ALL references from ALL user-defined keyspaces before removing the DC from the `MissionControlCluster` spec.

### Termination order (multi-DC / multi-cluster deployments)

To avoid cluster-wide downtime and instability, follow this order:

1. **One cluster at a time**: For multi-cluster deployments, terminate datacenters from one cluster before moving to the next.
2. **One datacenter at a time**: Terminate only one DC at a time within a cluster.
3. **DSE workload order** (DSE clusters only):
   - First: Analytics datacenters
   - Then: Cassandra (transactional) and Graph datacenters
   - Last: Search datacenters
   - For mixed workloads, assign priority by the highest-ranking workload in that DC (e.g., a SearchAnalytics DC is terminated before a Graph DC).

### Procedure

```sql
-- Step 1: For EVERY user keyspace, remove the DC from replication.
-- You must wait until active keyspaces become dormant before altering.
ALTER KEYSPACE my_keyspace WITH REPLICATION = {
  'class': 'NetworkTopologyStrategy',
  'dc1': 3    -- remove dc2 from this map
};

-- Repeat for ALL user-defined keyspaces.
-- System keyspaces (system_traces, system_distributed, system_auth,
-- dse_leases, dse_perf, dse_security): MC operators update these automatically.
```

```yaml
# Step 2: Remove the datacenter entry from MissionControlCluster
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1              # keep dc1
        # dc2 entry deleted from list
```

```bash
kubectl apply -f hcd-cluster.yaml
# MC verifies no user keyspaces reference dc2, then decommissions and deletes it.
```

### Monitor termination progress

```bash
# Check overall cluster status (shows ERROR column if blocked)
kubectl get k8ssandracluster CLUSTER_NAME

# Watch the datacenter-level status
kubectl get cassandradatacenter dc2 -n my-project -o yaml
# Termination is complete when all Decommission conditions.status are "False"

# Watch node logs during decommission
kubectl logs CLUSTER-dc2-rack1-sts-0 -c server-system-logger
```

### Error: keyspace still has replicas

If you see:

```
NAME    ERROR
demo    cannot decommission DC dc2: keyspace ks1 still has replicas on it
```

Run `ALTER KEYSPACE` for the offending keyspace (e.g., `ks1`) to remove the DC reference, then re-apply the `MissionControlCluster` manifest.

```sql
ALTER KEYSPACE ks1 WITH replication = {'class': 'NetworkTopologyStrategy', 'dc1': 3};
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

## 11A. Backup/Restore Compatibility

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/backup-restore-compatibility.html

### Backup types

**Full backup**: Captures all data at a specific point in time. Self-contained and independent of previous backups. Use for cross-version migrations or maximum restore flexibility.

**Differential backup**: Captures only data changed since the last full backup. More storage-efficient, but requires the base full backup to restore.

> ℹ️ When you trigger a differential backup, MC creates a *synthetic full backup* by merging the last full backup with all incremental changes. The immutable-file nature of SSTables makes this optimization possible.

### Version compatibility matrix

Same-version restores always work without additional steps. When restoring to a later version, you must run **Upgrade SSTables** (Section 5B) after the restore completes. Downgrades are **not supported**.

| Source version | Target version | Supported | Notes |
|---|---|---|---|
| Cassandra 3.11.x | Cassandra 3.11.x | ✅ Yes | Same version |
| Cassandra 3.11.x | Cassandra 4.0.x | ✅ Yes | Upgrade SSTables after restore |
| Cassandra 3.11.x | Cassandra 4.1.x | ✅ Yes | Upgrade SSTables after restore |
| Cassandra 4.0.x | Cassandra 4.0.x | ✅ Yes | Same version |
| Cassandra 4.0.x | Cassandra 4.1.x | ✅ Yes | Later minor version |
| Cassandra 4.0.x | HCD | ✅ Yes | Restore Cassandra 4.x backups to HCD |
| Cassandra 4.1.x | Cassandra 4.0.x | ❌ No | No downgrades |
| Cassandra 4.1.x | HCD | ✅ Yes | Restore Cassandra 4.x backups to HCD |
| Cassandra 4.x.x | Cassandra 3.11.x | ❌ No | No downgrades |
| DSE 6.8.x | DSE 6.8.x | ✅ Yes | Same version |
| DSE 6.8.x | DSE 6.9.x | ✅ Yes | Upgrade SSTables after restore |
| DSE 6.8.x | HCD | ✅ Yes | Validate schema during migration |
| DSE 6.9.x | DSE 6.9.x | ✅ Yes | Same version |
| DSE 6.9.x | HCD | ✅ Yes | Validate schema during migration |
| HCD 1.x | HCD 1.x | ✅ Yes | Same version |
| HCD 1.x | HCD 2.x | ✅ Yes | Upgrade SSTables after restore |
| HCD 2.x | HCD 2.x | ✅ Yes | Same version |

### SSTable upgrade after restore to a later version

When restoring to a newer major version with a different SSTable format, upgrade SSTables after the restore:

```yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: upgrade-sstables
spec:
  datacenter:
    name: dc1
  jobs:
    - name: upgradesstables
      command: upgradesstables
```

See **Section 5B** for the full procedure and status monitoring.

### Limitations

- Restoring to a newer **major** version requires SSTable upgrades.
- When restoring a DSE backup to a Cassandra or HCD cluster, DSE-specific features may not be available in the target cluster.
- **Materialized views** may need to be rebuilt after restore.
- If the source cluster used **custom compaction strategies**, the same strategies must be available in the target cluster before restore.
- **Time-series data with TTL** may expire during long restore operations.

### Schema and topology notes

- Backups capture schema along with data. Restoring reverts the schema to the backup point — columns dropped after the backup reappear; columns added after the backup do not appear.
- Reapply any schema changes (DDL) made after the backup was taken.
- Secondary indexes restore automatically; custom indexes require the same implementation in the target cluster.
- You can restore to a cluster with a different number of nodes, DCs, or rack topology. MC recalculates token ranges and handles rack assignment automatically.
- For multi-DC clusters, restore each DC's backup independently and verify schema consistency across all DCs.

### Security considerations

- **TDE-encrypted backups**: Require the same encryption keys for restore. Store keys securely, separate from backups.
- **Authentication and roles are NOT backed up**: After restore, manually re-create users and roles, reconfigure external auth providers, and verify user access.

### Post-restore checklist

1. Run `nodetool repair` for data consistency.
2. Run **Upgrade SSTables** (Section 5B) if restoring to a newer version.
3. Verify data integrity.
4. Re-create security settings (users, roles, external auth).
5. Update application connection strings if the cluster topology changed.
6. Adjust replication settings if the DC topology changed.

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
      serverVersion: "2.0.8"
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

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/upgrade-nosql-cluster.html

You can upgrade a DSE or HCD cluster with zero downtime by updating `serverVersion` in the `MissionControlCluster` manifest.

**Operator workflow:**
1. Cluster-level operator detects the version change and modifies each `CassandraDatacenter`, one at a time (for multi-DC clusters).
2. Datacenter-level operator processes each rack sequentially: modifies rack resource → terminates the instance → bootstraps a replacement node.
3. After upgrade completes, follow the **Upgrade SSTables** steps (Section 5B) if the new version uses a different SSTable format.

```yaml
# demo.yaml — change serverVersion only; other spec fields unchanged
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: production-hcd
spec:
  k8ssandra:
    cassandra:
      serverVersion: "2.0.8"   # was "2.0.5"
```

```bash
kubectl apply -f demo.yaml

# Monitor upgrade progress from the data plane cluster
kubectl get cassandradatacenter dc1 -o yaml
# cassandraOperatorProgress: Updating → watch conditions.type: Updating
# When complete, cass-operator sets conditions.status to False for Updating
kubectl get pods -n my-project -w
```

> ℹ️ Upgrades follow the same rolling process as config-change deployments. Use the canary pattern (Section 5) to validate a version change on one DC before promoting cluster-wide.

---

## 15. Manage Clusters During a Control Plane Outage

📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html

**When the Mission Control control plane is unavailable:**
- ✅ Scheduled backups **continue** running — Medusa runs inside each data-plane pod and executes independently of the CP.
- ❌ Repairs do **NOT** run by default — Reaper availability depends on your deployment. If Reaper is deployed only in the CP, repairs are unavailable. If Reaper is deployed in a data-plane datacenter (first DC with data nodes), repairs can still run in that DC.
- ❌ Multi-datacenter operations are unavailable.
- ✅ `cass-operator` continues managing individual datacenters in the data plane — you can interact with it directly.
- ❌ `K8ssandraCluster`-level operations are unavailable (the system ignores changes to `K8ssandraCluster` objects while CP is down).

> ⚠️ **Prerequisites before making changes**: have a backup of your current cluster configuration. Document all manual changes made during the outage so you can reconcile them when CP recovers.

### Verify control plane availability

```bash
kubectl cluster-info
# Expected when CP is down:
# Unable to connect to the server: dial tcp SERVER_IP:8443: i/o timeout
```

### Switch kubectl context to the data plane

```bash
# List available contexts
kubectl config get-contexts

# Switch to data plane
kubectl config use-context DATA_PLANE_CLUSTER_NAME
```

### View and modify CassandraDatacenter

```bash
# List all CassandraDatacenter resources
kubectl get cassandradatacenter -A

# View details
kubectl describe cassandradatacenter dc1 -n my-project

# Edit directly — cass-operator (running on the DP) will apply changes
kubectl edit cassandradatacenter dc1 -n my-project

# Or apply a YAML file
kubectl apply -f datacenter.yaml
```

### CassandraTask commands available during CP outage

All `CassandraTask` operations are handled by `cass-operator` on the data plane and work without CP connectivity:

| Command | Purpose |
|---------|---------|
| `restart` | Rolling restart of all pods in the DC |
| `cleanup` | Remove data not belonging to this node (after topology change) |
| `rebuild` | Stream data from a source DC |
| `replacenode` | Replace a failed node |
| `upgradesstables` | Upgrade SSTables to current version |
| `compaction` | Run a major compaction |
| `scrub` | Scrub SSTables (checks for corruption) |
| `move` | Move a token to a new range |
| `flush` | Flush all memtables to disk |
| `garbagecollect` | Remove deleted data from SSTables |
| `refresh` | Load new SSTables from disk (after manual copy) |

### Trigger a rolling restart during CP outage

```yaml
# restart-task.yaml
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: restart-task
spec:
  datacenter:
    name: dc1
    namespace: cass-operator
  jobs:
    - name: restart-dc1
      command: restart
```

```bash
kubectl apply -f restart-task.yaml
```

To restart a specific rack only, add an `args` entry to the job:

```yaml
      args:
        - rack: rack1
```

### Control CP recovery behavior with annotation

```bash
# Allow CP to overwrite local changes on recovery (default)
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=always

# Allow CP to overwrite only once (annotation is then removed)
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=once

# Prevent CP from overwriting local changes on recovery
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec-   # removes annotation
```

> ⚠️ Direct changes to a `CassandraDatacenter` remain active until either a new version is created on the CP, or the `cassandra.datastax.com/autoupdate-spec` annotation (`always` or `once`) explicitly allows overwriting. Document all manual changes made during the outage.

### Recovery checklist (after CP is restored)

1. Verify CP connectivity: `kubectl cluster-info`
2. Review any changes made during the outage.
3. Synchronize configurations if needed (update the `MissionControlCluster` manifest to match manual changes, or let `autoupdate-spec=always` reconcile automatically).
4. Test cluster functionality via `nodetool status`.
5. Update documentation with any permanent changes.

---

## 16. Migrate Existing Clusters to Mission Control

📖 Source: https://docs.datastax.com/en/mission-control/migrate/migration-overview.html
📖 Source: https://docs.datastax.com/en/mission-control/migrate/oss-cass-to-mission-control.html

> **Which migration path do you need?**
>
> | Your current setup | Migration path |
> |---|---|
> | **K8ssandra operator** managing your cluster | Load `09-k8ssandra-migration.md` — in-place operator adoption, no data migration. 📖 https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html |
> | **Standalone OSS Cassandra** cluster (not K8ssandra-managed) | Continue below — zero-downtime new DC strategy. 📖 https://docs.datastax.com/en/mission-control/migrate/oss-cass-to-mission-control.html |

### Zero-downtime: add a new MC-managed datacenter (OSS Cassandra → Mission Control)

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

> ℹ️ After migration, there are several users created automatically in new MC-managed clusters (superuser, Medusa, Reaper). If the cluster already existed before MC, you must manually re-create the Medusa and Reaper roles. See the OSS Cassandra migration guide for the full re-create procedure.

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
- **CRD reference** (pick the page version matching your installed release, e.g. `missioncontrolcluster-v1.21.0`, `backup-v1.21.0`, `k8ssandratask-v1.21.0`): https://docs.datastax.com/en/mission-control/crd-reference/crd-reference-overview.html
- Service accounts reference: https://docs.datastax.com/en/mission-control/reference/service-accounts.html
- **CQL gateway / CqlConnectivity / cqlsh console**: see `10-cql-access.md`
