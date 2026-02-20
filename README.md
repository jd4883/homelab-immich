# Immich (self-hosted photo and video backup)

> **Not ready for deployment yet.** The ArgoCD application for Immich is commented out. Do not enable it until external PostgreSQL (and other dependencies below) are in place and values are set. See [Dependencies (have these before deploying)](#dependencies-have-these-before-deploying) and [Deployment hold-off (ArgoCD)](#deployment-hold-off-argocd).

Immich runs on Kubernetes via the [official Immich Helm chart](https://github.com/immich-app/immich-charts) with **external PostgreSQL**, in-chart **Valkey** (Redis), and **1Password-backed secrets** for DB credentials. This chart does **not** deploy PostgreSQL; you must have an external database and other dependencies ready before deploying.

---

## Dependencies (have these before deploying)

Deploy or provision the following **before** installing Immich or enabling the ArgoCD application:

| Dependency | Required | Notes |
|------------|----------|--------|
| **External PostgreSQL** | Yes | **One database server** for homelab (same as Nextcloud): host `postgresql-rw.postgresql.svc.cluster.local`. Create an **individual database** `immich` and a dedicated user; credentials (`DB_USERNAME`, `DB_PASSWORD`) come from the 1Password-backed secret `immich-db-credentials`. |
| **PersistentVolumeClaim** (library) | Yes | Default: share **nextcloud-data** PVC with Nextcloud using `existingClaim: nextcloud-data` and `subPath: immich-library` so the Immich library lives in a fixed directory on the same volume. Ensure the chart supports `subPath` for library, or create a dedicated `immich-library` PVC. |
| **1Password item → secret `immich-db-credentials`** | Yes | Item must contain `DB_USERNAME` and `DB_PASSWORD` so the onepassworditem subchart can create the secret. See [1Password item](#1password-item-and-secret-keys) below. |
| **External Redis (optional)** | No | By default the chart runs Valkey (Redis-compatible) in-chart. For a dedicated Redis, set `immich.valkey.enabled: false` and add `REDIS_HOSTNAME` (and optionally `REDIS_PORT`, `REDIS_PASSWORD`) to `immich.controllers.main.containers.main.env` and/or the secret. |

### One database server, individual databases

- **PostgreSQL:** This chart uses the **same PostgreSQL server as Nextcloud**: `postgresql-rw.postgresql.svc.cluster.local` (namespace `postgresql`). Create a separate **database** `immich` and a dedicated **user** with access to it (e.g. via CloudNativePG or init scripts). Do not use the same DB name as Nextcloud.
- **Redis/Valkey:** Default is in-chart Valkey. If you prefer a **dedicated Redis** (e.g. `redis-master.redis.svc.cluster.local`), disable Valkey and set `REDIS_HOSTNAME` (and related env) to your Redis host.

---

## Chart contents

- **Immich server + machine-learning:** Upstream `immich-app/immich-charts` (subchart `immich`). Configured for external DB (env + envFrom) and in-chart Valkey or external Redis.
- **Secrets:** [onepassworditem](https://github.com/vquie/helm-charts) subchart syncs a 1Password item into a Kubernetes secret `immich-db-credentials` used for `DB_USERNAME` and `DB_PASSWORD`.

---

## 1Password item and secret keys

The chart expects a Kubernetes secret named **`immich-db-credentials`** with at least:

| Key           | Description |
|---------------|-------------|
| `DB_USERNAME` | Database username for Immich (e.g. `immich` or `postgres`). |
| `DB_PASSWORD` | Database password for that user. |

The **onepassworditem** subchart creates this secret from a 1Password item. Create an item (e.g. in vault **Kubernetes**, name **immich-db-credentials**) and add fields whose **labels** (or names) match the keys above so they sync into the secret.

In `values.yaml` the default item path is:

```yaml
onepassworditem:
  secrets:
    immich:
      - item: vaults/Kubernetes/items/immich-db-credentials
        name: immich-db-credentials
        type: Opaque
```

Change `item` to your vault/item path so it matches your 1Password setup.

---

## What you need to generate in 1Password

1. **Create a 1Password item** (e.g. **immich-db-credentials** in vault **Kubernetes**).
2. **Add fields** that map to Kubernetes secret keys:
   - **DB_USERNAME** – Database user for Immich (e.g. `immich` or `postgres`).
   - **DB_PASSWORD** – That user’s password (must match the password on your external PostgreSQL).
3. Ensure **1Password Connect** (and the Kubernetes operator, if used) can read this item and that the secret is created in the same namespace as the Immich release (e.g. **immich**).

On the **external PostgreSQL** side: create a database (e.g. `immich`) and a user with access to it, using the same username and password you put in 1Password.

---

## Values you must or may set

**Required (external Postgres):**

- **`immich.controllers.main.containers.main.env.DB_HOSTNAME`** – Set to **`postgresql-rw.postgresql.svc.cluster.local`** when using the homelab shared PostgreSQL server (same as Nextcloud). Override only if using a different DB host.
- **`immich.controllers.main.containers.main.env.DB_PORT`** – Port (default `5432`).
- **`immich.controllers.main.containers.main.env.DB_DATABASE_NAME`** – Database name (default `immich`). Must be an **individual database** on the shared server (do not use `nextcloud`).

**Optional:**

- **`onepassworditem.secrets.immich[0].item`** – 1Password item path.
- **`immich.server.controllers.main.ingress.main.hosts[0].host`** – Ingress host (default `immich.expectedbehaviors.com`).
- **`immich.immich.persistence.library.existingClaim`** – PVC for the photo library. Default in this chart: **nextcloud-data** (shared with Nextcloud).
- **`immich.immich.persistence.library.subPath`** – Subdirectory on that PVC for Immich (default **immich-library**). Ensures Immich and Nextcloud do not conflict on the same volume; the subPath directory must exist or be created (e.g. by the chart if it supports subPath).
- **External Redis:** Set `immich.valkey.enabled: false` and add `REDIS_HOSTNAME` (and optionally `REDIS_PORT`, `REDIS_PASSWORD`) to env or the secret.

---

## Deployment hold-off (ArgoCD)

The Immich application is **not deployed by default**. In `homelab/helm/argocd/terraform/argocd-config/configs/config.yaml`, the **immich** project has `applications: []` and the application entry is commented out.

When dependencies are ready (external PostgreSQL, PVC, 1Password item, and optionally external Redis), uncomment the application block so ArgoCD syncs the chart. Ensure `DB_HOSTNAME` (and any other DB/Redis overrides) are set in values or a value file used by ArgoCD.

---

## Install (when dependencies are ready)

- **Helm:** From this directory run `helm dependency update`, then e.g. `helm install immich . -n immich --create-namespace -f values.yaml`. Set `immich.controllers.main.containers.main.env.DB_HOSTNAME` (e.g. via `--set immich.controllers.main.containers.main.env.DB_HOSTNAME=your-postgres-host` or a custom values file). Ensure the `immich-db-credentials` secret exists and the `immich-library` PVC exists.
- **ArgoCD:** Uncomment the immich application in config as above; point source repo to this chart (e.g. `homelab-immich`), path `.`. Use the same namespace and ensure the 1Password item and external DB are available.

---

## Summary checklist (before deploying)

1. **External PostgreSQL:** Database and user created; note host, port, database name.
2. **1Password:** Item with **DB_USERNAME** and **DB_PASSWORD**; set `onepassworditem.secrets.immich[0].item` in values.
3. **Values:** Set **DB_HOSTNAME** (and DB_PORT/DB_DATABASE_NAME if different from defaults).
4. **PVC:** Use **nextcloud-data** with **subPath: immich-library** (default in values) to share the photo volume with Nextcloud, or create a dedicated **immich-library** PVC and set `existingClaim` accordingly.
5. **Optional:** Dedicated Redis – set `immich.valkey.enabled: false` and REDIS_* env.
6. **ArgoCD:** Uncomment the immich application in config when ready to deploy.
