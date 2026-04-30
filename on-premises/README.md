# On-Premises Tools

This directory contains Helm values files and documentation for the four tools served internally to other teams. Each tool runs as a single-replica deployment backed by a Longhorn PVC, enabling rolling OS maintenance via `kubectl drain` without full service outages.

## Tools Overview

| Tool | Namespace | Dev URL | Prod URL | Metrics |
|------|-----------|---------|----------|---------|
| Jenkins | `jenkins` | http://jenkins.localhost:8080 | https://jenkins.fhassis.top | `/prometheus` |
| GitLab | `gitlab` | http://gitlab.localhost:8080 | https://gitlab.fhassis.top | `/metrics` (built-in) |
| Nexus OSS | `nexus` | http://nexus.localhost:8080 | https://nexus.fhassis.top | `/service/metrics/prometheus` |
| XWiki | `xwiki` | http://xwiki.localhost:8080 | https://xwiki.fhassis.top | None (JMX only) |

For detailed documentation on each tool — chart choice, what is and is not configured, and production hardening notes — see the per-tool READMEs:

- [Jenkins →](jenkins/README.md)
- [GitLab →](gitlab/README.md)
- [Nexus OSS →](nexus/README.md)
- [XWiki →](xwiki/README.md)

## Storage Strategy

All tools use `storageClass: longhorn` in their Helm values. The underlying implementation differs by environment:

| Environment | Storage | How it works |
|-------------|---------|--------------|
| Dev (k3d) | Shared hostPath (see `storage/`) | `~/k3d-storage` is mounted into all k3d nodes. Static PVs with no `nodeAffinity` allow any node to mount the same data. |
| Prod (k3s) | Longhorn block storage | Longhorn detaches the volume from a draining node and reattaches it on the target node. |

See [Longhorn README](../longhorn/README.md) for details on how the dev simulation works and prod prerequisites.

## Startup Order (Sync Waves)

ArgoCD deploys components in this order across all environments:

```
Wave -1  →  dev-storage (dev) or longhorn (prod) — storage must exist first
Wave  0  →  prometheus, loki, tempo
Wave  1  →  grafana
Wave  2  →  k8s-monitoring (Alloy)
Wave  3  →  jenkins, nexus, xwiki
Wave  4  →  gitlab  (heaviest — starts last to reduce node pressure)
```

## Observability

**Logs**: Alloy's `podLogsViaLoki` collects logs from all namespaces automatically. To see Jenkins logs in Grafana: Explore → Loki → `{namespace="jenkins"}`.

**Metrics**: Jenkins, GitLab, and Nexus expose Prometheus endpoints. Alloy's `annotationAutodiscovery` scrapes any pod annotated with `prometheus.io/scrape: "true"`. XWiki does not expose Prometheus metrics (future work).

**Grafana dashboards** are pre-loaded in `observability/grafana.values.yaml`:
- Jenkins: [Performance and Health Overview](https://grafana.com/grafana/dashboards/9964) (gnetId 9964)
- GitLab: [GitLab Overview](https://grafana.com/grafana/dashboards/12341) (gnetId 12341)
- Nexus: [Nexus Repository Metrics](https://grafana.com/grafana/dashboards/16457) (gnetId 16457)

## Resource Requirements

| Tool | Dev RAM (approx) | Prod RAM (approx) |
|------|-----------------|------------------|
| Jenkins | ~1–2 GB | ~2–4 GB |
| GitLab | ~6–8 GB | ~8–12 GB |
| Nexus | ~1–2 GB | ~2–4 GB |
| XWiki | ~1–2 GB | ~2–4 GB |
| **Total** | **~9–14 GB** | **~14–24 GB** |

**Tip for dev**: if your machine has limited RAM, start with Jenkins and Nexus only for drain testing. GitLab can be deployed separately once the other tests are complete.

## Adding a New Tool

Follow the same pattern:
1. Create `on-premises/<tool>/` with base, dev, and prod values files.
2. Add an ArgoCD Application to `argocd/apps/base/on-premises.yaml` at the appropriate wave.
3. Add dev/prod patches in `argocd/apps/dev/kustomization.yaml` and `argocd/apps/prod/kustomization.yaml`.
4. Add PV entries to `storage/pvs.yaml` (dev) if the tool needs persistent storage.
5. Write a `on-premises/<tool>/README.md` documenting the decisions.
