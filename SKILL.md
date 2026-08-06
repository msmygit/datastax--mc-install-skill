---
name: mc-install
description: >-
  Install, configure, and operate DataStax Mission Control on any Kubernetes flavor (EKS, AKS, GKE, OpenShift) — online and air-gapped — to deploy and manage HCD, DSE, and Cassandra clusters. Use when a DBA, K8s operator, or platform engineer needs guidance on Mission Control installation, pre-flight checks, license acquisition (paid or trial), cloud-provider-specific configuration, air-gap/offline installation, HCD cluster deployment, multi-region setup, ingress, TLS/security, LDAP/OIDC auth, observability, backup/restore, upgrades, migration, or troubleshooting. Trigger phrases: "Mission Control", "install Mission Control", "HCD cluster", "mc-install", "DataStax MC", "K8ssandra operator", "cass-operator", "deploy Cassandra on K8s", "Mission Control on EKS", "Mission Control on AKS", "Mission Control on GKE", "Mission Control on OpenShift", "MissionControlCluster", "IBM Container Registry", "ICR", "Helm install MC", "migrate from Replicated", "air-gap Mission Control", "offline install", "private registry", "Skopeo mirror".
metadata:
  author: Bob
  version: "3.0"
  display_name: Mission Control Install & Operate
  short_description: Expert guide — install, configure, and operate DataStax Mission Control (online + air-gap) on EKS/AKS/GKE/OpenShift.
  iconName: dataBase
---

# Mission Control Install & Operate — Orchestrator

You are an expert DataStax Mission Control installer and operator.
This skill is split into focused topic files. **Load the relevant file(s) with `read_file` before answering.**

📁 **Skill directory:** `~/.bob/skills/mc-install/`  
📖 **Primary docs:** https://docs.datastax.com/en/mission-control/index.html

> ⚠️ **As of Mission Control 1.20.1, Replicated (KOTS) is no longer supported as an installation method.** All new installs use Helm from the IBM Container Registry (ICR). If you have an existing Replicated/KOTS installation, route to the migration guide first.
> 📖 Install (Helm/ICR): https://docs.datastax.com/en/mission-control/install/install-mc-helm.html
> 📖 Migrate Replicated → ICR: https://docs.datastax.com/en/mission-control/install/migrate-replicated-to-icr.html

---

## Architecture at a glance

```
┌──────────────────────────────────────────────┐
│            CONTROL PLANE CLUSTER              │
│  mission-control-operator  (lifecycle+certs)  │
│  Mission Control UI + REST API                │
│  Dex (auth: static / LDAP / OIDC)            │
│  Loki (logs) · Mimir (metrics) · Grafana      │
│  Reaper (repair) · Medusa (backup/restore)    │
│  Vector agent + aggregator                    │
│  ← also serves as data plane for region 1 →  │
└─────────────────┬────────────────────────────┘
                  │ K8s API (CRDs)
       ┌──────────┴──────────┐
       ▼                     ▼
 ┌──────────────┐    ┌──────────────┐
 │  DATA PLANE  │    │  DATA PLANE  │
 │   Region 2   │    │   Region 3   │
 │  k8ssandra-  │    │  k8ssandra-  │
 │   operator   │    │   operator   │
 │  cass-op     │    │  cass-op     │
 │  HCD Pods    │    │  HCD Pods    │
 └──────────────┘    └──────────────┘
```

---

## Key concepts

| Concept | Meaning |
|---------|---------|
| **Control Plane** | ONE per organization. Hosts API, UI, observability, operators. |
| **Data Plane** | Runs actual DB nodes. Can be the same K8s cluster as CP or separate. |
| **HCD** | Hyper-Converged Database — cloud-native Cassandra-compatible DB. |
| **`MissionControlCluster` CRD** | Primary resource to define an HCD/DSE/Cassandra cluster. |
| **cass-operator** | Per data plane. Manages `CassandraDatacenter` CRDs, pods, StatefulSets. |
| **k8ssandra-operator** | Per data plane. Manages `K8ssandraCluster` CRDs, Medusa, Reaper. |
| **mission-control-operator** | Control plane only. Orchestrates multi-cluster ops, certs, lifecycle. |

---

## Version matrix

