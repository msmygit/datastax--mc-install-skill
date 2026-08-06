# 07 — Troubleshooting, Diagnostics, Support Bundle, Uninstall & DBA Pre-Install Checklist

📖 Source: https://docs.datastax.com/en/mission-control/frequently-asked-questions/faq.html  
📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/uninstall.html  
📖 Source: https://docs.datastax.com/en/mission-control/misc/supported-platforms.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html

---

## 1. DBA Pre-Install Checklist (Print and Work Through This Before Every Install)

This checklist condenses all the non-obvious gotchas that cause the most install failures:

```
PRE-FLIGHT
  [ ] IBMid registered and IBM entitlement key copied from IBM Container Library
  [ ] Kubernetes: version 1.21.0+ confirmed. NOT 1.35.0–1.35.3 (MaxUnavailableStatefulSet bug)
  [ ] Helm: version 3.14.0–3.18.0 ONLY. Run: helm version --short
  [ ] cert-manager: installed + all 3 pods Running. --enable-certificate-owner-ref=true set
  [ ] OpenShift: cert-manager from OperatorHub (NOT Helm)
  [ ] Platform nodes: labeled mission-control.datastax.com/role=platform (at least 2)
  [ ] Database nodes: labeled mission-control.datastax.com/role=database (at least 3 prod)
  [ ] Two SEPARATE object storage buckets exist (one Mimir metrics, one Loki logs — CANNOT share)
  [ ] StorageClass with volumeBindingMode: WaitForFirstConsumer exists and named correctly
  [ ] EKS: gp2 (or equivalent) set as default StorageClass before install
  [ ] cp.icr.io image pull secret created in the operator namespace (username `cp`, password = entitlement key)
  [ ] Release name chosen — does NOT contain "mission-control"
  [ ] For multi-region: same release name planned for CP and all DPs
  [ ] Ports open: 7000, 7001, 8080, 9042, 30880, 30600 between nodes/clusters
  [ ] If migrating from Replicated/KOTS: maintenance window scheduled (cass-operator restart triggers rolling DB pod restarts)

INSTALLATION
  [ ] overrides.yaml configures Loki storage (or disables it) and Dex with an auth connector — bare install fails validation
  [ ] Data plane Vector aggregator URL is set and reachable from DP before install
  [ ] For OpenShift: SCC grants applied to all 9 service accounts BEFORE helm install
  [ ] For OpenShift: dnsNamespace: openshift-dns + Mimir nginx resolver set (silent failure without it)
  [ ] For multi-region: Vector client TLS cert replicated to data plane BEFORE registering DP
  [ ] Medusa bucket secret created in cluster namespace BEFORE MissionControlCluster applied
  [ ] Per-node ConfigMaps (if using perNodeConfigMapRef) created BEFORE MissionControlCluster applied

POST-INSTALL
  [ ] All pods Running or Completed (kubectl get pods -n mission-control)
  [ ] nodetool status shows all nodes as UN (Up/Normal)
  [ ] Grafana dashboards loading and showing metrics/logs
  [ ] UI accessible at configured hostname or port 30880
```

---

## 2. Diagnostic Commands Reference

### Mission Control platform health

```bash
# Overall pod status
kubectl get pods -n mission-control

# Watch pods in real time
kubectl get pods -n mission-control -w

# MC operator logs (most useful for install failures and cluster creation issues)
kubectl logs -n mission-control deploy/mc-release-mission-control-operator --tail=100 -f

# k8ssandra-operator logs (datacenter lifecycle)
kubectl logs -n mission-control deploy/mc-release-k8ssandra-operator --tail=100

# cass-operator logs (StatefulSet management)
kubectl logs -n mission-control deploy/mc-release-cass-operator --tail=100

# UI logs
kubectl logs -n mission-control deploy/mc-release-mission-control-ui --tail=50

# Dex auth logs
kubectl logs -n mission-control deploy/mc-release-dex --tail=50

# Vector aggregator logs (if no metrics/logs appearing)
kubectl logs -n mission-control deploy/mc-release-aggregator --tail=100
```

### Cluster and datacenter health

```bash
# All CRD statuses across all namespaces
kubectl get MissionControlCluster -A
kubectl get CassandraDatacenter -A
kubectl get K8ssandraCluster -A

# Drill into a stuck datacenter
kubectl describe CassandraDatacenter dc1 -n my-project | grep -A20 "Events:"
kubectl describe MissionControlCluster production-hcd -n my-project | grep -A30 "Status:"

# Node-level Cassandra health
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- nodetool status
# All nodes should show: UN (Up Normal)
# If any show DN (Down), investigate that node's pod logs
```

