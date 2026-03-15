# k8s-mapper

**Kubernetes Supply Chain & Reachability Mapper by O3 Security** — A lightweight agent that continuously tracks your cluster's pods, services, ingresses, and container image provenance (OCI labels, cosign signatures, SLSA attestations).

## Requirements

- Kubernetes 1.24+ (EKS, GKE, AKS, or self-managed)
- `kubectl` configured with cluster-admin or sufficient RBAC permissions
- **O3 Security API key** — generate from **Settings → API Keys** in your [O3 Security dashboard](https://app.codexsecurity.io)

## Quick Start

### 1. Create the namespace and API key secret

```bash
kubectl create namespace o3-security

kubectl create secret generic o3-security-credentials \
  --namespace o3-security \
  --from-literal=api-key=YOUR_API_KEY_HERE
```

### 2. Configure your cluster name

Edit `deploy/deployment.yaml` and set:
- `O3_CLUSTER_NAME` — a display name for this cluster (e.g., `production`, `staging`)
- `O3_API_URL` — your O3 Security GraphQL endpoint (default: `https://api.codexsecurity.io/graphql`)

### 3. Apply the manifests

```bash
kubectl apply -f deploy/
```

### 4. Verify Installation

```bash
# Check pod is running
kubectl get pods -n o3-security

# Check readiness
kubectl exec -n o3-security deploy/k8s-mapper -- wget -qO- http://localhost:8080/readyz

# View tracked resource counts
kubectl exec -n o3-security deploy/k8s-mapper -- wget -qO- http://localhost:8080/api/v1/stats
```

## Private Registry Authentication (ECR)

k8s-mapper uses `hostNetwork: true` to access the EC2 Instance Metadata Service (IMDS) directly — the **same way kubelet authenticates** to ECR for image pulls. No secrets, no IRSA needed.

### How it works

```
k8s-mapper pod (hostNetwork: true)
  → Uses node's network namespace (1 hop to IMDS)
  → Gets node's IAM role credentials
  → Calls ecr:GetAuthorizationToken
  → Authenticates to private ECR registries
```

This is the same mechanism that kubelet, `aws-node`, and `kube-proxy` use.

### Auth Strategy Chain

k8s-mapper tries authentication strategies in order until one works:

| Priority | Strategy | How it works | When to use |
|----------|----------|-------------|-------------|
| 1 | **K8s Secret** | Reads `registry-credentials` secret | Non-EKS clusters or custom registries |
| 2 | **ECR SDK** | IMDS / IRSA / env credentials | EKS with `hostNetwork: true` (default) |
| 3 | **Docker Config** | `~/.docker/config.json` | Local development |
| 4 | **Anonymous** | No auth | Public registries (Docker Hub, GHCR) |

### (Optional) Custom Registry Secret

For non-ECR registries or if you prefer explicit credentials:

```bash
kubectl create secret docker-registry registry-credentials \
  --docker-server=<REGISTRY_URL> \
  --docker-username=<USERNAME> \
  --docker-password=<PASSWORD> \
  -n o3-security
```

## RBAC Permissions

k8s-mapper requires **read-only** access. No write access, no secrets access.

| Resource | API Group | Verbs | Purpose |
|---|---|---|---|
| `namespaces` | core (`""`) | `get`, `list`, `watch` | Track namespace lifecycle |
| `pods` | core (`""`) | `get`, `list`, `watch` | Track pod images & metadata |
| `services` | core (`""`) | `get`, `list`, `watch` | Internet reachability mapping |
| `ingresses` | `networking.k8s.io` | `get`, `list`, `watch` | Internet reachability mapping |

To audit the exact permissions:

```bash
cat deploy/rbac.yaml
```

## What Gets Tracked

| Category | What | Purpose |
|---|---|---|
| **Supply Chain** | OCI labels, cosign signatures, SLSA provenance | Image-to-source-repo mapping, signature verification |
| **Reachability** | Services, Ingresses | Identify internet-exposed workloads |
| **Grouping** | Namespaces | Organizational context |

## Architecture

- **Single Deployment** — one pod watches the entire cluster via the K8s API server
- **Event-driven** — Kubernetes SharedInformers (watch streams), zero polling
- **hostNetwork** — shares node's network for IMDS access (ECR auth)
- **~15-30 MB** memory at steady state
- **Distroless image** — no shell, minimal attack surface, runs as non-root

## Configuration

| Env Var / Flag | Default | Description |
|---|---|---|
| `O3_API_KEY` / `--api-key` | **(required)** | O3 Security API key |
| `O3_API_URL` / `--api-url` | **(required)** | O3 Security GraphQL endpoint |
| `O3_CLUSTER_NAME` / `--cluster-name` | `default` | Display name for this cluster |
| `--sync-interval` | `60s` | Push interval to backend |
| `--addr` | `:8080` | HTTP server listen address |
| `--log-level` | `info` | Log level: `debug`, `info`, `warn`, `error` |
| `REGISTRY_SECRET_NAME` | `registry-credentials` | K8s secret name for registry auth |
| `REGISTRY_SECRET_NAMESPACE` | `o3-security` | Namespace of the registry secret |

## Debugging

### Check if ECR auth is working

```bash
kubectl logs -n o3-security deploy/k8s-mapper | grep ecr-keychain
```

**Success:**
```
[ecr-keychain] obtained ECR token for 471112898140.dkr.ecr.ap-south-1.amazonaws.com (expires ...)
```

**Failure (IMDS unreachable):**
```
[ecr-keychain] ECR auth unavailable — IRSA not configured?
```
→ Ensure `hostNetwork: true` is set in the deployment spec.

### Check if images are being inspected

```bash
kubectl logs -n o3-security deploy/k8s-mapper | grep inspector
```

You should see entries like:
```
[inspector] 471112898140.dkr.ecr.../codex/backend-private status=signed
```

### Verify hostNetwork is enabled

```bash
kubectl get pod -n o3-security -l app.kubernetes.io/name=k8s-mapper \
  -o jsonpath='{.items[0].spec.hostNetwork}' && echo
```

Should print `true`.

### Common Issues

| Symptom | Cause | Fix |
|---|---|---|
| `401 Unauthorized` for ECR images | hostNetwork not enabled | Apply latest `deployment.yaml` with `hostNetwork: true` |
| `status=unsigned` for signed images | Manifest list digest mismatch | k8s-mapper resolves multi-arch digests automatically |
| `status=signed_unverified` | Cosign v2+ DSSE PAE encoding | Ensure latest k8s-mapper image is deployed |
| No OCI labels / source mapping | Missing `--label` in `docker buildx` | Add `org.opencontainers.image.source` label to your CI |
| SLSA Provenance blank | No `cosign attest` in CI | Add SLSA attestation step to your build workflow |

## CI Setup for Image Signing

To enable cosign signing and SLSA provenance in your GitHub Actions:

```yaml
# After docker buildx build --push
- uses: sigstore/cosign-installer@v3

- name: Sign image
  run: |
    IMAGE_DIGEST=$(docker buildx imagetools inspect $REGISTRY/$IMAGE:latest --format '{{json .Manifest.Digest}}' | tr -d '"')
    cosign sign --yes $REGISTRY/$IMAGE@${IMAGE_DIGEST}

- name: SLSA Provenance
  run: |
    cosign attest --yes --type slsaprovenance \
      --predicate provenance.json \
      $REGISTRY/$IMAGE@${IMAGE_DIGEST}
```

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

For issues or questions, contact [support@codexsecurity.io](mailto:support@codexsecurity.io).
