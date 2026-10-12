# 10 — CQL Access: Gateways, Console, and External Connectivity

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html
📖 Source: https://docs.datastax.com/en/mission-control/databases/use-cql-console.html
📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html

## Overview

Mission Control exposes Apache Cassandra® database clusters to external clients through **CQL gateways** — lightweight router pods (the `cql-router` component) that proxy the CQL native protocol. The UI also includes an embedded **CQL Console** (`cqlsh` backed by a `cqlsh-pod`) for direct in-browser interaction.

This topic covers:
1. What CQL gateways are and when to use them
2. Prerequisites
3. Creating an Ingress gateway through the UI
4. DNS configuration for the gateway
5. Connecting with the Secure Connect Bundle (SCB)
6. Verifying gateway status and logs
7. Troubleshooting Ingress gateways
8. The `CqlConnectivity` CR shape (for `kubectl`/GitOps)
9. Security-context overrides for the `CqlConnectivity` CR
10. The CQL Console and `cqlsh-pod` customization
11. TLS / cert secrets for `cql-router`

---

## 1. What Are CQL Gateways?

CQL gateways provide **external access** to database clusters through the CQL native protocol. This enables connections through tools such as Apache Cassandra® drivers, `cqlsh`, and driver-based tools.

Internally, each gateway is a `cql-router` pod. Mission Control creates a `CqlConnectivity` custom resource that describes how a datacenter is exposed outside the Kubernetes cluster. There are two connectivity types:

| Type | Description |
|------|-------------|
| **Ingress** | Routes traffic by domain name through your Kubernetes Ingress controller (port 443, TLS passthrough with SNI). Recommended for most deployments. |
| **LoadBalancer** | Creates one or more `LoadBalancer` services on the native CQL port (default 9042). Configured via the `CqlConnectivity` CR's `spec.loadBalancers` field. |

> **Tip — performance:** Deploy at least three gateway replicas for high availability. Configure appropriate resource limits for gateway pods. Use connection pooling in client applications.

---

## 2. Prerequisites

