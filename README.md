# k3d-cluster

Platform repository for local and production Kubernetes cluster configuration. Manages a k3d dev cluster, ArgoCD GitOps bootstrap, and the Grafana observability stack.

Target environments:
- `k3d-dev` (local single node)
- Single-node VPS running k3s
- 5-node k3s cluster

## Repository layout

```
clusters/       k3d cluster config files
argocd/         ArgoCD install values + bootstrap Application
argocd/apps/    ArgoCD Application manifests (watched by bootstrap)
observability/  Helm values for the Grafana observability stack
```

## k3d Cluster Setup

```bash
# Create
k3d cluster create --config clusters/k3d-dev.yaml

# Delete
k3d cluster delete k3d-dev
```

## ArgoCD

GitOps controller running in the `argocd` namespace. See [argocd/README.md](argocd/README.md) for install, bootstrap, and access instructions.

## Observability

Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy) in the `observability` namespace. See [observability/README.md](observability/README.md) for install and access instructions.
