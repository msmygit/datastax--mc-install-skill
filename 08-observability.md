# 08 — Observability: Pod Sizing, Alerts, Scaling Tiers, External Sinks, Enterprise Scale

📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/pod-sizing-guidance.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/scale-components.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/external-metrics-logs.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/alerts.html  
📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/custom-alerting-rules.html

---

## 1. MC Platform Component Resource Requirements

📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/pod-sizing-guidance.html

Use these as your Kubernetes resource `requests` and `limits` for each MC platform component. Under-sizing these causes OOMKills and CrashLoopBackOff.

| Component | CPU Request | Mem Request | CPU Limit | Mem Limit | Replicas |
|-----------|------------|-------------|----------|----------|---------|
| `mission-control-operator` | 100m | 256Mi | 1000m | 512Mi | 1 |
| `cass-operator` | 200m | 64Mi | 1000m | 256Mi | 1 |
| `k8ssandra-operator` | 100m | 64Mi | 1000m | 256Mi | 1 |
| `mission-control-ui` | 100m | 256Mi | — | — | 2 (HA) |
| `dex` | 50m | 128Mi | — | — | 2 (HA) |
| `medusa` (sidecar, per node) | 100m | 256Mi | — | — | 1 per DB pod |
| `reaper` (per cluster) | 200m | 512Mi | — | — | 1 |
| `stargate` | 500m | 1Gi | — | — | 2 (HA) |
| `vector` (sidecar, per pod) | 50m | 128Mi | — | — | 1 per DB pod |
| `cqlsh` pod | 100m | 256Mi | — | — | on-demand |
| `cert-manager` | 10m | 32Mi | — | — | 1 |

> ℹ️ The UI and Dex should run 2 replicas for high availability of the control plane dashboard and authentication.

### Configure resource overrides in values.yaml

```yaml
# Example: bump the operator memory limit for large deployments
mission-control-operator:
  resources:
    requests:
      cpu: "200m"
      memory: "512Mi"
    limits:
      cpu: "2"
      memory: "1Gi"
```

---

## 2. Default Alerting Rules

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/alerts.html

MC ships with the following default Prometheus alerts (all operate on Cassandra/HCD cluster metrics):

| Alert | Threshold | Severity | Description |
|-------|-----------|---------|-------------|
| Node Down (warning) | >10 min | warning | A Cassandra node has been down for more than 10 minutes |
| Node Down (error) | >30 min | error | A Cassandra node has been down for more than 30 minutes |
| Nodes Down Across Racks | Multi-rack | error | Nodes down across multiple racks — LOCAL_QUORUM at risk |
| High CPU | >80% for 5 min | warning | Average CPU usage exceeds 80% for 5 consecutive minutes |
| Disk Usage (warning) | >50% | warning | Data volume disk usage exceeds 50% |
| Disk Usage (error) | >75% | error | Data volume disk usage exceeds 75% — urgent: add capacity |
| High Load Average (warning) | >20 | warning | 1-minute load average exceeds 20 |
| High Load Average (error) | >32 | error | 1-minute load average exceeds 32 |
| Dropped Messages | Any | warning | Cassandra has dropped messages (queue overload indicator) |

> ℹ️ Since MC v1.15, default and custom alerting rules are separated into distinct ConfigMaps. Default rules are managed by MC; custom rules live in a separate ConfigMap you manage.

### Configure Slack webhook notifications

```yaml
# In values.yaml — route alerts to a Slack channel
alertmanager:
  config:
    receivers:
      - name: slack-ops
        slack_configs:
          - api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
            channel: "#cassandra-alerts"
            title: '{{ .CommonAnnotations.summary }}'
            text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    route:
      receiver: slack-ops
      group_by: [alertname, cluster, namespace]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 3h
```

---

## 3. Custom Alerting Rules

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/custom-alerting-rules.html

Write PromQL-based alerts for cluster-specific thresholds:

```yaml
# custom-alerts-configmap.yaml
# Since v1.15, create this separately — MC will NOT overwrite it
apiVersion: v1
kind: ConfigMap
metadata:
  name: mc-custom-alerting-rules
  namespace: mission-control
  labels:
    mc-custom-alerts: "true"    # MC detects this label to load the rules
data:
  custom-rules.yaml: |
    groups:
      - name: custom-cassandra
        rules:
          # Alert when pending compactions exceed threshold
          - alert: HighPendingCompactions
            expr: cassandra_table_estimated_pending_flushes > 100
            for: 10m
            labels:
              severity: warning
            annotations:
              summary: "High pending compactions on {{ $labels.instance }}"
              description: "Pending compactions: {{ $value }}"

          # Alert when read latency p99 > 100ms
          - alert: HighReadLatency
            expr: cassandra_table_read_latency_99th_percentile > 100000
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High read latency p99 on {{ $labels.table }}"
              description: "p99 read latency: {{ $value }}µs"
```

```bash
kubectl apply -f custom-alerts-configmap.yaml
# Prometheus picks up the new rules within ~30 seconds
```

---

## 4. Observability Scaling Tiers

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/scale-components.html

