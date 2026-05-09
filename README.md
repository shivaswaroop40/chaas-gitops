# Chas GitOps Demo

A GitOps demonstration repository for **Chas Academy, Stockholm** — showcasing [Flux CD](https://fluxcd.io) on a live Kubernetes cluster running in [iximiuz Labs](https://labs.iximiuz.com).

---

## Architecture

```
┌──────────────────────────────────────────────────┐
│  GitHub (chaas-gitops) — single source of truth  │
└──────────────┬───────────────────────────────────┘
               │ Flux polls every 60s
               ▼
┌──────────────────────────────────────────────────┐
│  Flux CD running in-cluster (iximiuz Labs)        │
│  ┌─────────────┐  ┌──────────────┐               │
│  │ GitRepository│  │ Kustomization│               │
│  │ (pull YAML)  │  │ (apply YAML) │               │
│  └─────────────┘  └──────────────┘               │
└──────────────┬───────────────────────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────┐
│  Namespace: prod-stack                            │
│                                                   │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐  │
│  │ frontend  │   │  backend  │   │ postgres  │  │
│  │ 2 replicas│──▶│ 3 replicas│──▶│ 1 replica │  │
│  │  nginx    │   │  nginx    │   │ alpine 15 │  │
│  └─────┬─────┘   └───────────┘   └───────────┘  │
│        │                    ▲                    │
│  ┌─────▼─────┐   ┌───────────┐                  │
│  │  Ingress  │   │    HPA    │                  │
│  │app.example│   │ min:2     │                  │
│  │    .com   │   │ max:10    │                  │
│  └───────────┘   └───────────┘                  │
└──────────────────────────────────────────────────┘
```

### Repository Structure

```
chaas-gitops/
├── README.md
├── aliases                        # kubectl + flux shortcuts
├── flux/
│   ├── source.yaml                # GitRepository — where to pull from
│   ├── kustomizations.yaml        # Flux Kustomization CRs — what to deploy
│   └── image-automation.yaml      # ImagePolicy + ImageUpdateAutomation
├── infrastructure/
│   ├── kustomization.yaml
│   ├── namespace.yaml             # prod-stack namespace
│   ├── secrets.yaml               # db-credentials Secret + api-config ConfigMap
│   └── networkpolicy.yaml         # DB ingress policy (allow backend only)
├── apps/
│   ├── kustomization.yaml
│   ├── database/
│   │   ├── kustomization.yaml
│   │   ├── pvc.yaml               # 10Gi PersistentVolumeClaim
│   │   ├── service.yaml           # Headless ClusterIP for postgres
│   │   └── statefulset.yaml       # Postgres 15-alpine (1 replica)
│   ├── backend/
│   │   ├── kustomization.yaml
│   │   ├── deployment.yaml        # nginx:1.25-alpine (3 replicas)
│   │   ├── service.yaml           # ClusterIP backend-service
│   │   └── hpa.yaml               # CPU-autoscaling 2-10 replicas
│   └── frontend/
│       ├── kustomization.yaml
│       ├── deployment.yaml        # nginx:1.25-alpine (2 replicas)
│       ├── service.yaml           # ClusterIP frontend
│       └── ingress.yaml           # nginx ingress → frontend
└── clusters/
    └── prod/
        └── kustomization.yaml     # Root entry point (flux bootstrap target)
```

---

## Prerequisites

| Tool | Version | Check |
|------|---------|-------|
| Flux CLI | ≥ 2.x | `flux --version` |
| kubectl | ≥ 1.27 | `kubectl version --client` |
| GitHub personal access token (PAT) | `repo` scope | [Generate one](https://github.com/settings/tokens) |
| iximiuz Labs playground | Active session | [Launch here](https://labs.iximiuz.com/playgrounds) |

**Install Flux CLI:**
```bash
curl -s https://fluxcd.io/install.sh | sudo bash
# or
brew install fluxcd/tap/flux
```

---

## Demo Flow

### 1. Pre-flight check

```bash
# Verify cluster access (iximiuz provides a kubeconfig)
kubectl cluster-info
kubectl get nodes
```

### 2. Fork this repository

```bash
# Fork on GitHub, then:
git clone git@github.com:<YOUR_USER>/chaas-gitops.git
cd chaas-gitops
```

### 3. Bootstrap Flux onto the cluster

```bash
export GITHUB_TOKEN=<your-token>
export GITHUB_USER=<your-github-username>

flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=chaas-gitops \
  --branch=main \
  --path=clusters/prod \
  --personal
```

**What this does:**
- Installs Flux controllers in `flux-system` namespace
- Creates a deploy key in the GitHub repo
- Creates `flux-system/gitrepository.yaml` and `flux-system/kustomization.yaml` in the repo (auto-committed)
- Starts reconciling `clusters/prod/` immediately

### 4. Watch the reconciliation

```bash
# Use the aliases from this repo (source aliases)
flux get kustomizations --watch

# Or in another terminal
kubectl get pods -n prod-stack -w
```

Within 60 seconds you should see all resources converging:
```
NAME             READY   STATUS    RESTARTS   AGE
backend-api-...  3/3     Running   0          45s
frontend-...     2/2     Running   0          45s
postgres-0       1/1     Running   0          45s
```

### 5. Demonstrate drift detection (self-healing)

```bash
# Manually scale the frontend
kubectl scale deployment frontend -n prod-stack --replicas=4

# Watch Flux revert it (GitOps reconciliation loop)
kubectl get deployment frontend -n prod-stack -w
# Flux detects the drift and scales it back to 2
```

### 6. Demonstrate declarative updates via Git

```bash
# Edit the frontend replica count in git
vim apps/frontend/deployment.yaml   # change replicas from 2 to 4
git add apps/frontend/deployment.yaml
git commit -m "scale frontend to 4 replicas"
git push

# Flux picks it up automatically
flux get kustomizations
flux reconcile kustomization apps -n flux-system
```

### 7. Demonstrate image automation (advanced)

```bash
# Apply the ImageUpdateAutomation
kubectl apply -f flux/image-automation.yaml

# Check image policy
flux get image policy

# When a new image tag appears, Flux will:
#   1. Detect it via ImageRepository scanning
#   2. Match it against the ImagePolicy (semver range)
#   3. Commit the updated tag back to the repo
#   4. Kustomization picks up the commit and redeploys
```

### 8. Demonstrate dependency ordering

```bash
# Pause infrastructure — apps should stop reconciling
flux suspend kustomization infrastructure -n flux-system

# Delete a backend pod and watch it stay deleted
kubectl delete pod -n prod-stack -l app=backend

# Resume infrastructure — Flux brings everything back
flux resume kustomization infrastructure -n flux-system
```

---

## Flux Features Demonstrated

| Feature | Resource | What it shows |
|---------|----------|---------------|
| **Declarative GitOps** | `GitRepository` | Git as the single source of truth |
| **Automated reconciliation** | `Kustomization` | Flux continuously reconciles (every 5min) |
| **Drift detection** | `Kustomization` | Changes made outside Git are reverted automatically |
| **Dependency ordering** | `dependsOn` | Infrastructure deploys before apps |
| **Image scan & update** | `ImageRepository`, `ImagePolicy`, `ImageUpdateAutomation` | New container images trigger automated commits |
| **Kustomize native** | `kustomization.yaml` files | Overlays, patches, resource composition |
| **Pruning** | `prune: true` | Deleted resources in Git are deleted from cluster |
| **Health checks** | liveness/readiness probes | Flux reports ready/not ready status |

---

## Using with iximiuz Labs

iximiuz Labs provides disposable Kubernetes clusters via a browser-based playground.

1. **Start a playground:** Choose a Kubernetes cluster (e.g., "Kubernetes Cluster 1.29")
2. **Get the kubeconfig:** The playground provides a pre-configured terminal
3. **Install Flux CLI** in the playground:
   ```bash
   curl -s https://fluxcd.io/install.sh | sudo bash
   ```
4. **Expose your local GitHub token:** Copy it to the playground terminal
5. **Follow the demo flow above**

**Note:** iximiuz clusters are ephemeral. When the session ends, the cluster is destroyed. This makes it perfect for a clean-room demo.

---

## Quick Tips

- **Force reconciliation:** `flux reconcile kustomization <name> -n flux-system`
- **Check for errors:** `flux get kustomizations` — any `False` status means something broke
- **View events:** `flux events --all-namespaces`
- **Debug a kustomization:** `kubectl describe kustomization <name> -n flux-system`
- **Suspend/resume:** `flux suspend/resume kustomization <name> -n flux-system`
- **Source aliases:** `source aliases` to load kubectl + flux shortcuts

---

## License

MIT — feel free to use and adapt for your own GitOps demos.
