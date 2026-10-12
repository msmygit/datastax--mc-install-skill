# 05 — Security: TLS, BYO CA, Multi-Region Certs, LDAP, OIDC, DB Encryption, Cert Expiration, Authentication

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/tls.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/ldap-connector.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/openid-connector.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/authn.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/internode-encrypt.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/client-to-node-encrypt.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/transparent-data-encrypt.html

---

## 1. Platform TLS Configuration

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/tls.html

### Default: fully managed TLS (recommended)

MC auto-generates a self-signed root CA and all platform component leaf certificates using cert-manager.

```yaml
# Default values.yaml TLS block:
tls:
  generateCa: true                             # MC creates and manages root CA
  rootCaSecretName: "mission-control-platform-ca"
  certDuration: 8760h                          # 365 days — lifetime of certs MC creates
  certRenewBefore: 336h                        # cert-manager starts renewal 14 days before expiry
```

- `certDuration` / `certRenewBefore` apply only to certificates that MC creates (not to certs you supply).
- If `generateCa: false`, `rootCaSecretName` must point to an existing CA secret.

> ℹ️ These Helm `tls.*` keys control the **platform** (control/data plane component) certificates. Database-cluster certificate lifetimes are set per cluster with `spec.encryption.internodeEncryption.certs.certTemplate` in the `MissionControlCluster` — see [Section 10](#10-certificate-and-ca-expiration-management).

MC creates the following cert-manager resources automatically (in the `mission-control` namespace):
1. A `selfSigned` Issuer (`mission-control-platform-self-signed`)
2. A root CA Certificate (`mission-control-platform-ca-cert`, `isCA: true`) stored as secret `mission-control-platform-ca` (type `kubernetes.io/tls`, keys `ca.crt`, `tls.crt`, `tls.key`)
3. A CA Issuer that references the root CA secret (`mission-control-issuer`)
4. Leaf certificates signed by `mission-control-issuer`:

| Certificate / secret | Purpose |
|---|---|
| `mission-control-dex` → secret `mission-control-dex-tls` | Mounted into `ui` and `dex` pods; encrypts UI traffic and UI→Dex requests. If `ingress.enabled: true`, `dnsNames` also includes `ingress.regionDomain`. The UI does not enable hostname verification on this connection. |
| `grafana-server-tls` | Grafana ↔ sidecars and direct Grafana connections (adds Grafana hostname when ingress is enabled) |
| `loki-server-tls`, `mimir-server-tls` | Encrypt traffic between Loki/Mimir components; also mounted into the UI pod for observability queries |
| `mission-control-s-aggregator` (server), `mission-control-c-aggregator` (client) | Vector mTLS between Vector agents and the control plane aggregator |

> ℹ️ Earlier versions used a `mission-control-ui-cert` certificate for UI access. Current versions use the Dex certificate instead; the UI cert secret is deprecated and will be removed in a future release.

> ℹ️ Because the root CA is self-signed, browsers show a certificate warning when accessing the MC UI — this is expected and normal.

> ℹ️ Since v1.21.0, MC also enables TLS for `kube-state-metrics` and for the Mimir and Loki gossip rings (release-notes security fixes #2808, #2810, #2813).  
> 📖 Source: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html

### Bring Your Own CA (BYO CA)

To use your organization's CA instead of the auto-generated self-signed one:

```bash
# Step 1: Create a TLS secret from your CA cert and key
kubectl create secret tls MY_CUSTOM_CA_SECRET \
  --cert=path/to/ca.crt \
  --key=path/to/ca.key \
  -n mission-control
```

```yaml
# Step 2: Reference it in values.yaml
tls:
  generateCa: false                    # disable auto-generation
  rootCaSecretName: "MY_CUSTOM_CA_SECRET"
```

With `generateCa: false`, the Helm install does **not** create the self-signed issuer or root CA certificate. MC uses your secret to create `mission-control-issuer` and sign all leaf certificates.

> ⚠️ Ensure your CA can sign the leaf certificates that MC needs.

---

## 2. Multi-Region: Replicate Vector Client Certificate

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/tls.html

MC configures Vector with mutual TLS (mTLS) between data planes and the control plane aggregator. MC does **not** automatically replicate the client certificate — you must do this manually when adding a data plane.

```bash
# On the CONTROL PLANE context:
CP_CONTEXT=your-control-plane-context
kubectl config use-context "$CP_CONTEXT"

# Fetch the aggregator client cert secret, rename metadata for portability
kubectl get secret -n mission-control mission-control-aggregator-tls-client -o yaml \
  | yq eval '.metadata |= {"namespace": .namespace, "name": "cp-mission-control-aggregator-tls-client"}' \
  > cp-mission-control-aggregator-tls-client.yaml

# Switch to DATA PLANE context:
DP_CONTEXT=your-data-plane-context
kubectl config use-context "$DP_CONTEXT"

# Create the namespace if this is a fresh data plane
kubectl create namespace mission-control

# Apply the cert to the data plane
kubectl apply -n mission-control -f cp-mission-control-aggregator-tls-client.yaml
```

Do this **before** installing MC on the data plane cluster.

---

## 3. Image Pull Secrets (Helm installs)

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Required when image pulls need authentication (airgapped environments, private registries). Since MC v1.8.0, the operator **automatically creates** image pull secrets for Helm installs and replicates them to all namespaces. You only need to reference the secret name in your overrides.

```yaml
# Global pull secret reference (covers all components) — usually all you need
global:
  imagePullSecrets:
    - name: PULL_SECRET_NAME
```

Use component-specific settings only when individual components need different registry settings. The docs page provides full samples for: main MC component (`image.*`, `imageConfigs.registryOverride` + `reaper`/`medusa`/`dataApi`/`yq`/`cqlsh`), observability (`agent`, `aggregator`, `loki`, `mimir` incl. `minio`/`nginx`), operators (`k8ssandra-operator`, `cass-operator` incl. `imageConfig.defaults` for `cassandra`/`dse`/`hcd`), and others (`client`, `crdPatchJob`, `ui`, `dex`, `kube-state-metrics`, `kubernetes-ingress.controller.existingImagePullSecret`). Example of the main component block:

```yaml
image:
  registry: REGISTRY_ADDRESS
  repository: REGISTRY_NAMESPACE/mission-control
  imagePullSecret: PULL_SECRET_NAME
imageConfigs:
  registryOverride: REGISTRY_ADDRESS
```

### Verify pull secret creation
```bash
kubectl get secret PULL_SECRET_NAME -n mission-control
# Or list only image pull secrets:
kubectl get secrets -n mission-control --field-selector type=kubernetes.io/dockerconfigjson
```

### Troubleshoot pull failures
- Secret exists in the correct namespace; registry credentials are correct.
- Secret name in `values.yaml` matches the created secret.
- Network connectivity to the registry.
- Check events: `kubectl get events -n mission-control`

---

## 4. Platform Pod Security Context Overrides

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Override `podSecurityContext` and `securityContext` per component via Helm for least-privilege hardening.

> ℹ️ The docs examples show structure and placement only; use values that match your organization's security policies.

```yaml
# values-security.yaml
dex:
  podSecurityContext:
    runAsNonRoot: true
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: true

agent:                     # aggregator: same structure as agent
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: true
    runAsNonRoot: true
  podSecurityContext:
    fsGroup: 1001
    runAsUser: 1001
    runAsNonRoot: true

loki:
  sidecar:
    securityContext:       # same fields as agent.securityContext
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
  loki:
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: [ALL]
      readOnlyRootFilesystem: true
      runAsNonRoot: true
  podSecurityContext:
    runAsNonRoot: true

k8ssandra-operator:        # cass-operator: same structure
  podSecurityContext:
    runAsNonRoot: true
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: true
```

Apply:
```bash
helm upgrade mission-control oci://icr.io/mission-control-helm/mission-control \
  --namespace mission-control --create-namespace \
  -f values-security.yaml
```

> ⚠️ On an existing install, also pass your normal `-f values.yaml` (and `--version`) so the upgrade doesn't drop other settings.

---

## 5. LDAP Authentication

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/ldap-connector.html

MC uses Dex as its identity provider. Configure LDAP by adding a connector to the Dex config.

### Via Helm `values.yaml`

```yaml
dex:
  config:
    connectors:
      - type: ldap
        id: ldap
        name: LDAP
        config:
          host: ldap.example.com:636        # use port 636 for LDAPS (TLS)
          # insecureNoSSL: false            # set true ONLY for port 389 (not recommended)
          # insecureSkipVerify: false       # never true in production
          startTLS: false                   # set true if using StartTLS on port 389
          rootCAData: BASE64_PEM_CA         # base64-encoded PEM of LDAP server CA

          bindDN: "cn=bind-user,dc=example,dc=org"
          bindPW: "bind-password"

          usernamePrompt: "Username"

          userSearch:
            baseDN: "ou=users,dc=example,dc=org"
            filter: "(objectClass=inetOrgPerson)"
            username: uid                   # attribute matched against login input
            idAttr: uid
            emailAttr: mail
            nameAttr: displayName

          groupSearch:
            baseDN: "ou=groups,dc=example,dc=org"
            filter: "(objectClass=groupOfNames)"
            userMatchers:
              - userAttr: DN
                groupAttr: member
            nameAttr: cn
```

> ⚠️ Always use TLS (port 636 with `insecureNoSSL: false`). Plain LDAP (port 389) leaks credentials.

---

## 6. OIDC Authentication

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/openid-connector.html

### Via Helm `values.yaml`

```yaml
# REQUIRED when using OIDC with Ingress: set baseUrl to match your ingress hostname
ui:
  baseUrl: "https://mc.example.com"

dex:
  config:
    connectors:
      - type: oidc
        id: oidc
        name: "My SSO"
        config:
          issuer: "https://accounts.example.com"    # provider's OIDC discovery URL
          clientID: "YOUR_CLIENT_ID"
          clientSecret: "YOUR_CLIENT_SECRET"
          redirectURI: "https://mc.example.com/callback"

          # Optional fields:
          # basicAuthUnsupported: false    # set true if provider needs POST params
          scopes:
            - openid
            - profile
            - email
          getUserInfo: true               # query UserInfo endpoint for extra claims
          userIDKey: sub
          userNameKey: name
          preferredUsernameKey: preferred_username
          emailKey: email
          # promptType: consent           # override prompt type if provider requires
```

> ⚠️ `ui.baseUrl` **must** match the hostname clients use to access the UI. If this is wrong, OIDC redirect URIs won't match and login will fail.

### OIDC key considerations

| Setting | Notes |
|---------|-------|
| `issuer` | Must exactly match the `iss` claim in tokens from your IdP |
| `redirectURI` | Must be registered in your OIDC provider as an allowed redirect |
| `basicAuthUnsupported` | Set `true` for AWS Cognito, some Okta configs |
| `getUserInfo` | Use when the IDToken doesn't contain all required claims |
| `enableGroups` | Only enable if stale group claims are acceptable (groups refresh on IDToken refresh only) |

---

## 7. Database Resource Security Overrides (MissionControlCluster, CqlConnectivity, DataApi)

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Apply these with `kubectl apply -f FILE.yaml`. The docs page has the full samples; key structure below.

### MissionControlCluster

Container `securityContext` per init container / container, plus `podSecurityContext`, under `spec.k8ssandra.cassandra`; the `cqlsh` pod is overridden under `spec.cqlsh.spec`.

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: my-cluster
  namespace: my-project
spec:
  k8ssandra:
    cassandra:
      initContainers:
        - name: cass-config-builder        # docs also show DSE / HCD / Medusa init containers
          securityContext: &restricted
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
            privileged: false
            readOnlyRootFilesystem: true
            runAsGroup: 999
            runAsNonRoot: true
            runAsUser: 999
      containers:
        - name: cassandra
          securityContext: *restricted
        - name: server-system-logger
          securityContext: *restricted
      podSecurityContext:
        fsGroup: 999
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
  cqlsh:
    spec:
      initContainers: []
      containers:
        - name: CONTAINER_NAME
          securityContext: *restricted
      securityContext:                     # pod-level for the cqlsh pod
        fsGroup: 999
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
```

> ℹ️ YAML anchors (`&restricted` / `*restricted`) are used here only for brevity; the docs spell each block out in full.

### CqlConnectivity

Overrides go under `spec.loadBalancers.podConfig`:

```yaml
apiVersion: missioncontrol.datastax.com/v1alpha1
kind: CqlConnectivity
metadata:
  name: CQL_CONNECTIVITY_NAME
spec:
  cassandraDatacenterRef:
    name: DATACENTER_NAME
  loadBalancers:
    nativePort: 9042
    serviceConfig: {}
    size: 2
    podConfig:
      containerSecurityContext:
        allowPrivilegeEscalation: false
        capabilities:
          drop: [ALL]
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

### DataApi

Overrides go directly under `spec`:

```yaml
apiVersion: missioncontrol.datastax.com/v1alpha1
kind: DataApi
metadata:
  name: DATA_API_NAME
  namespace: NAMESPACE
spec:
  cassandraDatacenterRef:
    name: DATACENTER_NAME
  replicas: 1
  services:
    clusterIP:
      port: CLUSTER_IP_PORT     # for external access, use services.ingress instead of nodePort
  containerSecurityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    privileged: false
    readOnlyRootFilesystem: true
  podSecurityContext:
    runAsNonRoot: true
```

---

## 8. Database Authentication

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/authn.html

- Authentication is **enabled by default** (`spec.k8ssandra.auth: true`). Do not disable it.
- MC automatically creates a `CLUSTER_NAME-superuser` Kubernetes Secret in the cluster namespace.
- The default `cassandra`/`cassandra` user is **disabled** in MC-managed clusters.
- Retrieve the auto-generated superuser credentials:

```bash
# Secret name follows the pattern: <cluster-name>-superuser
kubectl get secret production-hcd-superuser -n my-project \
  -o jsonpath='{.data.username}' | base64 --decode && echo

kubectl get secret production-hcd-superuser -n my-project \
  -o jsonpath='{.data.password}' | base64 --decode && echo
```

### Running nodetool with authentication enabled

```bash
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- \
  nodetool -u production-hcd-superuser -pw YOUR_PASSWORD status
```

### Running nodetool with TLS + authentication

📖 Source: https://docs.datastax.com/en/mission-control/administration/control-plane/nodetool-tls.html

When client-to-node TLS is enabled, nodetool also needs SSL configuration:

**Step 1: Create the nodetool SSL ConfigMap**

```yaml
# nodetool-ssl-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodetool-ssl
  namespace: my-project
data:
  nodetool-ssl.properties: |
    ssl=true
    ssl.keystore=/etc/nodetool/ssl/keystore.jks
    ssl.keystore.password=keystorepassword
    ssl.truststore=/etc/nodetool/ssl/truststore.jks
    ssl.truststore.password=truststorepassword
```

**Step 2: Mount the ConfigMap into the Cassandra pod via `storageConfig.additionalVolumes`**

```yaml
spec:
  k8ssandra:
    cassandra:
      datacenters:
        - metadata:
            name: dc1
          storageConfig:
            additionalVolumes:
              - mountPath: /etc/nodetool/ssl
                name: nodetool-ssl
                pvcSpec: {}
                # Or reference a ConfigMap directly:
              - mountPath: /etc/nodetool
                name: nodetool-ssl-props
                configMapRef:
                  name: nodetool-ssl
```

**Step 3: Run nodetool with `--ssl` flag**

```bash
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- \
  nodetool --ssl -u USERNAME -pw PASSWORD status
```

---

## 9. Database Internode Encryption

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/internode-encrypt.html

MC manages and orchestrates internode encryption for DSE, HCD, and Apache Cassandra clusters. For a new cluster MC generates a certificate per node, signs it with the cluster's root CA, then generates each node's keystore and truststore and configures the cluster to use them.

| Mode | Use case |
|------|---------|
| MC-provided root CA + self-signing (default) | Production: no extra fields needed in the spec |
| Alternate cert-manager issuer | Enterprise: Vault, Venafi, ACME/Let's Encrypt, or external issuers |
| No encryption | Dev/test only |

### Mode 1: MC-provided root CA (default)

Nothing to add — a standard `MissionControlCluster` (no `encryption` block) gets internode encryption. Within seconds MC creates the root CA secret, then per-node cert and keystore secrets:

```bash
kubectl get secrets -n PROJECT_SLUG | grep inode
# mission-control-test-inode-selfsigned-ca-cert   kubernetes.io/tls  3
# test-dc1-default-sts-0-inode-cert               kubernetes.io/tls  3
# test-dc1-default-sts-0-inode-cert-keystore      Opaque             2
# ... one cert + keystore pair per node
```

The `-keystore` secrets are mounted into the HCD/DSE/Cassandra pods, which then start within a few minutes.

### Mode 2: Alternate cert-manager Issuer (Vault, Venafi, ACME)

Create the issuer with the standard cert-manager API, then reference it:

```yaml
spec:
  createIssuer: false                 # do NOT create MC's built-in issuer
  encryption:
    internodeEncryption:
      certs:
        certTemplate:
          issuerRef:
            name: my-alternate-issuer # your cert-manager Issuer
```

> ℹ️ Works with any cert-manager-supported issuer: HashiCorp Vault, Venafi, ACME (Let's Encrypt), or external issuers. Per-node `-inode-cert` / `-inode-cert-keystore` secrets are created the same way as Mode 1.

### Mode 3: Disable internode encryption

```yaml
spec:
  createIssuer: false
  encryption:
    internodeEncryption:
      enabled: false
```

---

## 10. Certificate and CA Expiration Management

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html

MC sets default certificate expiration to **20 years** for:
- Root CA
- Internode certificates
- Native protocol ingress certificates
- Management API certificates

> ⚠️ **Pre-1.9.0 DSE clusters:** before upgrading to MC 1.9.0, complete the 1.9.0 post-upgrade steps to rotate the CA and each node's certificate in a rolling fashion for all existing DSE clusters (retrieve superuser creds, run `dsetool ... tsreload server` in every DSE pod, then a rolling restart via **Bulk Actions → Restart**). These steps still apply to any older DSE cluster being brought forward.  
> 📖 Source: https://docs.datastax.com/en/mission-control/release-notes/release-notes.html#version-1-9-0

### Configure duration with `certTemplate`

```yaml
spec:
  encryption:
    internodeEncryption:
      certs:
        certTemplate:
          duration: "17520h"    # 2 years (default is 20 years = "175200h")
          renewBefore: "720h"   # start renewal 30 days before expiration
          keySize: 4096         # RSA key size
```

📖 CRD field reference: https://docs.datastax.com/en/mission-control/crd-reference/missioncontrolcluster-v1.9.0.html

Then `kubectl apply -f MISSION_CONTROL_CLUSTER.yaml`.

### Rotation behavior

1. The **keystore** holds only the new certificates, so all outgoing connections use the updated credentials.
2. During the transition the **truststore** holds both the old and new CAs, so nodes using different certs can still interoperate.
3. Once all nodes use the new certificates, MC triggers the **rotate internode certificates** bulk action to remove the old CA from truststores and keystores.

### Verify

```bash
kubectl get secrets -n mission-control
kubectl get certificate -n mission-control          # READY should be True
kubectl get issuer -n mission-control               # READY should be True
kubectl get certificaterequest -n mission-control
```

### Recover from certificate expiration

When certificates expire, components cannot establish secure communication and the cluster is unavailable.

> ⚠️ This procedure causes downtime. Use it only when the cluster cannot recover/restart because certificates have already expired.

1. Stop the datacenter — set `stopped: true` and apply:
   ```yaml
   apiVersion: missioncontrol.datastax.com/v1
   kind: MissionControlCluster
   metadata:
     name: CLUSTER_NAME
     namespace: PROJECT_SLUG
   spec:
     k8ssandra:
       cassandra:
         datacenters:
           - name: DATACENTER_NAME
             stopped: true
   ```
   ```bash
   kubectl apply -f MISSION_CONTROL_CLUSTER.yaml
   kubectl get pods -n PROJECT_SLUG | grep DATACENTER_NAME   # confirm pods are gone
   ```
2. Delete the cluster's certificate secrets (repeat the `-inode` / `-inode-ks` pair for every `RACK_NAME`-`NODE_INDEX`):
   ```bash
   C=CLUSTER_NAME; DC=DATACENTER_NAME
   kubectl delete secret -n PROJECT_SLUG \
     $C-$DC-internode-client-cert \
     $C-$DC-internode-server-cert \
     $C-$DC-c-mgmt \
     $C-$DC-c-mgmt-ks \
     $C-$DC-s-mgmt \
     $C-$DC-s-mgmt-ks \
     $C-$DC-RACK_NAME-NODE_INDEX-inode \
     $C-$DC-RACK_NAME-NODE_INDEX-inode-ks \
     $C-$DC-client-cert \
     $C-$DC-management-api-cert \
     $C-$DC-cql-router-cert \
     $C-$DC-cql-router-client-cert \
     $C-$DC-cql-router-server-cert \
     $C-$DC-cql-router-management-api-cert \
     $C-$DC-cql-router-client-management-api-cert \
     $C-$DC-cql-router-server-management-api-cert
   ```
3. Verify with `kubectl get secrets -n PROJECT_SLUG` and delete any remaining secrets related to the expired certificates: `kubectl delete secret SECRET_NAME -n PROJECT_SLUG`.
4. Delete the `Certificate` resource and verify:
   ```bash
   kubectl delete certificate -n PROJECT_SLUG cluster-certificate
   kubectl get certificate -n PROJECT_SLUG
   ```
5. Restart — set `stopped: false` in the `MissionControlCluster` and `kubectl apply -f MISSION_CONTROL_CLUSTER.yaml`. This triggers reconciliation and creates new certificates.
6. Verify:
   ```bash
   kubectl get secrets -n PROJECT_SLUG               # new certificate secrets exist
   kubectl describe secret -n PROJECT_SLUG           # secrets re-created
   kubectl get pods -n PROJECT_SLUG | grep DATACENTER_NAME   # datacenter running
   ```

---

## 11. Database Client-to-Node Encryption (TLS for App Connections)

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/client-to-node-encrypt.html

> ⚠️ MC does **NOT** orchestrate client-to-node TLS. You must provide your own JKS keystore and truststore. This is a bring-your-own-certificate model.

**Step 1: Create the keystore secret**

```bash
# Create a secret with keystore.jks and truststore.jks
kubectl create secret generic client-encryption-stores \
  --namespace my-project \
  --from-file=keystore.jks=/path/to/keystore.jks \
  --from-file=truststore.jks=/path/to/truststore.jks
```

**Step 2: Reference the secret and enable encryption in MissionControlCluster**

```yaml
spec:
  k8ssandra:
    cassandra:
      # Reference the keystore/truststore secrets
      clientEncryptionStores:
        keystoreSecretRef:
          name: client-encryption-stores
          key: keystore.jks
        keystorePasswordSecretRef:
          name: client-encryption-stores
          key: keystore-password          # if password is stored as a secret key
        truststoreSecretRef:
          name: client-encryption-stores
          key: truststore.jks
        truststorePasswordSecretRef:
          name: client-encryption-stores
          key: truststore-password
      # Enable encryption in cassandra.yaml
      config:
        cassandraYaml:
          client_encryption_options:
            enabled: true
            optional: false             # set true to allow non-TLS clients during migration
            require_client_auth: false  # set true for mutual TLS (clients need a cert too)
```

**Step 3: Verify TLS is active**

```bash
kubectl exec -it CASSANDRA_POD -n my-project -c cassandra -- \
  cqlsh --ssl --cqlshrc=/path/to/.cqlshrc -u USERNAME -p PASSWORD \
  -e "DESCRIBE KEYSPACES;"
```

---

## 12. Transparent Data Encryption — TDE (DSE Only)

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/transparent-data-encrypt.html

> ⚠️ MC supports the TDE feature of **DSE**. Apache Cassandra does not have this feature.

**Step 1: Generate the encryption/decryption key file**

```bash
dsetool createsystemkey 'AES/ECB/PKCS5Padding' 128 tde.key -d .
```

**Step 2: Store the key file in a secret in the cluster's namespace**

```bash
kubectl create secret generic tde-secret \
  --from-file=tde.key=/path/to/tde.key \
  -n NAMESPACE
```

**Step 3: Add the volume, mount it into the `cassandra` container, and configure `dse.yaml`**

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: test
  namespace: NAMESPACE
spec:
  createIssuer: true
  k8ssandra:
    cassandra:
      serverType: dse
      serverVersion: 6.8.36
      # ... storageConfig, datacenters, etc.
      extraVolumes:
        volumes:
          - name: tde-key
            secret:
              secretName: tde-secret
      containers:
        - name: "cassandra"
          volumeMounts:
            - name: tde-key
              mountPath: /var/tde
      config:
        dseYaml:
          system_key_directory: /var/tde
          config_encryption_key_name: tde.key
```

**Step 4: Create encrypted tables**

```sql
CREATE TABLE test.encryption_test (id int PRIMARY KEY, val text) WITH COMPRESSION = {
  'class': 'EncryptingLZ4Compressor',
  'cipher_algorithm': 'AES/ECB/PKCS5Padding',
  'secret_key_strength': 128,
  'system_key_file': 'tde.key' };
```

Read and write the table with normal CQL (`INSERT` / `SELECT`).

---

## 13. OpenShift SCC Management

📖 Source: https://docs.datastax.com/en/mission-control/install/install-mc-openshift.html

### Grant SCCs before install
```bash
for SA in loki mission-control mission-control-agent mission-control-aggregator \
  mission-control-cass-operator mission-control-dex \
  mission-control-k8ssandra-operator mission-control-kube-state-metrics \
  mission-control-mimir; do
  oc adm policy add-scc-to-user nonroot-v2 -z $SA -n mission-control
done
```

### Remove SCCs before uninstall (critical — omitting this causes resources to hang)
```bash
for SA in loki mission-control mission-control-agent mission-control-aggregator \
  mission-control-cass-operator mission-control-dex \
  mission-control-k8ssandra-operator mission-control-kube-state-metrics; do
  oc adm policy remove-scc-from-user nonroot-v2 -z $SA -n mission-control
done
```

---

## 14. Helm Upgrade RBAC (narrow-scope, no cluster-admin required)

📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html

For organizations where app teams should be able to run `helm upgrade` without being cluster-admin, MC provides a `helm-upgrader` ServiceAccount + ClusterRole pattern with `resourceNames` scoped to only MC CRDs. Follow the guide at the URL above to configure.

---

## See also

- Certificate and CA expiration management: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html
- Override default security settings: https://docs.datastax.com/en/mission-control/install/security-overrides.html
- Securing overview: https://docs.datastax.com/en/mission-control/secure/mc/securing.html
- Platform components reference: https://docs.datastax.com/en/mission-control/reference/platform-components.html
- CRD reference: https://docs.datastax.com/en/mission-control/crd-reference/crd-reference-overview.html