Choose the tier that matches your cluster size:

| Tier | DB Nodes | Mimir Ingesters | Mimir Distributors | Loki Writers | Loki Readers |
|------|----------|-----------------|-------------------|-------------|-------------|
| Small | ≤ 10 | 2 | 2 | 2 | 1 |
| Medium | 11–30 | 3 | 3 | 3 | 2 |
| Large | 31–50 | 5 | 5 | 4 | 3 |
| Enterprise | 50+ | Formula-based | Formula-based | Formula-based | Formula-based |

### Small tier (≤ 10 DB nodes)

```yaml
# values-small.yaml
mimir:
  ingester:
    replicas: 2
    resources:
      requests:
        cpu: "200m"
        memory: "512Mi"
  distributor:
    replicas: 2

loki:
  write:
    replicas: 2
  read:
    replicas: 1

aggregator:
  resources:
    requests:
      cpu: "50m"
      memory: "256Mi"
```

### Medium tier (11–30 DB nodes)

```yaml
# values-medium.yaml
mimir:
  ingester:
    replicas: 3
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
  distributor:
    replicas: 3

loki:
  write:
    replicas: 3
  read:
    replicas: 2

aggregator:
  resources:
    requests:
      cpu: "100m"
      memory: "512Mi"
```

### Large tier (31–50 DB nodes)

```yaml
# values-large.yaml
mimir:
  ingester:
    replicas: 5
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
  distributor:
    replicas: 5

loki:
  write:
    replicas: 4
  read:
    replicas: 3

grafana:
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"

aggregator:
  resources:
    requests:
      cpu: "200m"
      memory: "1Gi"
```

---

## 5. Enterprise Scale: 45+ Nodes (Critical Configuration)

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/scale-components.html

> ⚠️ Without the changes in this section, clusters with **45+ DB nodes** will experience cascading timeout failures in Vector, Mimir HTTP 429 rate limit errors, and Grafana dashboards going dark. These are NOT optional optimizations — they are required for production scale.

### Capacity planning formulas

```
Mimir Ingester replicas  = ceil(Total time series / 240,000)
                           e.g., 60 nodes × 2,000 series/node = 120k series → 1 ingester (min 3 for HA)

Mimir Distributor replicas = ceil((Total ingestion rate / 50,000) × 2)
                              e.g., 60 nodes × 500 samples/s = 30k/s → (30k/50k)×2 = 1.2 → 2 (min 2)
```

### Vector: Enable compression and increase buffer size

Without compression, Vector buffers metrics until it OOMs or timeouts cascade:

```yaml
# In values.yaml or values-enterprise.yaml
aggregator:
  customConfig:
    transforms:
      compress_metrics:
        type: remap
        inputs: ["internal_metrics"]
        source: ". = {}"    # placeholder — see docs for full compression config
    # Enable compression on the Vector sink sending to Mimir
    sinks:
      mimir:
        compression: true             # REQUIRED at 45+ nodes
        buffer:
          type: disk
          max_size: 2147483648        # 2 GB disk buffer (vs 256MB default)
          when_full: drop_oldest
```

### Mimir: Raise ingestion rate limits

Default Mimir rate limits cause HTTP 429 errors (silent drops) at 40+ nodes:

```yaml
# In values.yaml
mimir:
  mimir:
    structuredConfig:
      limits:
        ingestion_rate: 250000         # default is 10,000 — raise to 250,000
        ingestion_burst_size: 500000   # default is 50,000 — raise to 500,000
        max_global_series_per_user: 0  # 0 = unlimited
        max_fetched_chunks_per_query: 2000000
  ingester:
    replicas: 6                        # 60 nodes ÷ 240k series = 1, but use 6 for HA + headroom
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        memory: "8Gi"
  distributor:
    replicas: 4
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
```

### Loki: Scale for log ingestion at 50+ nodes

```yaml
loki:
  write:
    replicas: 6
    resources:
      requests:
        cpu: "1"
        memory: "2Gi"
  read:
    replicas: 4
    resources:
      requests:
        cpu: "500m"
        memory: "1Gi"
  limits_config:
    ingestion_rate_mb: 32              # raise from default 4MB/s
    ingestion_burst_size_mb: 64
    per_stream_rate_limit: "8MB"
    per_stream_rate_limit_burst: "16MB"
```

### Apply the enterprise configuration

```bash
helm upgrade mc-release \
  oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace mission-control \
  -f values.yaml \
  -f values-enterprise.yaml

# Verify Mimir ingesters come up healthy
kubectl get pods -n mission-control | grep mimir-ingester
kubectl logs deploy/mc-release-mimir-ingester -n mission-control --tail=50 | grep -i "error\|rate"
```

---

## 6. External Observability Sinks

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/external-metrics-logs.html

Forward metrics and/or logs from MC to an external platform alongside (or instead of) the built-in Mimir/Loki stack.

Default Vector sources available as inputs:
- `vector` — all metrics from MC components
- `internal_metrics` — Vector's own health metrics
- `kube_state_metrics` — Kubernetes object state metrics
- `cass_operator_metrics` — cass-operator specific metrics
- `vector_with_defaults` — logs (use for log sinks)

