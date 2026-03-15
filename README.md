# k8s-mapper

**Kubernetes Supply Chain & Reachability Mapper by O3 Security** — A lightweight agent that continuously tracks your cluster's pods, services, ingresses, and container image provenance (OCI labels, cosign signatures, SLSA attestations).

## Requirements

- Kubernetes 1.24+
- `kubectl` configured with cluster-admin or sufficient RBAC permissions
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
kubectl apply -f deploy/
```

### 4. (Optional) Private Registry Authentication

If your cluster uses private container registries (e.g., AWS ECR, GCR, etc.), create a `registry-credentials` secret so k8s-mapper can inspect image provenance:

```bash
# AWS ECR example
TOKEN=$(aws ecr get-login-password --region <REGION>)
kubectl create secret docker-registry registry-credentials \
  --docker-server=<ACCOUNT_ID>.dkr.ecr.<REGION>.amazonaws.com \
  --docker-username=AWS \
  --docker-password=$TOKEN \
  -n o3-security
```

> **Note:** ECR tokens expire after 12h. For a permanent solution, use [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) — the inspector will automatically use IRSA credentials if configured.

**Auth strategies are tried in order (smart fallback chain):**
1. **K8s secret** — reads the `registry-credentials` secret (this step)
2. **ECR SDK** — uses IRSA / instance profile / env credentials
3. **Docker config** — `~/.docker/config.json`
4. **Anonymous** — public registries (Docker Hub, GHCR, etc.)

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

k8s-mapper requires **read-only** access. No write access, no broad secrets access.

| Resource | API Group | Verbs | Purpose |
|---|---|---|---|
| `namespaces` | core (`""`) | `get`, `list`, `watch` | Track namespace lifecycle |
| `pods` | core (`""`) | `get`, `list`, `watch` | Track pod images & metadata |
| `services` | core (`""`) | `get`, `list`, `watch` | Internet reachability mapping |
| `ingresses` | `networking.k8s.io` | `get`, `list`, `watch` | Internet reachability mapping |
| `secrets` (scoped) | core (`""`) | `get` | **Only** the `registry-credentials` secret for registry auth |

> The secrets permission uses `resourceNames: ["registry-credentials"]` — k8s-mapper can only read this ONE specific secret, not any other secrets in the cluster.

To audit the exact permissions before deploying:

```bash
cat deploy/rbac.yaml
```

## What Gets Tracked

| Category | Resources | Purpose |
|---|---|---|
| **Supply Chain** | Pods, container images, OCI labels, cosign signatures, SLSA provenance | Image-to-source-repo mapping, signature verification |
| **Reachability** | Services, Ingresses | Identify internet-exposed workloads |
| **Grouping** | Namespaces | Organizational context |

## Architecture

- **Single Deployment** (not DaemonSet) — one pod watches the entire cluster via the K8s API server
- **Event-driven** — uses Kubernetes SharedInformers (watch streams), zero polling
- **~15-30 MB** memory at steady state
- **Distroless image** — no shell, minimal attack surface, runs as non-root

## Configuration

| Env Var / Flag | Default | Description |
|---|---|---|
| `O3_API_KEY` / `--api-key` | **(required)** | O3 Security API key |
| `O3_API_URL` / `--api-url` | **(required)** | O3 Security GraphQL endpoint |
| `O3_CLUSTER_NAME` / `--cluster-name` | `default` | Display name for this cluster |
| `--sync-interval` | `60s` | How often to push data to O3 backend |
| `--addr` | `:8080` | HTTP server listen address |
| `--log-level` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `REGISTRY_SECRET_NAME` | `registry-credentials` | K8s secret name for registry auth |
| `REGISTRY_SECRET_NAMESPACE` | `o3-security` | Namespace of the registry secret |

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
