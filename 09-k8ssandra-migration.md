# 09 — Migrate K8ssandra Clusters to Mission Control

📖 **Public docs (always cite this URL in responses):** https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

> ⚠️ **This guide is for clusters currently managed by the K8ssandra operator.**  
> If you have a standalone Apache Cassandra® cluster not managed by K8ssandra, see Section 16 of `06-cluster-ops.md` instead.

This is an **in-place operator adoption migration** — no data migration, no cluster recreation. Your existing Cassandra cluster, data, Reaper, and Medusa all remain intact. Only the management layer changes.

---

## Migration at a Glance

| Phase | What happens | Downtime? |
|-------|-------------|-----------|
| 1. Verify & back up config | Document cluster state, export YAML | None |
| 2. Scale down K8ssandra operators | Both operators scaled to 0 replicas | None — Cassandra keeps running |
| 3. Remove K8ssandra webhooks | Delete validating + mutating webhook configs | None |
| 4. Install Mission Control | Helm install on the same cluster | None |
| 5. Create `MissionControlCluster` | Wraps existing `K8ssandraCluster` spec | None |
| 6. Verify adoption | MC's embedded operator adopts the cluster | None |
| 7. Validate data integrity | Query data, run `nodetool status` | None |
| 8. Post-migration tasks | UI access, Reaper/Medusa checks, cleanup | None |

---

## How the Migration Works

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

1. Scale down the **standalone** K8ssandra operator deployments (not the Cassandra cluster itself).
2. Install Mission Control with its embedded `k8ssandra-operator`.
3. Mission Control's embedded operator **automatically adopts** and manages the existing `K8ssandraCluster` resources.

After migration, you manage clusters through `MissionControlCluster` objects instead of `K8ssandraCluster` objects directly. The underlying `K8ssandraCluster` resources still exist, but Mission Control owns and configures them.

> ⚠️ **Any direct changes made to a `K8ssandraCluster` object after migration will be reverted** the next time the Mission Control reconciler runs. Always edit the `MissionControlCluster` object instead.

There are no competing operators: Mission Control includes the same `cass-operator` that K8ssandra uses, plus additional management capabilities.

---

## Prerequisites

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

- An existing K8ssandra-managed cluster with:
  - K8ssandra operator installed
  - One or more `K8ssandraCluster` resources
  - Optional: Reaper and Medusa configured
- `kubectl` configured with cluster admin permissions
- An IBM entitlement key (required for Mission Control Helm install)
- Helm 3.8 or later (3.14.0–3.18.0 recommended; 3.19+ not yet supported)

---

## Phase 1 — Verify and Back Up Your Existing Cluster Configuration

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

Before migration, verify your K8ssandra clusters are healthy and document their configuration.

```bash
# List all K8ssandra clusters across namespaces
kubectl get k8ssandracluster --all-namespaces
```

```bash
# Check cluster status
kubectl describe k8ssandracluster CLUSTER_NAME -n NAMESPACE
```

```bash
# Verify all pods are running
kubectl get pods -n NAMESPACE
```

Document the following before proceeding:
- Cluster name and namespace
- Number of datacenters and nodes
- Cassandra version
- Reaper and Medusa configuration
- Storage class and volume sizes
- Superuser secret name

```bash
# Export the K8ssandraCluster spec — save this as your backup
kubectl get k8ssandracluster CLUSTER_NAME -n NAMESPACE -o yaml > k8ssandra-cluster-backup.yaml
```

---

## Phase 2 — Scale Down K8ssandra Operators

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

> ⚠️ **You must scale down BOTH the main K8ssandra operator and the `cass-operator` deployment.**  
> Scaling down only one results in competing operators.

```bash
# Scale down the main K8ssandra operator
kubectl scale deployment RELEASE_NAME -n K8SSANDRA_NAMESPACE --replicas=0

# Scale down cass-operator
kubectl scale deployment RELEASE_NAME-cass-operator -n K8SSANDRA_NAMESPACE --replicas=0
```

Replace `RELEASE_NAME` with the Helm release name used when installing the K8ssandra operator, and `K8SSANDRA_NAMESPACE` with the namespace where the K8ssandra operator is installed.

```bash
# Verify both operators are at 0/0 ready replicas
kubectl get deployments -n K8SSANDRA_NAMESPACE
```

Your Cassandra cluster continues running throughout this step. Only the operator management layer is affected.

---

## Phase 3 — Remove K8ssandra Webhooks

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

Remove K8ssandra webhooks to prevent validation conflicts with Mission Control's webhooks:

```bash
kubectl delete validatingwebhookconfiguration RELEASE_NAME-validating-webhook-configuration
kubectl delete mutatingwebhookconfiguration RELEASE_NAME-mutating-webhook-configuration
```

Replace `RELEASE_NAME` with the Helm release name for the original K8ssandra operator installation.

---

## Phase 4 — Install Mission Control

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html  
📖 Install guide: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html

Install Mission Control on the **same Kubernetes cluster** where K8ssandra was installed.

```bash
# Create the Mission Control namespace
kubectl create namespace NAMESPACE
```

