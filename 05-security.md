# 05 — Security: TLS, BYO CA, Multi-Region Certs, LDAP, OIDC, DB Encryption, Authentication

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/tls.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/ldap-connector.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/openid-connector.html  
📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/authn.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/internode-encrypt.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/client-to-node-encrypt.html  
📖 Source: https://docs.datastax.com/en/mission-control/secure/database/transparent-data-encrypt.html

---

## 1. TLS Configuration

📖 Source: https://docs.datastax.com/en/mission-control/secure/mc/tls.html

### Default: fully managed TLS (recommended)

MC auto-generates a self-signed root CA and all component leaf certificates using cert-manager.

```yaml
# Default values.yaml TLS block:
tls:
  generateCa: true                             # MC creates and manages root CA
  rootCaSecretName: "mission-control-platform-ca"
  certDuration: 8760h                          # 365 days per leaf cert
  certRenewBefore: 336h                        # renew 14 days before expiry
```

MC creates the following cert-manager resources automatically:
1. A `selfSigned` Issuer (`mission-control-platform-self-signed`)
2. A root CA Certificate (`mission-control-platform-ca-cert`) stored as secret `mission-control-platform-ca`
3. An Issuer that references the root CA (`mission-control-issuer`)
4. Leaf certificates for: Dex/UI, Grafana, Loki, Mimir, Vector (mTLS)

> ℹ️ Because the root CA is self-signed, browsers show a certificate warning when accessing the MC UI — this is expected and normal.

> ℹ️ Since v1.9.0, root CA and internode certs are set to 20-year expiry by default.

📖 Certificate expiry management: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html

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

MC will use your CA secret to create the `mission-control-issuer` and sign all leaf certificates.

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

Since MC v1.8.0, the operator **automatically creates** image pull secrets for Helm installs and replicates them to all namespaces. You only need to reference the secret name.

```yaml
# Global pull secret reference (covers all components)
global:
  imagePullSecrets:
    - name: MC_PULL_SECRET_NAME
```

### Verify pull secret creation
```bash
kubectl get secrets -n mission-control --field-selector type=kubernetes.io/dockerconfigjson
# Should list your pull secret
```

---

## 4. Platform Pod Security Context Overrides

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Override pod security for hardening (least-privilege):

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

agent:
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

aggregator:
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

k8ssandra-operator:
  podSecurityContext:
    runAsNonRoot: true
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: [ALL]
    readOnlyRootFilesystem: true
```

Apply after install:
```bash
helm upgrade mc-release oci://icr.io/mission-control-helm/mission-control \
  --version $MC_VERSION \
  --namespace mission-control \
  -f values.yaml \
  -f values-security.yaml
```

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

## 7. MissionControlCluster Security Overrides

📖 Source: https://docs.datastax.com/en/mission-control/install/security-overrides.html

Override security contexts for database pods in the `MissionControlCluster` CRD:

```yaml
apiVersion: missioncontrol.datastax.com/v1beta2
kind: MissionControlCluster
metadata:
  name: my-cluster
  namespace: my-project
spec:
  k8ssandra:
    cassandra:
      podSecurityContext:
        fsGroup: 999
        runAsGroup: 999
        runAsNonRoot: true
        runAsUser: 999
      containers:
        - name: cassandra
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
            privileged: false
            readOnlyRootFilesystem: true
            runAsGroup: 999
            runAsNonRoot: true
            runAsUser: 999
        - name: server-system-logger
          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: [ALL]
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 999
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

Three modes of internode encryption are available:

| Mode | Use case |
|------|---------|
| No encryption | Dev/test only |
| MC-provided CA (default) | Production: MC auto-generates and rotates internode certs |
| Alternate cert-manager issuer | Enterprise: use Vault, Venafi, ACME/Let's Encrypt, or external issuers |

### Mode 1: MC-managed internode encryption (recommended)

```yaml
spec:
  createIssuer: true                  # create the MC issuer in the cluster namespace
  encryption:
    internodeEncryption:
      enabled: true
      certs:
        createCerts: true             # MC generates and rotates certs automatically
```

### Mode 2: Alternate cert-manager Issuer (Vault, Venafi, ACME)

