# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Infrastructure-as-code for a local k3d dev cluster and future k3s environments. There is no application code — everything here is Kubernetes/Helm configuration consumed by ArgoCD. Changes take effect by pushing to git; ArgoCD reconciles the cluster automatically.

Target environments: `dev` (local, 3-node k3d), single-node VPS, 5-node k3s cluster.

## Cluster lifecycle

```bash
# Create the k3d dev cluster (sets kubeconfig automatically)
k3d cluster create --config clusters/dev.yaml

# Delete it
k3d cluster delete dev
```

## ArgoCD

```bash
# Install (run once after cluster create)
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f argocd/argocd.values.yaml
kubectl -n argocd rollout status deployment/argocd-server

# Bootstrap GitOps (run once — ArgoCD takes over from here)
kubectl apply -f argocd/bootstrap.yaml

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Access UI
kubectl -n argocd port-forward svc/argocd-server 8080:80
# → http://localhost:8080  (admin / <password above>)

# Watch sync status
kubectl -n argocd get applications -w
```

## Observability stack (manual install, bypasses ArgoCD)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update
kubectl create namespace observability

# Backends first (wave 0), then Grafana (wave 1), then collection (wave 2)
helm install prometheus prometheus-community/prometheus -n observability -f observability/prometheus.values.yaml
helm install loki grafana-community/loki -n observability -f observability/loki.values.yaml
helm install tempo grafana-community/tempo -n observability -f observability/tempo.values.yaml
helm install grafana grafana-community/grafana -n observability -f observability/grafana.values.yaml
helm install k8s-monitoring grafana/k8s-monitoring -n observability -f observability/k8s-monitoring.values.yaml

# Access Grafana
kubectl -n observability port-forward svc/grafana 3000:80
# → http://localhost:3000  (admin / admin)
```

## Architecture

### GitOps flow

```
git push
  └── ArgoCD detects change
        └── ArgoCD applies Helm release with updated values
```

`argocd/bootstrap.yaml` is a single ArgoCD Application applied once manually. It points ArgoCD at `argocd/apps/`. Every manifest added to `argocd/apps/` is automatically discovered and managed — no further manual `kubectl apply` needed.

### Multi-source Helm pattern

Every Application in `argocd/apps/` uses ArgoCD's multi-source feature (≥ 2.6):
- **source[0]**: upstream Helm chart repo
- **source[1]**: this git repo, referenced as `$values`, providing the `*.values.yaml` file

To change a chart's config: edit the relevant `observability/*.values.yaml`, push — ArgoCD reconciles.

### Observability sync waves

`argocd/apps/observability.yaml` deploys five Applications in dependency order:
- **Wave 0** — Prometheus, Loki, Tempo (independent backends)
- **Wave 1** — Grafana (datasources wired to wave 0)
- **Wave 2** — k8s-monitoring/Alloy (remote-writes to wave 0 backends)

### OTLP ingestion

Apps send traces to the Alloy singleton (not the DaemonSet):
- gRPC: `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4317`
- HTTP: `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4318`

Set `OTEL_EXPORTER_OTLP_ENDPOINT` in workload manifests to one of these.

## Key conventions

- `targetRevision: "*"` (latest chart) is used in dev; pin to a semver for production.
- All values files use `local-path` StorageClass (k3d/k3s default) and single-replica mode — no HA anywhere.
- The Grafana admin password (`admin`) is committed intentionally for dev. See `observability/README.md` § Secrets for the migration path (SOPS + age + helm-secrets) before exposing publicly.
- Adding a new app: create a manifest in `argocd/apps/`, commit, push. No other step required.

## Production hardening notes (not implemented yet)

- ArgoCD: flip `server.insecure: false`, add Ingress, enable Dex SSO, increase replicas, `redis-ha.enabled: true`.
- Observability: add Ingress + TLS for Grafana, larger PVCs, longer Prometheus retention, `alertmanager.enabled: true`, object storage for Loki/Tempo.
- Grafana Cloud migration: change only the `destinations` block in `k8s-monitoring.values.yaml`; backends and app workloads need no changes.
