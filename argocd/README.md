# ArgoCD

GitOps controller running in the `argocd` namespace. Manages all Application deployments — including the observability stack — by reconciling the cluster against this git repository.

## Architecture

```
argocd/bootstrap.yaml     ← apply once manually
  └── argocd/apps/        ← ArgoCD watches this directory
        observability.yaml   (5 Applications, sync waves 0→1→2)
```

Each Application uses the **multi-source** pattern (ArgoCD ≥ 2.6):
- Source 1: upstream Helm chart
- Source 2: this git repo, providing the `*.values.yaml` file via `$values` reference

This means: edit a values file, push, ArgoCD reconciles — no manual `helm upgrade`.

## Install

```bash
# Add Helm repo (once per machine)
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace and install
kubectl create namespace argocd
helm install argocd argo/argo-cd \
  -n argocd -f argocd/argocd.values.yaml

# Wait for rollout
kubectl -n argocd rollout status deployment/argocd-server
```

## Access

```bash
# Get the auto-generated admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Port-forward the UI
kubectl -n argocd port-forward svc/argocd-server 8080:80
```

Open <http://localhost:8080> — log in as `admin` / `<password from above>`.

## Bootstrap GitOps

```bash
# Apply the root Application — ArgoCD takes over from here
kubectl apply -f argocd/bootstrap.yaml
```

ArgoCD discovers `argocd/apps/` and deploys all Applications automatically. Watch progress:

```bash
kubectl -n argocd get applications -w
```

The observability stack deploys in waves (0 → 1 → 2), respecting startup ordering.

## Adding new apps

Create a new Application manifest in `argocd/apps/`, commit, and push. ArgoCD picks it up automatically via the bootstrap root app.

## Uninstall

```bash
kubectl delete -f argocd/apps/
kubectl delete -f argocd/bootstrap.yaml
helm uninstall argocd -n argocd
kubectl delete namespace argocd
```

## Upgrading ArgoCD

```bash
helm repo update
helm upgrade argocd argo/argo-cd \
  -n argocd -f argocd/argocd.values.yaml
```

## Adapting for VPS / 5-node k3s

- Ingress + TLS: flip `configs.params.server.insecure` to `false` and add an Ingress.
- SSO: `dex.enabled: true` + OIDC config for multi-user access.
- HA: increase `controller.replicas` and enable `redis-ha.enabled: true`.
- Pin `targetRevision` in all Application manifests to a specific chart version for production.

## Private repo credentials

If this repo is private, register it in ArgoCD after install:

```bash
argocd repo add https://github.com/fhassis/k3d-cluster.git \
  --username fhassis --password YOUR_PAT
```