- A running database cluster managed by Mission Control
- Access to the Mission Control UI
- An Ingress controller installed in your Kubernetes cluster (see [04-cloud-config.md](04-cloud-config.md) and [ingress.adoc](https://docs.datastax.com/en/mission-control/install/ingress.html))
- DNS configuration access

---

## 3. Create an Ingress Gateway (UI)

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html

1. In the Mission Control UI, select your project, then select your database cluster.
2. Click **Connect**, then click **Add Gateway**.
3. In the list of datacenters, select the datacenter you want to expose.
4. For **Connectivity Type**, select **Ingress**.
5. Enter the number of **replicas**.
   > Triple replication (≥ 3) is recommended for production to ensure high availability.
6. For **DNS Base Name**, enter the gateway domain name. For example: `dc1.cluster.ingress.example.io`
7. Click **Add Gateway**.

Mission Control creates an Ingress resource with the specified domain name. The configuration is represented as:

```json
{
  "name": "cql-gateway",
  "type": "Ingress",
  "size": 3,
  "dnsBaseName": "cql.example.com"
}
```

---

## 4. DNS Configuration

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html

### Step 1: Find the Ingress controller's external IP or hostname

```bash
kubectl get service -n <INGRESS_NAMESPACE> <HAPROXY_SERVICE_NAME>
```

Replace:
- `<INGRESS_NAMESPACE>`: the namespace where your Ingress controller is deployed
- `<HAPROXY_SERVICE_NAME>`: the name of the HAProxy service. To find it, run:

```bash
kubectl get services -n <INGRESS_NAMESPACE>
```

### Step 2: Create the DNS record

If you are **not** using wildcard DNS, create a DNS A record or CNAME pointing your gateway domain to the Ingress controller's external IP or hostname.

For example, point `cql.example.com` to the external IP of the Ingress controller.

If you are using **wildcard DNS** (e.g. `*.mc.example.com`), this step is not required — all subdomains automatically resolve to the configured address.

### Step 3: Verify DNS resolution

```bash
nslookup <DNS_BASE_NAME>
```

### Step 4: Test TLS connectivity

The Ingress gateway uses TLS passthrough on port 443. The SNI hostname routes each connection to the correct `cql-router` pod.

```bash
echo | openssl s_client -connect <CLUSTER_DOMAIN_NAME>:443 -servername <HOST_ID>
```

Replace:
- `<CLUSTER_DOMAIN_NAME>`: the domain name of your Ingress controller (listens on port 443)
- `<HOST_ID>`: the host ID used as the SNI hostname to route the connection to the correct CQL router

---

## 5. Connect with the Secure Connect Bundle (SCB)

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html

### What the SCB contains

The Secure Connect Bundle (SCB) is a zip file that contains everything a client application needs to connect securely to a CQL gateway:

| File | Purpose |
|------|---------|
| `config.json` | Connection configuration: host, port, datacenter, keystore passwords |
| `ca.crt` | Certificate Authority certificate for TLS verification |
| `cert` | Client certificate for authentication |
| `key` | Private key for the client certificate |
| `identity.jks` | Java KeyStore containing the client identity |
| `trustStore.jks` | Java TrustStore containing trusted CA certificates |

**Important notes:**
- Each SCB is associated with **one** CQL gateway. For multiple gateways (e.g., different datacenters), download the SCB for each separately and add selection logic in your application.
- SSL encryption is **automatically enabled** for driver connections using the SCB. Do not specify separate SSL options.
- The SCB **never expires** after download. Delete it if no longer needed.
- SCB connections require username/password authentication. Kerberos authentication is not compatible.
- Treat the SCB as sensitive — it contains TLS certificates and connection credentials.

### Download the SCB

1. In the Mission Control UI, navigate to your cluster's **Connect** tab.
2. Click **Gateway**.
3. Select the gateway you want to connect to.
4. Click **Download Secure Connect Bundle**.

### Get your database credentials

```bash
kubectl get secret <CLUSTER_NAME>-superuser -n <PROJECT_SLUG> \
  -o jsonpath='{.data.username}' | base64 --decode

kubectl get secret <CLUSTER_NAME>-superuser -n <PROJECT_SLUG> \
  -o jsonpath='{.data.password}' | base64 --decode
```

Replace `<CLUSTER_NAME>` and `<PROJECT_SLUG>` with your values.

### Use the SCB in your driver

Pass the SCB zip file to your driver's connection configuration along with the username and password. **Do not** specify contact points or SSL options — the SCB provides these automatically.

Some drivers refer to the SCB connection as a _cloud_ connection (originally designed for Astra DB). If using the `gocql` driver, you must use `gocql-astra` instead (adds SCB support to `gocql`).

For driver-specific examples, see:
- HCD: https://docs.datastax.com/en/hyper-converged-database/drivers/cassandra-drivers-overview.html
- DSE: https://docs.datastax.com/en/dse/drivers/cassandra-drivers-overview.html

---

## 6. Gateway Status and Logs

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html

In the following commands, replace `<PROJECT_SLUG>` with your project namespace.

**Check gateway pod status:**
```bash
kubectl get pods -l app=cql-gateway -n <PROJECT_SLUG>
```

**Stream gateway logs:**
```bash
kubectl logs -l app=cql-gateway -n <PROJECT_SLUG>
```

---

## 7. Troubleshoot Ingress Gateways

📖 Source: https://docs.datastax.com/en/mission-control/databases/cql-gateway-connectivity.html

If Ingress routing does not work, follow these steps:

**1. Verify Mission Control created the Ingress resource:**
```bash
kubectl get ingress -n <PROJECT_SLUG>
```

**2. Check Ingress controller logs:**
```bash
kubectl logs -n <INGRESS_NAMESPACE>
```

**3. Verify TLS passthrough is enabled on the Ingress controller:**
```bash
kubectl get ingress -n <INGRESS_NAMESPACE> -o=jsonpath='{.items[*].spec.tls}'
```

**4. Confirm the Ingress controller supports TCP services:**
```bash
echo | openssl s_client -connect <CLUSTER_DOMAIN_NAME>:443 -servername <HOST_ID>
```

Replace `<CLUSTER_DOMAIN_NAME>` and `<HOST_ID>` as described in [Section 4](#4-dns-configuration).

---

## 8. `CqlConnectivity` CR (kubectl / GitOps)

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

The `CqlConnectivity` CR (`apiVersion: missioncontrol.datastax.com/v1alpha1`) describes how a datacenter is exposed outside the cluster. The UI creates this CR automatically; you can also manage it directly.

### Key spec fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `spec.cassandraDatacenterRef.name` | string | true | Name of the `CassandraDatacenter` to expose |
| `spec.ingress` | object | false | Ingress-based connectivity settings |
| `spec.loadBalancers.size` | integer | **true** (for LB mode) | Number of LoadBalancer services to create (minimum 1) |
| `spec.loadBalancers.nativePort` | integer | false | Port for the CQL native protocol on the service(s). Default: `9042` |
| `spec.loadBalancers.serviceConfig` | object | false | Common config for all LoadBalancer services |
| `spec.loadBalancers.podConfig` | object | false | Common config for all `cql-router` pods backing LoadBalancer services |
| `spec.loadBalancers.podConfigs[]` | array | false | Per-pod overrides (can have fewer elements than `size`) |
| `spec.encryption` | object | false | Certificate settings for the CQL connectivity |

The `CqlPod` (i.e., `podConfig`) controls settings for the `cql-router` pod backing each LoadBalancer service: `affinity`, `containerSecurityContext`, `podSecurityContext`, `resources`, and `tolerations`.

---

## 9. Security-Context Overrides for `CqlConnectivity`

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Override `containerSecurityContext` and `podSecurityContext` on the `CqlConnectivity` CR for hardened environments:

```yaml
apiVersion: missioncontrol.datastax.com/v1alpha1
kind: CqlConnectivity
metadata:
  name: <CQL_CONNECTIVITY_NAME>
spec:
  cassandraDatacenterRef:
    name: <DATACENTER_NAME>
  loadBalancers:
    nativePort: 9042
    serviceConfig: {}
    size: 2
    podConfig:
      containerSecurityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop:
            - ALL
        privileged: false
        readOnlyRootFilesystem: true
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
      podSecurityContext:
        fsGroup: 999
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
```

Apply with:
```bash
kubectl apply -f <CQL_CONNECTIVITY.yaml>
```

⚠️ These examples show the structure of security-context settings; use specific values that match your organization's security policies.

---

## 10. CQL Console and `cqlsh-pod`

📖 Source: https://docs.datastax.com/en/mission-control/databases/use-cql-console.html

### About the CQL Console

The Mission Control UI includes an embedded **CQL Console** — a `cqlsh` interface that lets you run CQL and `cqlsh` commands directly from the browser. You can create keyspaces, insert data, and query data.

Supported references for available commands:
- **HCD**: HCD `cqlsh` commands reference and HCD CQL commands reference
- **DSE**: DSE `cqlsh` commands reference and DSE CQL commands reference
- **Apache Cassandra®**: https://cassandra.apache.org/doc/5.0/cassandra/managing/tools/cqlsh.html

### Connect to the CQL Console

1. In the Mission Control UI navigation menu, select your project, then select your database cluster.
2. Click **CQL Console**.
3. Select your datacenter from the list of available datacenters.
4. Enter your username and password.

### The `cqlsh-pod` container

When you deploy a Mission Control cluster, Mission Control automatically creates a `cqlsh-pod` to support the CQL Console. The pod serves as the backend for the embedded `cqlsh` UI.

You can connect to the pod manually:
```bash
kubectl exec -it <cqlsh-pod-name> -n <PROJECT_SLUG> -- cqlsh
```

The pod has the following labels (useful for selection with `-l`):

| Label | Value |
|-------|-------|
| `app.kubernetes.io/name` | `cqlsh` |
| `app.kubernetes.io/part-of` | `missioncontrol` |
| `cassandra.datastax.com/datacenter` | `<DATACENTER_NAME>` |
| `missioncontrol.datastax.com/cluster-name` | `<CLUSTER_NAME>` |
| `missioncontrol.datastax.com/cluster-namespace` | `<PROJECT_SLUG>` |

> The `cqlsh-pod` image is included in the Mission Control airgap bundle. For non-airgap installs, it is pulled through an authenticated proxy. The image is not published on Docker Hub (proprietary).

### Customize the `cqlsh-pod`

📖 Source: https://docs.datastax.com/en/mission-control/databases/use-cql-console.html

Customize the `cqlsh-pod` using `spec.cqlsh.podTemplateSpec` in the `MissionControlCluster` CR. The operator merges your values with its own generated configuration — you only need to specify the fields you want to change.

Customizable aspects:
- **Container image**: point to a custom registry/repository/tag
- **Resource management**: `resources.requests` and `resources.limits`
- **Security hardening**: `securityContext` at container and pod level
- **Scheduling control**: `nodeSelector`, `affinity`, `tolerations`
- **Environment variables**: `env`
- **Volume mounts**: additional volumes and config files

**Full customization template:**

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: <CLUSTER_NAME>
spec:
  cqlsh:
    podTemplateSpec:
      spec:
        containers:
        - name: cqlsh
          image: <CUSTOM_IMAGE_REGISTRY>/<CUSTOM_CQLSH_IMAGE>:<TAG>
          resources:
            requests:
              memory: "<MEMORY_REQUEST>"
              cpu: "<CPU_REQUEST>"
            limits:
              memory: "<MEMORY_LIMIT>"
              cpu: "<CPU_LIMIT>"
          env:
          - name: <CUSTOM_ENV_VAR_NAME>
            value: "<CUSTOM_ENV_VAR_VALUE>"
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
              - ALL
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: <USER_ID>
        securityContext:
          fsGroup: <GROUP_ID>
          runAsGroup: <GROUP_ID>
          runAsNonRoot: true
          runAsUser: <USER_ID>
        nodeSelector:
          <NODE_LABEL_KEY>: "<NODE_LABEL_VALUE>"
        tolerations:
        - key: "<TAINT_KEY>"
          operator: "Equal"
          value: "<TAINT_VALUE>"
          effect: "<TAINT_EFFECT>"
  k8ssandra:
    cassandra:
      # ... rest of your cluster configuration
```

### Configure image coordinates separately (v1.15.0+)

Available in Mission Control 1.15.0 and later (requires `k8ssandra-operator` 1.27 / `cass-operator` 1.27). This lets you override registry, repository, and tag independently:

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: <CLUSTER_NAME>
spec:
  cqlsh:
    image:
      registry: <CUSTOM_REGISTRY_ADDRESS>:<CUSTOM_REGISTRY_PORT>
      repository: <CUSTOM_CQLSH_REPOSITORY>
      tag: <CUSTOM_TAG>
    podTemplateSpec:
      spec:
        containers:
        - name: cqlsh
          # Additional container configuration...
  k8ssandra:
    cassandra:
      # ... rest of your cluster configuration
```

### `cqlsh-pod` security overrides in `MissionControlCluster`

To override security contexts for the `cqlsh-pod` at the `MissionControlCluster` level (the `cqlsh.spec` path in `security-overrides.adoc`):

```yaml
spec:
  cqlsh:
    spec:
      initContainers: []
      containers:
        - name: cqlsh
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop:
                - ALL
            privileged: false
            readOnlyRootFilesystem: true
            runAsGroup: 999
            runAsNonRoot: true
            runAsUser: 999
      securityContext:
        fsGroup: 999
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
```

Apply with `kubectl apply -f <MISSION_CONTROL_CLUSTER.yaml>`.

---

## 11. TLS / Cert Secrets for `cql-router`

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html

Mission Control automatically manages TLS certificates for `cql-router`. When you use the certificate-expiration recovery runbook (see [05-security.md](05-security.md)), the following `cql-router`-related secrets must be deleted as part of the recovery to force regeneration:

```bash
kubectl delete secret -n <PROJECT_SLUG> \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-cert \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-client-cert \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-server-cert \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-management-api-cert \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-client-management-api-cert \
  <CLUSTER_NAME>-<DATACENTER_NAME>-cql-router-server-management-api-cert
```

These six secrets are in addition to the internode, management-API, and per-node `inode` secrets deleted in the same recovery step.

The `cql-router` image is an **entitled image** pulled from `cp.icr.io/cp/ibm-ds-mission-control/cql-router`. An IBM entitlement key (and corresponding pull secret) is required. See [05-security.md](05-security.md) and the image pull secret guidance in [security-overrides.adoc](https://docs.datastax.com/en/mission-control/install/security-overrides.html).

**Version notes:**
- `cql-router` v1.20.3 is included in Mission Control 1.21.0 (upgraded from v1.20.2 in 1.20.2).
- The 1.21.0 airgap bundle image: `cp.icr.io/cp/ibm-ds-mission-control/cql-router:v1.20.3`

---

## See Also

- [04-cloud-config.md](04-cloud-config.md) — Ingress configuration (`kubernetes-ingress`, platform ingress, region domain)
- [05-security.md](05-security.md) — TLS, certificate expiration/recovery, image pull secrets
- [06-cluster-ops.md](06-cluster-ops.md) — Cluster creation, `MissionControlCluster` CR, datacenter management