### Pod failure diagnosis

```bash
# Why is a pod Pending?
kubectl describe pod POD_NAME -n mission-control
# Look at: Events section → "FailedScheduling", "Insufficient memory", etc.

# Why is a pod CrashLoopBackOff?
kubectl logs POD_NAME -n mission-control --previous

# OOMKilled events
kubectl get events -n mission-control --field-selector reason=OOMKilling

# Recent events sorted by time
kubectl get events -n mission-control --sort-by='.lastTimestamp' | tail -30
kubectl get events -n my-project --sort-by='.lastTimestamp' | tail -30

# Resource usage
kubectl top pod -n mission-control
kubectl top node
```

---

## 3. Common Problems and Fixes

📖 Source: https://docs.datastax.com/en/mission-control/frequently-asked-questions/faq.html

| Symptom | Root cause | Fix |
|---------|-----------|-----|
| Pods stuck `Pending` forever | No `platform`-labeled node exists, or node lacks resources | `kubectl describe pod` → check Events; verify node labels |
| `ImagePullBackOff` on entitled images (hcd, dse, cql-router, cqlsh) | Pull secret missing, wrong namespace, or bad entitlement key | Verify secret exists in `$MC_NAMESPACE`, decode and check credentials — see `02-install-online.md` Step C |
| `helm install` fails: "no connectors specified" | Dex has no auth connector configured | Add at least `enablePasswordDB: true` + a static password to `overrides.yaml` |
| `helm install` naming conflict error | Release name contains `mission-control` | Rename to `mc-release`, `mc-prod`, etc. |
| Loki/Mimir silently stop collecting on OCP | Missing `dnsNamespace: openshift-dns` + Mimir nginx resolver | Add OCP DNS config to `values.yaml` — see `04-cloud-config.md` |
| Authentication failure pulling from `cp.icr.io` | Username isn't exactly `cp`, or entitlement key has whitespace/expired | Regenerate key at IBM Container Library; recreate the secret |
| `WaitForFirstConsumer` not set on StorageClass | Volumes bind on wrong node → deadlock | Create a new StorageClass with `volumeBindingMode: WaitForFirstConsumer` |
| Data plane aggregator crashes (OOM) | No valid sink configured; buffers logs/metrics forever | Set `aggregator.customConfig.sinks.control_plane_aggregator.address` to reachable CP URL |
| OIDC login redirects to wrong URL | `ui.baseUrl` missing or mismatched | Set `ui.baseUrl` to exactly match the Ingress hostname |
| Cassandra nodes stuck `Pending` after cluster create | Medusa secret not pre-created | Create Medusa secret in cluster namespace BEFORE applying `MissionControlCluster` |
| Rancher webhook failures | NGINX 1MB body size limit | Add `nginx.ingress.kubernetes.io/proxy-body-size: "0"` annotation to Ingress |
| cert-manager secret not cleaned up (OCP) | OCP cert-manager Operator doesn't auto-delete secrets | Manually `oc delete secret SECRET_NAME -n NAMESPACE` after removing Certificate |
| Multi-region: data plane can't reach CP aggregator | mTLS Vector cert not replicated | Copy CP Vector client cert secret to DP namespace (see `05-security.md`) |
| K8s 1.35.x StatefulSet failures | `MaxUnavailableStatefulSet` feature flag bug | Avoid K8s 1.35.0–1.35.3; upgrade to 1.35.4+ or disable the flag |
| Helm 3.19+ fails | Not yet supported | Downgrade to Helm 3.18.x |
| Air-gap: image pull errors | Image not mirrored to private registry, or wrong tag | Verify all required images are in registry; check tags match release notes |
| OCP: pods remain after `helm uninstall` | SCC grants not removed first | Remove SCC grants before uninstalling (see Section 7 below) |
| EKS install stalls on PVC provisioning | No default StorageClass set | Patch gp2 as default before install |
| Post-migration: DB pods still on old images | `cass-operator` not restarted after Replicated→ICR migration | Restart `cass-operator` deployment, see `02-install-online.md` migration section |
| `NotEnoughSpaceToScaleDown` on DC scale-down | Remaining nodes lack disk for decommissioned data | Add more storage or reduce to fewer nodes in stages |
| Scale-up/down fails or is rejected | Size not a multiple of rack count | Always change size in multiples of rack count (e.g., 3, 6, 9 for a 3-rack DC) |
| MC blocks DC deletion | User keyspaces still reference the DC | `ALTER KEYSPACE` to remove the DC from replication first |
| No metrics above 40 nodes | Default Mimir rate limits too low | Raise `ingestion_rate` to 250000, `ingestion_burst_size` to 500000 — see `08-observability.md` |
| Grafana/metrics timeout at 45+ nodes | Vector not compressing; buffer overflow | Enable `compression: true` + `buffer.max_size: 2147483648` in Vector config — see `08-observability.md` |

