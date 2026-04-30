# k3d-cluster

Platform repository for local and production Kubernetes cluster configuration. Manages a k3d dev cluster, ArgoCD GitOps bootstrap, an observability stack, and the on-premises tools (Jenkins, GitLab, Nexus, XWiki) served to internal teams.

**Primary goal**: eliminate monthly full-service outages caused by OS maintenance. With Longhorn block storage, nodes can be drained one at a time — each application pod migrates to another node and resumes from its existing data within minutes, instead of going fully offline.

Target environments:
- `dev` (local k3d: 1 server + 3 agents, shared-hostPath storage simulation)
- 5-node k3s cluster on SLES (with Longhorn on a dedicated `/data` partition)

## Repository Layout

```
clusters/        k3d cluster config files
argocd/          ArgoCD install values + bootstrap Applications
argocd/apps/     ArgoCD Application manifests (base + dev/ and prod/ overlays)
observability/   Helm values for the Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy)
storage/         Dev-only: shared-hostPath StorageClass + static PVs (simulates Longhorn)
longhorn/        Prod-only: Longhorn Helm values + README
on-premises/     Helm values + README for Jenkins, GitLab, Nexus, XWiki
docs/            Step-by-step test guides
```

## k3d Cluster Setup

### First-time setup (dev)

The dev cluster mounts `~/k3d-storage` into every node to simulate Longhorn storage. Create the directory before starting the cluster:

```bash
mkdir -p ~/k3d-storage
```

Create the cluster:
```bash
k3d cluster create --config clusters/dev.yaml
```

Verify all 4 nodes are ready:
```bash
kubectl get nodes
# Expected: 1 server + 3 agents, all Ready
```

Verify the shared storage mount is visible in all nodes:
```bash
kubectl debug node/k3d-dev-agent-0 -it --image=busybox -- ls /data/longhorn
# Should return (empty dir) without error — repeat for agent-1, agent-2
```

### Recreating the cluster after config changes

```bash
k3d cluster delete dev
k3d cluster create --config clusters/dev.yaml
```

All PVC data is preserved in `~/k3d-storage` on the WSL host and survives cluster recreation.

### Delete the cluster

```bash
k3d cluster delete dev
```

## Ingress and DNS

k3s ships Traefik as the ingress controller. The cluster maps host ports 8080 and 8443 to the k3d load balancer — no `port-forward` needed.

Dev uses `*.localhost` hostnames. Add entries to the Windows hosts file (since k3d runs in WSL2):

**`C:\Windows\System32\drivers\etc\hosts`**
```
127.0.0.1  argocd.localhost
127.0.0.1  grafana.localhost
127.0.0.1  jenkins.localhost
127.0.0.1  gitlab.localhost
127.0.0.1  nexus.localhost
127.0.0.1  xwiki.localhost
```

| Service | Dev URL | Prod URL |
|---------|---------|----------|
| ArgoCD | http://argocd.localhost:8080 | https://argocd.fhassis.top |
| Grafana | http://grafana.localhost:8080 | https://grafana.fhassis.top |
| Jenkins | http://jenkins.localhost:8080 | https://jenkins.fhassis.top |
| GitLab | http://gitlab.localhost:8080 | https://gitlab.fhassis.top |
| Nexus | http://nexus.localhost:8080 | https://nexus.fhassis.top |
| XWiki | http://xwiki.localhost:8080 | https://xwiki.fhassis.top |
| Longhorn UI | — | https://longhorn.fhassis.top |

## Storage Strategy

| Environment | Storage | How it works |
|-------------|---------|--------------|
| Dev (k3d) | Shared hostPath (`storage/`) | `~/k3d-storage` mounted on all nodes. Static PVs with no `nodeAffinity`. Any node can access the same data. |
| Prod (k3s) | Longhorn (`longhorn/`) | Block volume detaches from drained node, reattaches on the target node. Single replica — mobility, not redundancy. |

All on-premises tools use `storageClass: longhorn` in their Helm values. The name is the same in both environments; only the backing implementation differs.

See [longhorn/README.md](longhorn/README.md) for full details, including SLES prerequisites.

## ArgoCD

GitOps controller in the `argocd` namespace. See [argocd/README.md](argocd/README.md) for install, bootstrap, and access instructions.

### Sync Wave Order

```
Wave -1  dev-storage (dev) OR longhorn (prod) — storage must exist first
Wave  0  prometheus, loki, tempo
Wave  1  grafana
Wave  2  k8s-monitoring (Alloy)
Wave  3  jenkins, nexus, xwiki
Wave  4  gitlab (heaviest — starts last)
```

## Observability

Grafana stack (Prometheus, Loki, Tempo, Grafana, Alloy) in the `observability` namespace. See [observability/README.md](observability/README.md) for install and access instructions.

Pre-loaded Grafana dashboards include Jenkins, GitLab, and Nexus in addition to the existing node, Kubernetes, and RabbitMQ dashboards.

Pod logs from all namespaces (including `jenkins`, `gitlab`, `nexus`, `xwiki`) are collected automatically by Alloy and queryable in Grafana → Explore → Loki.

## On-Premises Tools

Jenkins, GitLab, Nexus OSS, and XWiki deployed in separate namespaces. See [on-premises/README.md](on-premises/README.md) for the overview and links to per-tool documentation.

## Tests

- [Drain Test](docs/drain-test.md) — drain a node and verify an app migrates to another node without data loss. Uses Jenkins but applies to all tools.
- [Jenkins Agent Test](docs/jenkins-agent-test.md) — start a long-running build on a WSL agent, drain the Jenkins controller node mid-build, and verify the build resumes and completes successfully.
