# Immich (self-hosted photo and video backup)

> **Not ready for deployment yet.** The ArgoCD application for Immich is commented out. Do not enable it until external PostgreSQL (and other dependencies below) are in place and values are set. See [Dependencies (have these before deploying)](#dependencies-have-these-before-deploying) and [Deployment hold-off (ArgoCD)](#deployment-hold-off-argocd).

Immich runs on Kubernetes via the [official Immich Helm chart](https://github.com/immich-app/immich-charts) with **external PostgreSQL**, in-chart **Valkey** (Redis), and **1Password-backed secrets** for DB credentials. This chart does **not** deploy PostgreSQL; you must have an external database and other dependencies ready before deploying. **Volumes** (e.g. library PVC) **are defined in Longhorn** or as existingClaims.

---

## Dependencies (have these before deploying)

Deploy or provision the following **before** installing Immich or enabling the ArgoCD application:

| Dependency | Required | Notes |
|------------|----------|--------|
| **External PostgreSQL** | Yes | **One database server** for homelab (same as Nextcloud): host `postgresql-rw.postgresql.svc.cluster.local`. Create an **individual database** `immich`; credentials use the **same** 1Password item as the Postgres cluster (`postgresql` / `password`) via External Secrets → secret `immich-db-credentials` (DB_USERNAME=postgres). |
| **PersistentVolumeClaim** (library) | Yes | Default: share **nextcloud-data** PVC with Nextcloud using `existingClaim: nextcloud-data` and `subPath: immich-library` so the Immich library lives in a fixed directory on the same volume. Ensure the chart supports `subPath` for library, or create a dedicated `immich-library` PVC. |
| **External Secrets Operator** | Yes | ClusterSecretStore (e.g. **onepassword-connect**). The chart creates `immich-db-credentials` from the same 1Password item as the Postgres cluster (**postgresql**, property **password**). No separate 1Password item for Immich DB. |
| **External Redis (optional)** | No | By default the chart runs Valkey (Redis-compatible) in-chart. For a dedicated Redis, set `immich.valkey.enabled: false` and add `REDIS_HOSTNAME` (and optionally `REDIS_PORT`, `REDIS_PASSWORD`) to `immich.controllers.main.containers.main.env` and/or the secret. |

### One database server, individual databases

- **PostgreSQL:** This chart uses the **same PostgreSQL server as Nextcloud**: `postgresql-rw.postgresql.svc.cluster.local` (namespace `postgresql`). Create a separate **database** `immich`; Immich connects as the **postgres** superuser using the same password as the Postgres cluster bootstrap (1Password item **postgresql**). Do not use the same DB name as Nextcloud.
- **Redis/Valkey:** Default is in-chart Valkey. If you prefer a **dedicated Redis** (e.g. `redis-master.redis.svc.cluster.local`), disable Valkey and set `REDIS_HOSTNAME` (and related env) to your Redis host.

---

## Chart contents

- **Immich server + machine-learning:** Upstream `immich-app/immich-charts` (subchart `immich`). Configured for external DB (env + envFrom) and in-chart Valkey or external Redis.
- **Secrets:** ExternalSecret syncs the **postgres** password from the same 1Password item as the Postgres cluster (**postgresql** / **password**) into Kubernetes secret `immich-db-credentials` with `DB_USERNAME=postgres` and `DB_PASSWORD`. Single source of truth; no separate 1Password item for Immich DB.

---

## DB credentials (same as Postgres cluster)

The chart creates a Kubernetes secret **`immich-db-credentials`** with `DB_USERNAME=postgres` and `DB_PASSWORD` using **External Secrets**: it reads from the **same** 1Password item as the Postgres cluster bootstrap.

| Source | Details |
|--------|---------|
| **1Password item** | **postgresql** (same item used by `homelab/helm/postgresql` cluster bootstrap). |
| **Property** | **password** (postgres superuser password). |
| **Kubernetes secret** | `immich-db-credentials` in the release namespace, keys `DB_USERNAME`, `DB_PASSWORD`. |

You do **not** need a separate 1Password item for Immich. Ensure the **postgresql** item exists (for the Postgres cluster) and that External Secrets Operator and ClusterSecretStore (e.g. **onepassword-connect**) are configured; the Immich chart will create `immich-db-credentials` from it.

---

## PostgreSQL database

No separate 1Password item is needed; see [DB credentials (same as Postgres cluster)](#db-credentials-same-as-postgres-cluster) above. On the **external PostgreSQL** side: create the database **immich** (e.g. `CREATE DATABASE immich;`). The **postgres** user already exists and is used by Immich.

---

## Values you must or may set

**Required (external Postgres):**

- **`immich.controllers.main.containers.main.env.DB_HOSTNAME`** – Set to **`postgresql-rw.postgresql.svc.cluster.local`** when using the homelab shared PostgreSQL server (same as Nextcloud). Override only if using a different DB host.
- **`immich.controllers.main.containers.main.env.DB_PORT`** – Port (default `5432`).
- **`immich.controllers.main.containers.main.env.DB_DATABASE_NAME`** – Database name (default `immich`). Must be an **individual database** on the shared server (do not use `nextcloud`).

**Optional:**

- **`externalSecrets.dbCredentials`** – Override 1Password item or secret store (default: item **postgresql**, property **password**, store **onepassword-connect**).
- **`immich.server.controllers.main.ingress.main.hosts[0].host`** – Ingress host (default `immich.expectedbehaviors.com`).
- **`immich.immich.persistence.library.existingClaim`** – PVC for the photo library. Default in this chart: **nextcloud-data** (shared with Nextcloud).
- **`immich.immich.persistence.library.subPath`** – Subdirectory on that PVC for Immich (default **immich-library**). Ensures Immich and Nextcloud do not conflict on the same volume; the subPath directory must exist or be created (e.g. by the chart if it supports subPath).
- **External Redis:** Set `immich.valkey.enabled: false` and add `REDIS_HOSTNAME` (and optionally `REDIS_PORT`, `REDIS_PASSWORD`) to env or the secret.

---

## Deployment hold-off (ArgoCD)

The Immich application is **not deployed by default**. In `homelab/helm/argocd/terraform/argocd-config/configs/config.yaml`, the **immich** project has `applications: []` and the application entry is commented out.

When dependencies are ready (external PostgreSQL with database `immich`, PVC, ESO + postgresql 1Password item, and optionally external Redis), uncomment the application block so ArgoCD syncs the chart. Ensure `DB_HOSTNAME` (and any other DB/Redis overrides) are set in values or a value file used by ArgoCD.

Example Argo CD Application (adjust repo/path/namespace):

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: immich
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/jd4883/homelab-immich
    path: .
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: immich
  syncPolicy:
    automated: { prune: true, selfHeal: true }
```

---

## Install (when dependencies are ready)

- **Helm:** From this directory run `helm dependency update`, then e.g. `helm install immich . -n immich --create-namespace -f values.yaml`. The chart creates `immich-db-credentials` via ExternalSecret from the **postgresql** 1Password item. Ensure the Postgres cluster and ESO are in place and the `immich` database exists.
- **ArgoCD:** Uncomment the immich application in config as above; point source repo to this chart (e.g. `homelab-immich`), path `.`. Use the same namespace; ESO will create the secret from the postgresql item.

---

## Summary checklist (before deploying)

1. **External PostgreSQL:** Database **immich** created (postgres user is used; same creds as cluster bootstrap).
2. **1Password + ESO:** Same **postgresql** item as Postgres cluster; ESO and ClusterSecretStore (e.g. onepassword-connect) create `immich-db-credentials`.
3. **Values:** **DB_HOSTNAME** is set to `postgresql-rw.postgresql.svc.cluster.local` by default.
4. **PVC:** Use **nextcloud-data** with **subPath: immich-library** (default in values) to share the photo volume with Nextcloud, or create a dedicated **immich-library** PVC and set `existingClaim` accordingly.
5. **Optional:** Dedicated Redis – set `immich.valkey.enabled: false` and REDIS_* env.
6. **ArgoCD:** Uncomment the immich application in config when ready to deploy.