---

## 4. Control Plane Outage: Emergency Operations

📖 Source: https://docs.datastax.com/en/mission-control/administration/interact-with-local-operators-during-failure.html

If the Mission Control control plane goes down, **your database clusters continue running**. `cass-operator` on the data plane keeps the StatefulSets healthy. Here's what you can and cannot do:

| Capability | CP Down? |
|-----------|----------|
| Scheduled backups (Medusa) | ✅ Continue |
| Active queries (Cassandra) | ✅ Continue |
| Repairs (Reaper) | ❌ Stop |
| MC UI / API | ❌ Unavailable |
| New cluster creation | ❌ Unavailable |
| Direct `kubectl` management of data plane | ✅ Available |

### Switch context and operate directly on the data plane

```bash
# Point kubectl at the data plane cluster
kubectl config use-context DP_WEST

# Check datacenter status directly
kubectl get CassandraDatacenter -n my-project

# Edit the CassandraDatacenter directly (cass-operator will act on changes)
kubectl edit CassandraDatacenter dc1 -n my-project
```

### Run CassandraTask operations without the control plane

All `CassandraTask` commands are executed by `cass-operator` and do not require CP:

```yaml
# emergency-restart.yaml — works even when CP is down
apiVersion: control.k8ssandra.io/v1alpha1
kind: CassandraTask
metadata:
  name: emergency-restart
  namespace: my-project
spec:
  datacenter:
    name: production-hcd-dc1
    namespace: my-project
  jobs:
    - command: restart      # or: cleanup, rebuild, upgradesstables, compaction,
      name: restart         #     scrub, move, flush, garbagecollect, refresh
  restartPolicy: Never
  ttlSecondsAfterFinished: 300
```

```bash
kubectl apply -f emergency-restart.yaml
kubectl describe CassandraTask emergency-restart -n my-project
```

### Control whether CP overwrites your changes on recovery

```bash
# DEFAULT: CP will overwrite local changes when it reconnects (safe for normal use)
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=always

# CP will overwrite local changes exactly once, then stops
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec=once

# Remove annotation: CP will NOT overwrite local changes on recovery
kubectl annotate CassandraDatacenter dc1 -n my-project \
  cassandra.datastax.com/autoupdate-spec-
```

> ⚠️ If you make emergency topology changes (add/remove nodes) during a CP outage, set `autoupdate-spec=once` so CP re-syncs once on recovery but then respects your new state.

---

## 5. Generate Support Bundle

📖 Source: https://docs.datastax.com/en/mission-control/misc/generating-support-bundle.html

Attach this bundle to every IBM Support ticket.

### Online (internet access available)
```bash
# Install the support-bundle kubectl plugin first (if not installed):
curl https://krew.sh | bash
kubectl krew install support-bundle

# Generate bundle (includes MC-specific checks when MC is installed)
kubectl support-bundle https://kots.io

# Generates: support-bundle-TIMESTAMP.tar.gz
# Submit to: https://www.ibm.com/mysupport
```

### Air-gapped (no internet)
```bash
# On an internet-connected machine: download the spec
curl -o support-bundle-spec.yaml https://kots.io

# Copy support-bundle-spec.yaml to the air-gapped server, then:
kubectl support-bundle ./support-bundle-spec.yaml
```

### What the bundle includes
- Pod logs for all MC components
- Pod descriptions and events
- Node resource state
- CRD status for all MC resources
- Helm release state
- cert-manager certificate status

---

## 6. Observability Troubleshooting

### No metrics in Grafana dashboards

