# 📷 Add Immich Helm chart and ArgoCD project (incomplete – do not deploy)

**Section:** New chart + Argo CD project; app not deployed until dependencies ready.

---

## 🎯 TL;DR

| **What** | **Why safe** | **Proof** |
|----------|--------------|------------|
| Add **Immich** Helm chart (official subchart + onepassworditem) and Argo CD project. **One database server** (same as Nextcloud): `postgresql-rw.postgresql.svc.cluster.local`; **individual database** `immich`. **App not deployed** until that Postgres (and DB/user), PVC, and 1Password credentials are in place. | Chart and project are additive; no deploy until user enables. DB host set to homelab shared Postgres; credentials from 1Password; library uses existingClaim + subPath (e.g. nextcloud-data/immich-library). | `helm template` ✅ — Services (server, machine-learning, valkey), Deployments, Ingress (immich.expectedbehaviors.com) render; deploy held off until deps ready. |

---

## 📦 Summary

| Icon | Change |
|------|--------|
| 📷 | **Immich chart** — Official immich-charts 0.10.3 + onepassworditem for DB credentials. Controllers: server, machine-learning, valkey. |
| 🔐 | **DB credentials** — envFrom secretRef `immich-db-credentials`; DB_HOSTNAME set to **postgresql-rw.postgresql.svc.cluster.local** (same server as Nextcloud); individual database **immich**. |
| 💾 | **Persistence** — Library: existingClaim (e.g. nextcloud-data) + subPath immich-library; configuration via chart. |
| 🌐 | **Ingress** — immich.expectedbehaviors.com, path `/`, nginx proxy size/timeouts, external-dns. |
| 📖 | **Incomplete** — Do not deploy until: shared PostgreSQL (with DB `immich` and user created), PVC/subPath, 1Password item for DB credentials. |

---

## 🔧 Setup requirements

**Before enabling deploy:** (1) **One database server** at `postgresql-rw.postgresql.svc.cluster.local` with an **individual database** `immich` and a dedicated user; (2) PVC (or existingClaim + subPath for library); (3) 1Password item and secret `immich-db-credentials` with DB_USERNAME, DB_PASSWORD. See README.

---

## ✅ Render & validation

> **Command used:**  
> `helm dependency update && helm template immich . -f values.yaml --namespace default`

| Check | Result |
|-------|--------|
| `helm dependency update` | ✅ OK |
| `helm template immich . -f values.yaml --namespace default` | ✅ OK, no errors |
| **Rendered** | Services (immich-server, immich-machine-learning, immich-valkey); Deployments; Ingress for server |

---

## 📋 Supporting evidence (rendered manifests)

<details>
<summary>🔐 <b>Server Service — ClusterIP port 2283</b></summary>

```yaml
# Source: immich/charts/immich/templates/server.yaml
apiVersion: v1
kind: Service
metadata:
  name: immich-server
  labels:
    app.kubernetes.io/instance: immich
    app.kubernetes.io/name: server
    helm.sh/chart: immich-0.10.3
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 2283
      targetPort: http
      protocol: TCP
      name: http
  selector:
    app.kubernetes.io/controller: main
    app.kubernetes.io/instance: immich
    app.kubernetes.io/name: server
```

</details>

<details>
<summary>🌐 <b>Ingress — single host, proxy settings</b></summary>

```yaml
# Ingress for immich-server (from chart)
# host: immich.expectedbehaviors.com
# path: /
# annotations: nginx proxy-body-size 0, proxy timeouts 3600, external-dns
```

</details>

---

## 🛡️ Why each change is safe & correct

| Change | What we did | Why it's safe | Proof in render |
|--------|-------------|---------------|-----------------|
| Add chart | Official Immich chart + onepassworditem dependency. | Additive; no impact until deployed. | helm template succeeds; Services/Deployments present. |
| DB credentials | envFrom secretRef immich-db-credentials; DB_* in values. | No secrets in repo; 1Password at deploy time. | Deployment envFrom in rendered manifest. |
| Library path | existingClaim + subPath (e.g. nextcloud-data/immich-library). | Isolated from other consumers of same PVC. | Persistence config in values; render shows volume. |
| Incomplete notice | README + PR state "do not deploy" until deps ready. | Prevents deploy before Postgres/PVC/1Password. | N/A. |

---

## 🚀 Next steps

| Step | Action |
|------|--------|
| 1️⃣ | **Merge** — Chart and Argo CD project added; do not enable deploy yet. |
| 2️⃣ | **Before enabling** — Ensure shared PostgreSQL has database `immich` and a user (e.g. via postgresql chart or init); create PVC or configure existingClaim + subPath; create 1Password item and Kubernetes secret `immich-db-credentials`; DB_HOSTNAME is already set to `postgresql-rw.postgresql.svc.cluster.local`. |
| 3️⃣ | **Enable deploy** — Sync Argo CD application (or helm upgrade --install) after deps ready. |
| 4️⃣ | **Smoke-test** — Immich UI loads; library and DB connectivity verified. |
