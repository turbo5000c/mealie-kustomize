# mealie-kustomize

Kubernetes manifests for deploying [Mealie](https://mealie.io/) — a self-hosted recipe manager — using [Kustomize](https://kustomize.io/).

Mealie provides a clean web interface for managing recipes, planning meals, and generating shopping lists. It also ships a native [Home Assistant integration](https://www.home-assistant.io/integrations/mealie/) that exposes meal-plan and shopping-list data directly inside Home Assistant dashboards and automations.

---

## Table of Contents

- [What is Mealie?](#what-is-mealie)
- [Home Assistant Integration](#home-assistant-integration)
- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Repository Layout](#repository-layout)
- [Configuration](#configuration)
- [Installation — FluxCD (GitOps)](#installation--fluxcd-gitops)
- [Installation — kubectl](#installation--kubectl)
- [Upgrading](#upgrading)
- [Maintenance](#maintenance)
- [Troubleshooting](#troubleshooting)

---

## What is Mealie?

[Mealie](https://mealie.io/) is an open-source, self-hosted recipe manager and meal planner with:

- 🍽️ Recipe import from any URL
- 📅 Weekly meal planning
- 🛒 Automatic shopping-list generation
- 👨‍👩‍👧 Multi-user support with household groups
- 🔗 REST API and Home Assistant integration

This repository contains the Kustomize manifests needed to run Mealie on any Kubernetes cluster. It is designed to work out of the box for both **FluxCD (GitOps)** and **kubectl** users.

---

## Home Assistant Integration

Mealie ships a first-class [Home Assistant integration](https://www.home-assistant.io/integrations/mealie/).

Once Mealie is deployed and reachable, you can add it to Home Assistant via **Settings → Devices & Services → Add Integration → Mealie**. You will need:

| Field | Value |
|---|---|
| Host | The URL of your Mealie instance (e.g. `https://mealie.yourdomain.com`) |
| API Token | A token generated in Mealie under **Profile → API Tokens** |

After adding the integration, Home Assistant will expose:

- Sensor entities for today's meal plan
- Shopping-list items as to-do list entities
- Service calls for adding items to shopping lists

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│  Kubernetes Cluster                                      │
│                                                          │
│  Namespace: mealie                                       │
│  ┌─────────────┐   ┌────────────────┐   ┌────────────┐  │
│  │  Deployment │──▶│    Service     │──▶│  Ingress / │  │
│  │  (Mealie)   │   │  (ClusterIP    │   │  Ext. LB   │  │
│  └──────┬──────┘   │   port 9000)  │   └────────────┘  │
│         │          └────────────────┘                   │
│  ┌──────▼──────┐                                        │
│  │     PVC     │  (/app/data — recipe media, SQLite     │
│  │  mealie-data│   backups, etc.)                       │
│  └─────────────┘                                        │
│                                                          │
│  ┌─────────────┐   ┌────────────────┐                   │
│  │  ConfigMap  │   │     Secret     │                   │
│  │ mealie-     │   │ mealie-        │                   │
│  │  config     │   │  database-     │                   │
│  └─────────────┘   │   secret       │                   │
│                    └────────────────┘                   │
└─────────────────────────────────────────────────────────┘
                             │
                    ┌────────▼────────┐
                    │  PostgreSQL DB  │
                    │  (external /    │
                    │   in-cluster)   │
                    └─────────────────┘
```

The Mealie container reads configuration from a **ConfigMap** (non-sensitive settings) and a **Secret** (database credentials). Persistent recipe data is stored on a **PersistentVolumeClaim**. An external PostgreSQL server is required; see [Configuration](#configuration) for SQLite-only deployments.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Kubernetes ≥ 1.25 | Any distribution (k3s, k8s, OpenShift, etc.) |
| `kubectl` + `kustomize` | `kustomize` ≥ 4.x or `kubectl` ≥ 1.14 (built-in kustomize) |
| Persistent storage | A StorageClass that supports `ReadWriteOnce` |
| PostgreSQL | External or in-cluster. See note below for SQLite |
| Ingress / load-balancer | To expose Mealie outside the cluster (not included here) |

> **SQLite:** Mealie defaults to SQLite when `DB_ENGINE` is not set. If you prefer SQLite, remove the `DB_ENGINE`, `POSTGRES_*` keys from `configmap.yaml`, remove the `POSTGRES_USER`/`POSTGRES_PASSWORD` env vars from `deployment.yaml`, and comment out (or delete) `secret.yaml` from `kustomization.yaml` — which the default config already does.

---

## Repository Layout

```
mealie-kustomize/
├── kustomization.yaml       # Root Kustomization — references all resources
├── namespace.yaml           # Namespace: mealie
├── configmap.yaml           # Non-sensitive configuration (timezone, DB host, etc.)
├── secret.yaml              # Database credentials (not applied by default — see note)
├── deployment.yaml          # Mealie Deployment
├── pvc.yaml                 # PersistentVolumeClaim for recipe data
├── service.yaml             # ClusterIP Service (port 9000)
├── renovate.json            # Renovate bot config for automatic image updates
│
├── examples/
│   └── secret.yaml          # Example Secret with placeholder values (safe to commit)
│
└── flux/
    ├── gitrepository.yaml   # FluxCD GitRepository pointing at this repo
    └── kustomization.yaml   # FluxCD Kustomization deploying the manifests
```

> **Secret management:** `secret.yaml` is commented out in `kustomization.yaml` by default so that plain-text credentials are never accidentally applied. Use a secrets manager (Sealed Secrets, External Secrets Operator, SOPS) and apply the secret separately, or apply it once manually with `kubectl apply -f secret.yaml` after editing the values.

---

## Configuration

All non-sensitive configuration lives in `configmap.yaml`. Edit this file before applying:

```yaml
# configmap.yaml
data:
  ALLOW_SIGNUP: "false"       # Set to "true" to allow open registration
  TZ: "America/New_York"      # Your timezone (IANA format)
  BASE_URL: "https://mealie.yourdomain.com"  # Public URL of your Mealie instance
  DB_ENGINE: "postgres"       # "postgres" or remove key to use SQLite
  POSTGRES_SERVER: "your-postgres-host"
  POSTGRES_PORT: "5432"
  POSTGRES_DB: "mealie"
```

Database credentials live in `secret.yaml`. Values must be **base64-encoded**:

```bash
# Encode a value
echo -n 'my-secure-password' | base64
```

```yaml
# secret.yaml
data:
  POSTGRES_USER: <base64-encoded username>
  POSTGRES_PASSWORD: <base64-encoded password>
```

See `examples/secret.yaml` for a ready-to-edit template.

### Changing the image version

The image tag in `deployment.yaml` is managed by [Renovate](https://docs.renovatebot.com/). To pin to a specific version, edit `deployment.yaml`:

```yaml
image: ghcr.io/mealie-recipes/mealie:v2.8.0
```

---

## Installation — FluxCD (GitOps)

This is the recommended approach for FluxCD users. Flux will watch this repository and keep the cluster in sync automatically.

### 1. Fork or reference this repository

You can either:
- **Fork** this repo and point Flux at your fork (recommended — lets you commit your own `configmap.yaml`)
- Reference this repo directly and use a Flux `postBuild` substitution or overlay

### 2. Edit configuration

In your fork, update `configmap.yaml` with your values.

Create your secret (using SOPS, Sealed Secrets, or ESO) so that a Secret named `mealie-database-secret` exists in the `mealie` namespace with keys `POSTGRES_USER` and `POSTGRES_PASSWORD`.

### 3. Apply the FluxCD manifests

The `flux/` directory contains ready-to-use examples. Copy them to your Flux cluster repository (e.g. `clusters/my-cluster/mealie/`):

```bash
cp flux/gitrepository.yaml  clusters/my-cluster/mealie/
cp flux/kustomization.yaml  clusters/my-cluster/mealie/
```

Edit `flux/gitrepository.yaml` to point at your fork:

```yaml
spec:
  url: https://github.com/YOUR-ORG/mealie-kustomize
```

Commit and push. Flux will reconcile within the configured interval.

### 4. Verify

```bash
# Watch the Flux Kustomization
flux get kustomization mealie

# Watch pod status
kubectl -n mealie get pods -w
```

---

## Installation — kubectl

Use this method if you are not running FluxCD.

### 1. Clone the repository

```bash
git clone https://github.com/turbo5000c/mealie-kustomize.git
cd mealie-kustomize
```

### 2. Edit configuration

```bash
# Update timezone, base URL, database host, etc.
$EDITOR configmap.yaml
```

### 3. Create the database secret

Copy the example and fill in your credentials:

```bash
cp examples/secret.yaml secret.yaml
# Edit secret.yaml — replace placeholder base64 values with real ones
echo -n 'mydbuser'     | base64   # → paste as POSTGRES_USER
echo -n 'mydbpassword' | base64   # → paste as POSTGRES_PASSWORD
$EDITOR secret.yaml
```

Apply the secret directly (before running kustomize, so it is not managed by kustomize):

```bash
kubectl apply -f secret.yaml
```

### 4. Apply the manifests

```bash
kubectl apply -k .
```

### 5. Expose Mealie

The Service is a `ClusterIP` on port `9000`. Add an Ingress, LoadBalancer, or port-forward to access it:

```bash
# Quick port-forward for testing
kubectl -n mealie port-forward svc/mealie 9000:9000
# Open http://localhost:9000
```

### 6. Verify

```bash
kubectl -n mealie get all
kubectl -n mealie logs -l app=mealie
```

---

## Upgrading

Renovate automatically opens pull requests when a new Mealie image is released. Review and merge the PR to upgrade.

**Manual upgrade:**

1. Update the image tag in `deployment.yaml`:
   ```yaml
   image: ghcr.io/mealie-recipes/mealie:vX.Y.Z
   ```
2. Apply the change:
   ```bash
   # FluxCD: commit and push — Flux reconciles automatically
   # kubectl:
   kubectl apply -k .
   ```
3. Verify the rollout:
   ```bash
   kubectl -n mealie rollout status deployment/mealie
   ```

---

## Maintenance

### Backup

Mealie stores persistent data (media, SQLite database if used) in the PVC mounted at `/app/data/`. Back this up regularly:

```bash
# Example: create a tarball from a temporary pod
kubectl -n mealie run backup --image=busybox --restart=Never \
  --overrides='{"spec":{"volumes":[{"name":"data","persistentVolumeClaim":{"claimName":"mealie-data"}}],"containers":[{"name":"backup","image":"busybox","command":["tar","czf","/tmp/mealie-backup.tar.gz","/app/data"],"volumeMounts":[{"name":"data","mountPath":"/app/data"}]}]}}'
kubectl -n mealie cp backup:/tmp/mealie-backup.tar.gz ./mealie-backup.tar.gz
kubectl -n mealie delete pod backup
```

> For PostgreSQL-backed deployments, back up the database using `pg_dump` in addition to the PVC.

### Scaling

Mealie does not support multi-replica active/active deployments with the default SQLite backend. For PostgreSQL-backed deployments, horizontal scaling is possible but not officially documented by Mealie upstream.

```bash
# Scale down for maintenance
kubectl -n mealie scale deployment/mealie --replicas=0

# Scale back up
kubectl -n mealie scale deployment/mealie --replicas=1
```

---

## Troubleshooting

### Pod is stuck in `Pending`

```bash
kubectl -n mealie describe pod -l app=mealie
```

Common causes:
- **No available PersistentVolume** — ensure your cluster has a default StorageClass.
- **Secret not found** — apply `secret.yaml` before running `kubectl apply -k .`.

### Pod is in `CrashLoopBackOff`

```bash
kubectl -n mealie logs -l app=mealie --previous
```

Common causes:
- Incorrect database credentials or unreachable PostgreSQL server.
- Wrong `BASE_URL` format (must be a full URL including `https://`).

### Cannot reach Mealie from Home Assistant

- Verify the Service is running: `kubectl -n mealie get svc mealie`
- Ensure your Ingress or LoadBalancer is correctly configured and DNS resolves.
- Check that `BASE_URL` in `configmap.yaml` matches the URL you configured in Home Assistant.

### Mealie API token for Home Assistant

1. Log in to Mealie.
2. Go to **Profile (top-right) → Manage Your API Tokens**.
3. Create a new token and copy it.
4. In Home Assistant: **Settings → Devices & Services → Mealie → Configure** and paste the token.

---

## License

This project is provided as-is under the [MIT License](LICENSE). Mealie itself is licensed under the [AGPL-3.0 License](https://github.com/mealie-recipes/mealie/blob/mealie-next/LICENSE).