```yaml
spec:
  createIssuer: false                 # do NOT use MC's built-in issuer
  encryption:
    internodeEncryption:
      enabled: true
      certs:
        createCerts: false            # do NOT create certs; use external issuer
        issuerRef:
          name: vault-issuer          # name of your cert-manager Issuer or ClusterIssuer
          kind: ClusterIssuer         # or Issuer (namespace-scoped)
          group: cert-manager.io
```

> ℹ️ This works with any cert-manager-compatible issuer: HashiCorp Vault (`vault.cert-manager.io`), Venafi, ACME/Let's Encrypt (`acme.cert-manager.io`), or custom external issuers.

Once enabled, all gossip and streaming traffic between Cassandra nodes is encrypted.

---

## 10. Database Client-to-Node Encryption (TLS for App Connections)

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

## 11. Transparent Data Encryption — TDE (DSE Only)

📖 Source: https://docs.datastax.com/en/mission-control/secure/database/transparent-data-encrypt.html

> ⚠️ TDE is available **only for DSE** clusters. It is **not** available for Cassandra or HCD.

TDE encrypts data at rest at the SSTable level. Every table that needs encryption must use `EncryptingLZ4Compressor`.

**Step 1: Generate the system encryption key**

```bash
# Run this inside a DSE pod to generate the key
kubectl exec -it DSE_POD -n my-project -c cassandra -- \
  dsetool createsystemkey 'AES/ECB/PKCS5Padding' 128 tde.key
```

**Step 2: Retrieve and store the key as a Kubernetes Secret**

```bash
# Copy the generated key from the pod
kubectl cp my-project/DSE_POD:/etc/dse/conf/tde.key ./tde.key

# Create a Kubernetes secret from the key file
kubectl create secret generic dse-tde-key \
  --namespace my-project \
  --from-file=tde.key=./tde.key
```

**Step 3: Mount the key and configure DSE in MissionControlCluster**

```yaml
spec:
  k8ssandra:
    cassandra:
      # Mount the key file into the DSE container
      extraVolumes:
        - name: tde-key-vol
          secret:
            secretName: dse-tde-key
      containers:
        - name: cassandra
          volumeMounts:
            - name: tde-key-vol
              mountPath: /etc/dse/conf/system_keys
              readOnly: true
      config:
        dseYaml:
          # Tell DSE where to find the system key
          system_key_directory: /etc/dse/conf/system_keys
          config_encryption_key_name: tde.key
          # Enable encryption at rest
          encryption_options:
            enabled: true
            cipher_algorithm: AES
            secret_key_strength: 128
            key_name: tde.key
```

**Step 4: Create encrypted tables**

```sql
-- Tables must use EncryptingLZ4Compressor to be encrypted at rest
CREATE TABLE my_keyspace.my_encrypted_table (
  id UUID PRIMARY KEY,
  data TEXT
) WITH compression = {
  'class': 'com.datastax.bdp.cassandra.crypto.EncryptingLZ4Compressor',
  'cipher_algorithm': 'AES',
  'secret_key_strength': 128,
  'chunk_length_in_kb': 64
};
```

> ℹ️ Existing tables must be altered to use `EncryptingLZ4Compressor` and then `nodetool upgradesstables` must be run on each node to re-compress existing data with encryption.

---

## 12. OpenShift SCC Management

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

## 13. Helm Upgrade RBAC (narrow-scope, no cluster-admin required)

📖 Source: https://docs.datastax.com/en/mission-control/administration/mc/helm-upgrade-rbac.html

For organizations where app teams should be able to run `helm upgrade` without being cluster-admin, MC provides a `helm-upgrader` ServiceAccount + ClusterRole pattern with `resourceNames` scoped to only MC CRDs. Follow the guide at the URL above to configure.

---

## See also

- Certificate and CA expiration management: https://docs.datastax.com/en/mission-control/secure/mc/manage-cert-ca-expiration.html
- Securing overview: https://docs.datastax.com/en/mission-control/secure/mc/securing.html
- Platform components reference: https://docs.datastax.com/en/mission-control/reference/platform-components.html
- CRD reference: https://docs.datastax.com/en/mission-control/crd-reference/crd-reference-overview.html
