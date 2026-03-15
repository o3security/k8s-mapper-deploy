# k8s-mapper

**Kubernetes Resource Mapper by O3 Security** — A lightweight agent that continuously tracks your cluster's namespaces, pods, and jobs for security visibility.

## Requirements

- Kubernetes 1.24+
- `kubectl` configured with cluster-admin or sufficient RBAC permissions to create ClusterRoles
- **O3 Security API key** — generate from **Settings → API Keys** in your [O3 Security dashboard](https://app.o3security.com)

## Quick Start

### 1. Create the O3 Security API key secret

```bash
kubectl create namespace o3-security

kubectl create secret generic o3-security-credentials \
  --namespace o3-security \
  --from-literal=api-key=YOUR_API_KEY_HERE
```

> Replace `YOUR_API_KEY_HERE` with the API key from your O3 Security dashboard.

### 2. Configure your cluster name and API URL

Edit `deploy/deployment.yaml` and set:
- `O3_CLUSTER_NAME` — a display name for this cluster (e.g., `production`, `staging`)
- `O3_API_URL` — your O3 Security GraphQL endpoint (default: `https://api.o3security.com/graphql`)

### 3. Apply the manifests

```bash
kubectl apply -f deploy/rbac.yaml
kubectl apply -f deploy/deployment.yaml
```

Or in one shot:

```bash
kubectl apply -f deploy/
```

### Verify Installation

```bash
# Check pod is running
kubectl get pods -n o3-security

# Check readiness (200 = synced with API server)
kubectl exec -n o3-security deploy/k8s-mapper -- wget -qO- http://localhost:8080/readyz

# View tracked resource counts
kubectl exec -n o3-security deploy/k8s-mapper -- wget -qO- http://localhost:8080/api/v1/stats
```

## RBAC Permissions

k8s-mapper requires **read-only** access to 3 resource types. No write access, no secrets, no configmaps.

| Resource | API Group | Verbs | Purpose |
|---|---|---|---|
| `namespaces` | core (`""`) | `get`, `list`, `watch` | Track namespace lifecycle |
| `pods` | core (`""`) | `get`, `list`, `watch` | Track pod metadata & container images |
| `jobs` | `batch` | `get`, `list`, `watch` | Track job metadata & container images |

To audit the exact permissions before deploying:

```bash
cat deploy/rbac.yaml
```

## Architecture

- **Single Deployment** (not DaemonSet) — one pod watches the entire cluster via the K8s API server
- **Event-driven** — uses Kubernetes SharedInformers (watch streams), zero polling
- **~15-30 MB** memory at steady state
- **Distroless image** — no shell, minimal attack surface, runs as non-root

## Resource Limits

| | Request | Limit |
|---|---|---|
| CPU | 50m | 200m |
| Memory | 64Mi | 256Mi |

## Configuration

| Env Var / Flag | Default | Description |
|---|---|---|
| `O3_API_KEY` / `--api-key` | **(required)** | O3 Security API key |
| `O3_API_URL` / `--api-url` | **(required)** | O3 Security GraphQL endpoint |
| `O3_CLUSTER_NAME` / `--cluster-name` | `default` | Display name for this cluster |
| `--sync-interval` | `60s` | How often to push data to O3 backend |
| `--addr` | `:8080` | HTTP server listen address |
| `--log-level` | `info` | Log level: `debug`, `info`, `warn`, `error` |

## Health Endpoints

| Endpoint | Purpose |
|---|---|
| `GET /healthz` | Liveness probe (always 200) |
| `GET /readyz` | Readiness probe (200 after initial sync) |

## Uninstall

```bash
kubectl delete -f deploy/
```

## Support

For issues or questions, contact [support@o3security.com](mailto:support@o3security.com).