```bash
# Check Mimir status
kubectl get pods -n mission-control | grep mimir
kubectl logs deploy/mc-release-mimir-query-frontend -n mission-control --tail=50

# Check Vector aggregator is receiving data
kubectl logs deploy/mc-release-aggregator -n mission-control --tail=100 | grep -i "error\|warn\|sink"

# Verify the metrics bucket is accessible
kubectl exec -it deploy/mc-release-mimir-ingester -n mission-control -- \
  curl -I https://s3.amazonaws.com/mc-metrics-prod/ 2>&1 | head -5
```

### No logs in Grafana Explore

```bash
# Check Loki pods
kubectl get pods -n mission-control | grep loki
kubectl logs deploy/mc-release-loki-write -n mission-control --tail=50

# OpenShift only: confirm DNS resolution is working
oc exec -it LOKI_POD -n mission-control -- \
  curl http://dns-default.openshift-dns.svc.cluster.local
```

### Metrics/logs stop at scale (40+ nodes)

See `08-observability.md` → Section 5 (Enterprise Scale) for Vector compression settings and Mimir rate limit increases required above 40 nodes.

---

## 7. Uninstall Mission Control

📖 Source: https://docs.datastax.com/en/mission-control/install/uninstall.html

> ⚠️ **Back up all data before uninstalling.** The process permanently deletes all MC data and configurations.

### Helm uninstall

```bash
# Uninstall the Helm release
helm uninstall mc-release -n mission-control

# Remove remaining PVCs
kubectl delete pvc --all -n mission-control

# Delete cluster namespaces (if you deployed global-scoped clusters)
kubectl delete namespace my-project

# Delete the MC namespace
kubectl delete namespace mission-control

# Remove the clusterconfigs CRD
kubectl delete crd clusterconfigs.datastax.com

# If installed with separate cluster resources, also remove those:
kubectl delete -f mission-control-cluster-resources.yaml --namespace mission-control
```

> If this cluster still has a leftover KOTS admin console from a pre-migration Replicated install that was never cleaned up, remove it directly with `kubectl` (not `kubectl kots remove`) — see the "Remove KOTS" step in `02-install-online.md`'s migration section.

### OpenShift: REMOVE SCC GRANTS FIRST (critical)

> ⚠️ If you skip this step, resources will hang during uninstall.

```bash
# Remove SCC grants BEFORE helm uninstall
for SA in loki mission-control mission-control-agent mission-control-aggregator \
  mission-control-cass-operator mission-control-dex \
  mission-control-k8ssandra-operator mission-control-kube-state-metrics; do
  oc adm policy remove-scc-from-user nonroot-v2 -z $SA -n mission-control
done

# Then proceed with Helm uninstall
helm uninstall mc-release -n mission-control
oc delete namespace mission-control
```

---

## 8. Accessing the MC CLI (mcctl)

📖 Source: https://docs.datastax.com/en/mission-control/install/access-cli.html

```bash
# Run mcctl commands via kubectl exec
kubectl exec -it deploy/mc-release-mission-control \
  -n mission-control -- mcctl version

# Common mcctl commands:
# Register a data plane:
kubectl exec -it deploy/mc-release-mission-control -n mission-control -- \
  mcctl register --source-context DP_WEST --dest-context CP_EAST

# List clusters:
kubectl exec -it deploy/mc-release-mission-control -n mission-control -- \
  mcctl get clusters
```

---

## 9. Reset the Mission Control UI Password

📖 Source: https://docs.datastax.com/en/mission-control/frequently-asked-questions/faq.html

### Via Helm (update values.yaml)

```bash
# Generate new bcrypt hash
echo NEW_PASSWORD | htpasswd -BinC 10 admin | cut -d: -f2

# Update the hash in your values.yaml under dex.config.staticPasswords[0].hash
# Then apply:
helm upgrade mc-release oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace mission-control -f values.yaml

# Restart Dex to apply the new password immediately
kubectl rollout restart deployment -n mission-control -l app.kubernetes.io/name=dex
```

---

## 10. IBM Support

📖 Source: https://docs.datastax.com/en/mission-control/misc/support.html

- Open support tickets: https://www.ibm.com/mysupport
- Only accounts with paid HCD or DSE plans can submit support tickets
- Always attach a support bundle (see Section 5 above) with every ticket
- For entitlement key issues, replacement, or Public Preview conversion: contact your IBM account team, or visit https://myibm.ibm.com/products-services/containerlibrary

---

## 11. Release Notes

📖 Source: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

Check release notes before every upgrade for:
- Exact image tags for all components in the new version
- Breaking changes or migration steps
- Supported upgrade paths (which previous versions can upgrade directly)
- Known issues and workarounds