| Component | Required version |
|-----------|-----------------|
| Kubernetes | 1.21.0+ (**avoid 1.35.0–1.35.3** — `MaxUnavailableStatefulSet` bug) |
| OpenShift | 4.8+ |
| Helm | **3.14.0–3.18.0 only** (3.19+ not yet supported) |
| cert-manager | 1.16.1 recommended |

**Supported DB versions:** HCD 2.0, HCD 1.2.3+, HCD 1.1, DSE 6.8 (6.8.25+, except 6.8.27/6.8.45), DSE 6.9, Cassandra 3.11 (3.11.7–3.11.17), 4.0, 4.1, 5.0 (except 5.0.0).

📖 Source: https://docs.datastax.com/en/mission-control/misc/supported-platforms.html

---

## STEP 1 — Discover user context first

Ask the user these questions before giving specific guidance:

1. **K8s platform?** EKS / AKS / GKE / OpenShift / kind / Other
2. **Online or air-gapped?** (Changes the entire install path)
3. **Install method?** Helm (recommended) / Helm + separate cluster resources
4. **Database type?** HCD / DSE / Apache Cassandra
5. **Entitlement status?** IBM entitlement key in hand / Need to obtain one from IBM Container Library / Existing Replicated install to migrate
6. **Single region or multi-region?**
7. **Goal today?** Fresh install / Migrate from Replicated to ICR / Upgrade / Day-2 ops / Troubleshoot

---

## STEP 2 — Route to the correct topic file

After gathering context, load the appropriate file(s) with `read_file`:

| User need | Load this file |
|-----------|---------------|
| Hardware, software, ports, node labels, entitlement key, storage class setup | `~/.bob/skills/mc-install/01-preflight.md` |
| Online install: cert-manager, Helm (ICR), OCP, separate resources, **migrating an existing Replicated/KOTS install to ICR** | `~/.bob/skills/mc-install/02-install-online.md` |
| Air-gap/offline install: Skopeo mirroring from ICR, private-registry values | `~/.bob/skills/mc-install/03-install-airgap.md` |
| Cloud-specific values (EKS/AKS/GKE/OCP), DNS, ingress | `~/.bob/skills/mc-install/04-cloud-config.md` |
| TLS/BYO CA, multi-region cert copy, LDAP, OIDC, DB encryption, TDE, client TLS, nodetool SSL | `~/.bob/skills/mc-install/05-security.md` |
| Deploy HCD cluster, config, canary, scale nodes, replace node, rebuild DC, backup/restore, upgrades, CP outage, migration, day-2 | `~/.bob/skills/mc-install/06-cluster-ops.md` |
| Troubleshooting, diagnostics, CP outage ops, support bundle, uninstall, DBA checklist | `~/.bob/skills/mc-install/07-troubleshoot.md` |
| Pod sizing, Grafana/Mimir/Loki scaling, alerts, custom PromQL rules, external sinks, enterprise 45+ node config | `~/.bob/skills/mc-install/08-observability.md` |

---

## Quick FAQ

- **MC vs OpsCenter?** MC supersedes OpsCenter for cloud/K8s deployments.
- **Pricing?** Included in HCD or DSE license. OSS Cassandra nodes have additional cost.
- **Co-location?** Nodes from different clusters can share a host; same-cluster nodes cannot.
- **Cert expiry?** Root CA and internode certs default to 20-year expiry (v1.9.0+).
- **Two separate buckets?** Yes — Mimir (metrics) and Loki (logs) **cannot share** a bucket.
- **Release name rule?** Must NOT contain `mission-control` — causes naming conflicts.
- **What happened to Replicated/KOTS?** Deprecated as of 1.20.1 and removed entirely in later versions. All installs now use Helm from `oci://icr.io/mission-control-helm/mission-control`. Existing Replicated/KOTS installs must migrate in place — see `02-install-online.md`.
- **Where do entitled images come from?** `cp.icr.io` (requires an IBM entitlement key as the pull-secret password, username `cp`). Public MC images (`mission-control`, `mission-control-ui`, `mission-control-dex`, `k8ssandra-client`) come from the non-entitled `icr.io` registry.

📖 FAQ: https://docs.datastax.com/en/mission-control/frequently-asked-questions/faq.html  
📖 Install: https://docs.datastax.com/en/mission-control/install/install-mc-helm.html  
📖 Migrate from Replicated: https://docs.datastax.com/en/mission-control/install/migrate-replicated-to-icr.html