Follow the standard Helm install procedure from the ICR registry. See `02-install-online.md` for the full Helm install steps.

```bash
# Verify Mission Control pods are running
kubectl get pods -n NAMESPACE
```

Wait for all Mission Control pods to reach `Running` state, then verify the embedded operators:

```bash
kubectl get deployment -n NAMESPACE | grep -E "operator|ui"
```

You should see all four of these deployments ready:
- `mission-control-k8ssandra-operator`
- `mission-control-cass-operator`
- `mission-control-operator`
- `mission-control-ui`

---

## Phase 5 — Create a MissionControlCluster Object

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

Create a `MissionControlCluster` object that wraps your existing `K8ssandraCluster` spec.

> ⚠️ **The `metadata.name` of the `MissionControlCluster` MUST exactly match the name of your existing `K8ssandraCluster`.** This is how Mission Control identifies and adopts the existing cluster.

```bash
# First, retrieve your existing K8ssandraCluster spec
kubectl get k8ssandracluster CLUSTER_NAME -n NAMESPACE -o yaml
```

Create a `MissionControlCluster` manifest embedding the `K8ssandraCluster` spec in the `spec.k8ssandra` field:

```yaml
# missioncontrolcluster.yaml
apiVersion: mission-control.k8ssandra.io/v1alpha1
kind: MissionControlCluster
metadata:
  name: CLUSTER_NAME        # MUST match your existing K8ssandraCluster name exactly
  namespace: NAMESPACE
spec:
  k8ssandra:
    # Paste the full spec section from your existing K8ssandraCluster here.
    # Omit: apiVersion, kind, metadata, and status fields.
    datacenters:
      - metadata:
          name: DATACENTER_NAME
        size: NODE_COUNT
        # ... remaining K8ssandraCluster spec fields
```

Replace:
- `CLUSTER_NAME` — the name of your existing K8ssandra cluster
- `NAMESPACE` — the namespace where your cluster is running
- `DATACENTER_NAME` — the name of your datacenter
- `NODE_COUNT` — the number of nodes in your datacenter

```bash
kubectl apply -f missioncontrolcluster.yaml

# Verify the object was created
kubectl get missioncontrolcluster CLUSTER_NAME -n NAMESPACE
```

---

## Phase 6 — Verify Cluster Adoption

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

```bash
# Monitor Mission Control k8ssandra-operator logs for adoption events
kubectl logs -f deployment/mission-control-k8ssandra-operator -n NAMESPACE
```

Look for log entries indicating the operator has discovered and is managing your `K8ssandraCluster`.

```bash
# Verify K8ssandraCluster status
kubectl get k8ssandracluster CLUSTER_NAME -n NAMESPACE

# Check CassandraDatacenter status — should show Ready
kubectl get cassandradatacenter -n NAMESPACE

# Verify all Cassandra pods are Running with all containers ready
kubectl get pods -n NAMESPACE | grep cassandra

# Verify cross-namespace RBAC is configured
kubectl get clusterrolebinding | grep mission-control
```

---

## Phase 7 — Validate Data Integrity

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

```bash
# Retrieve superuser credentials
CASS_USERNAME=$(kubectl get secret SUPERUSER_SECRET -n NAMESPACE -o jsonpath='{.data.username}' | base64 -d)
CASS_PASSWORD=$(kubectl get secret SUPERUSER_SECRET -n NAMESPACE -o jsonpath='{.data.password}' | base64 -d)
```

```bash
# Query a table to verify data is accessible
kubectl exec POD_NAME -n NAMESPACE -c cassandra -- \
  cqlsh -u $CASS_USERNAME -p $CASS_PASSWORD \
  -e "SELECT * FROM KEYSPACE.TABLE LIMIT 10;"
```

```bash
# Verify cluster health — all nodes should show UN (Up/Normal)
kubectl exec POD_NAME -n NAMESPACE -c cassandra -- nodetool status
```

```bash
# If using Reaper, verify it is still running
kubectl get pods -n NAMESPACE | grep reaper

# If using Medusa, verify it is still running
kubectl get pods -n NAMESPACE | grep medusa
```

---

## Phase 8 — Post-Migration Tasks

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

**Access the Mission Control UI:**

```bash
# See UI access instructions
# 📖 https://docs.datastax.com/en/mission-control/install/access-ui.html
```

**If using Reaper:**
- Access the Reaper UI or use the Mission Control UI to view repair schedules
- Verify existing repair schedules remain active
- Test running a new repair to confirm Reaper integration

**If using Medusa:**

```bash
# Check backup schedules remain active
kubectl get cassandratask -n NAMESPACE

# Test a new backup to confirm Medusa integration
```

Verify backup storage connectivity and run a test backup.

**Verify Mission Control observability components:**

```bash
kubectl get pods -n mission-control | grep -E "loki|mimir|minio"
```

**Remove original K8ssandra operator resources (after 30 days of stability):**

> ⚠️ **Only do this after your cluster has been stable under Mission Control management for at least 30 days.**