### Prometheus Remote Write

```yaml
aggregator:
  customConfig:
    sinks:
      prometheus_remote:
        type: prometheus_remote_write
        inputs: ["vector", "kube_state_metrics", "cass_operator_metrics"]
        endpoint: "https://prometheus.example.com/api/v1/write"
        auth:
          strategy: bearer
          token: "YOUR_BEARER_TOKEN"
        tls:
          verify_certificate: true
```

### New Relic

```yaml
aggregator:
  customConfig:
    sinks:
      new_relic_metrics:
        type: new_relic
        inputs: ["vector", "internal_metrics"]
        license_key: "YOUR_NEW_RELIC_LICENSE_KEY"
        api: metrics                   # or: logs, events
        region: us                     # or: eu
```

### Elasticsearch (for logs)

```yaml
aggregator:
  customConfig:
    sinks:
      elasticsearch_logs:
        type: elasticsearch
        inputs: ["vector_with_defaults"]    # log source
        endpoints:
          - "https://elasticsearch.example.com:9200"
        auth:
          strategy: basic
          user: "elastic"
          password: "YOUR_PASSWORD"
        index: "mission-control-logs-%F"
        tls:
          verify_certificate: true
```

### Datadog

```yaml
aggregator:
  customConfig:
    sinks:
      datadog_metrics:
        type: datadog_metrics
        inputs: ["vector", "kube_state_metrics"]
        api_key: "YOUR_DATADOG_API_KEY"
        site: "datadoghq.com"              # or: datadoghq.eu, us3.datadoghq.com
      datadog_logs:
        type: datadog_logs
        inputs: ["vector_with_defaults"]
        api_key: "YOUR_DATADOG_API_KEY"
        site: "datadoghq.com"
```

### Multiple sinks simultaneously

You can forward to multiple destinations at the same time — just list all sinks:

```yaml
aggregator:
  customConfig:
    sinks:
      mimir: {}              # built-in Mimir (keep this unless you're replacing it)
      loki: {}               # built-in Loki (keep this unless you're replacing it)
      prometheus_remote:     # also forward to external Prometheus
        type: prometheus_remote_write
        inputs: ["vector"]
        endpoint: "https://prometheus.example.com/api/v1/write"
      datadog_logs:          # also forward logs to Datadog
        type: datadog_logs
        inputs: ["vector_with_defaults"]
        api_key: "YOUR_DATADOG_API_KEY"
        site: "datadoghq.com"
```

---

## 7. Key Observability Metrics to Monitor

📖 Source: https://docs.datastax.com/en/mission-control/administration/observability/metrics.html

Critical metrics exposed by MC that every DBA should watch:

| Metric | What it signals |
|--------|----------------|
| `cassandra_table_read_latency_99th_percentile` | Read tail latency per table |
| `cassandra_table_write_latency_99th_percentile` | Write tail latency per table |
| `cassandra_table_estimated_pending_flushes` | Memtable flush backpressure |
| `cassandra_storage_total_disk_space_used` | Total disk used by Cassandra data |
| `cassandra_compaction_pending_tasks` | Compaction queue depth |
| `cassandra_dropped_messages_total` | Dropped messages (queue overflow) |
| `cassandra_nodes_down` | Number of down nodes in the cluster |
| `jvm_memory_bytes_used` | JVM heap usage per pod |
| `jvm_gc_collection_seconds_sum` | GC pause time |
| `cassandra_thread_pool_active_tasks` | Thread pool saturation |

### Access Grafana dashboards

```bash
# Port-forward Grafana if not using Ingress
kubectl port-forward -n mission-control svc/mc-release-grafana 3000:3000

# Access at: http://localhost:3000
# Default credentials: admin / (set in values.yaml or generated)
```

---

## 8. Viewing Logs in Grafana Explore

MC collects logs from all Cassandra, HCD, and platform pods into Loki. Access via:

1. **Grafana → Explore** (left panel, compass icon)
2. Select **Loki** as the data source
3. Query examples:
   ```
   # All logs from a specific cluster
   {namespace="my-project", app="cassandra"}

   # ERROR and WARN logs only
   {namespace="my-project"} |= "ERROR" or |= "WARN"

   # Startup errors
   {namespace="my-project"} |~ "Exception|ERROR" | logfmt

   # Logs from a specific pod
   {pod="production-hcd-dc1-r1-sts-0", namespace="my-project"}
   ```

---

## See Also

- Metrics reference: https://docs.datastax.com/en/mission-control/administration/observability/metrics.html
- Logs reference: https://docs.datastax.com/en/mission-control/administration/observability/logs.html
- Pod sizing guidance: https://docs.datastax.com/en/mission-control/administration/mc/pod-sizing-guidance.html
- Custom alerting rules: https://docs.datastax.com/en/mission-control/administration/observability/custom-alerting-rules.html
- Scale observability components: https://docs.datastax.com/en/mission-control/administration/observability/scale-components.html
