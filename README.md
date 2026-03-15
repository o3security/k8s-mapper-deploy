# k8s-mapper

**Kubernetes Resource Mapper by O3 Security** — A lightweight agent that continuously tracks your cluster's namespaces, pods, and jobs for security visibility.

## Requirements

- Kubernetes 1.24+
- `kubectl` configured with cluster-admin or sufficient RBAC permissions to create ClusterRoles

## Quick Start

```bash
kubectl apply -f deploy/namespace.yaml
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

The deployment accepts the following arguments (edit `deploy/deployment.yaml` if needed):

| Flag | Default | Description |
|---|---|---|
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