```bash
kubectl delete deployment K8SSANDRA_RELEASE_NAME -n K8SSANDRA_NAMESPACE
kubectl delete deployment K8SSANDRA_RELEASE_NAME-cass-operator -n K8SSANDRA_NAMESPACE
kubectl delete serviceaccount K8SSANDRA_RELEASE_NAME -n K8SSANDRA_NAMESPACE
kubectl delete clusterrolebinding K8SSANDRA_RELEASE_NAME
kubectl delete clusterrole K8SSANDRA_RELEASE_NAME
```

Replace `K8SSANDRA_RELEASE_NAME` with the Helm release name of your K8ssandra operator installation, and `K8SSANDRA_NAMESPACE` with the namespace where it was installed.

---

## Troubleshooting

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

### Competing Operators Error

If you see errors about competing operators or webhook conflicts:

```bash
# Verify both K8ssandra operators are at 0 replicas
kubectl get deployments -n K8SSANDRA_NAMESPACE

# Check for lingering K8ssandra webhooks
kubectl get validatingwebhookconfiguration | grep k8ssandra
kubectl get mutatingwebhookconfiguration | grep k8ssandra
```

> ⚠️ Only delete webhooks belonging to your **original K8ssandra release**.  
> Do NOT delete webhooks belonging to your new Mission Control release — that will break your migrated cluster.

```bash
# Delete any lingering original K8ssandra webhooks
kubectl delete validatingwebhookconfiguration RELEASE_NAME-validating-webhook-configuration
kubectl delete mutatingwebhookconfiguration RELEASE_NAME-mutating-webhook-configuration
```

---

### Pods Not Starting (Restart Loop)

```bash
# Check pod logs
kubectl logs POD_NAME -n NAMESPACE -c cassandra

# Verify Management API is responding
kubectl exec POD_NAME -n NAMESPACE -c cassandra -- \
  curl -s http://localhost:8080/api/v0/probes/readiness

# Check for resource constraints or storage issues
kubectl describe pod POD_NAME -n NAMESPACE
```

---

### Mission Control Pods Not Starting

```bash
# Check pod logs
kubectl logs -n mission-control POD_NAME

# Check PVC binding
kubectl get pvc -n mission-control

# Verify storage class configuration
kubectl get storageclass
```

---

### Cluster Not Being Adopted

```bash
# Verify the cluster name matches exactly — names are case-sensitive
kubectl get k8ssandracluster -n NAMESPACE

# Review Mission Control k8ssandra-operator logs
kubectl logs -n mission-control deployment/mission-control-k8ssandra-operator

# Check for RBAC issues
kubectl get clusterrolebinding | grep mission-control
```

The most common cause: the `metadata.name` in your `MissionControlCluster` does not exactly match the `K8ssandraCluster` name.

---

### Operator Version Drift

If you encounter compatibility issues due to operator version differences:

```bash
# Check the K8ssandra operator version you were using
kubectl get deployment RELEASE_NAME -n K8SSANDRA_NAMESPACE \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Mission Control bundles specific operator versions per release. As of **MC 1.21.0**:

| Operator | Bundled version |
|----------|----------------|
| `k8ssandra-operator` | 1.33.0 |
| `cass-operator` | 1.32.3 |

Review the Mission Control release notes for the operator version included in your target release:
📖 https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

---

## Rollback a Failed Migration

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

> ⚠️ Only perform a rollback if the migration has failed and your cluster is not functioning properly.

```bash
# Step 1: Scale down Mission Control operators
kubectl scale deployment mission-control-k8ssandra-operator -n mission-control --replicas=0
kubectl scale deployment mission-control-cass-operator -n mission-control --replicas=0
```

Step 2: Reinstall K8ssandra webhooks with the original configuration.

```bash
# Step 3: Scale K8ssandra operators back up
kubectl scale deployment RELEASE_NAME -n K8SSANDRA_NAMESPACE --replicas=1
kubectl scale deployment RELEASE_NAME-cass-operator -n K8SSANDRA_NAMESPACE --replicas=1
```

```bash
# Step 4: Verify K8ssandra resumes management of your cluster
kubectl get k8ssandraclusters -n NAMESPACE
kubectl logs deployment/RELEASE_NAME -n K8SSANDRA_NAMESPACE --tail=50
```

```bash
# Step 5 (permanent rollback): Remove Mission Control entirely
helm uninstall mission-control -n mission-control
kubectl delete namespace mission-control
```

```bash
# Step 6 (if needed): Restore K8ssandra configuration from backup
kubectl apply -f k8ssandra-cluster-backup.yaml
```

---

## Next Steps After Migration

📖 Source: https://docs.datastax.com/en/mission-control/migrate/k8ssandra-to-mission-control.html

- Explore Mission Control features: https://docs.datastax.com/en/mission-control/administration/control-plane/backup.html
- Configure monitoring and alerting: https://docs.datastax.com/en/mission-control/administration/observability/metrics.html
- Learn about cluster upgrades: https://docs.datastax.com/en/mission-control/administration/control-plane/upgrade-nosql-cluster.html
- Review cluster best practices: https://docs.datastax.com/en/mission-control/reference/cluster-best-practices.html
