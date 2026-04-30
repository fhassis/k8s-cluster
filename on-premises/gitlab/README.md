# GitLab

GitLab provides Git hosting, issue tracking, a container registry, and Pages for internal documentation. It is deployed via the official [charts.gitlab.io](https://charts.gitlab.io) Helm chart from the GitLab project.

## Chart Choice

The official GitLab chart is the only production-grade, actively maintained option. Third-party community charts exist but lag behind in features and security updates. The chart is large and opinionated, but gives full control over every component.

## Edition: Community Edition (CE)

`global.edition: ce` selects the open-source Community Edition, which is sufficient for internal teams. The Enterprise Edition (EE) can be used with a free license but adds unnecessary complexity.

## Dev / Prod Parity Decision

Unlike other tools where dev uses a trimmed-down config, GitLab uses the **same component set in both environments**. The 3 GB RAM difference between a minimal dev config (~5 GB) and the full CE config (~8 GB) is acceptable on the WSL dev machine. This choice means:
- Every feature available in prod is also testable in dev.
- No surprises when deploying to production.
- Only resource limits and ingress differ between environments.

## Component Decisions

| Component | Status | Reason |
|-----------|--------|--------|
| webservice + workhorse | ✅ Enabled | Core web UI and API |
| Gitaly | ✅ Enabled | Git repository storage |
| PostgreSQL (bundled) | ✅ Enabled | Acceptable for single-node. See prod note. |
| Redis (bundled) | ✅ Enabled | Same reasoning |
| Container Registry | ✅ Enabled | Teams can push/pull Docker images to the internal registry |
| GitLab Pages | ✅ Enabled | Internal documentation hosting |
| gitlab-runner | ❌ Disabled | CI runners are a separate concern; deploy when needed |
| gitlab-kas (agent server) | ❌ Disabled | Kubernetes agent — not needed in current setup |
| MinIO | ❌ Disabled | Object storage; not needed when Gitaly handles repos |
| nginx-ingress (bundled) | ❌ Disabled | Using cluster-wide Traefik |

## What Is Configured

- **Persistence**: Gitaly, PostgreSQL, and Redis all use Longhorn PVCs for pod mobility.
- **Metrics**: `global.metrics.enabled: true` exposes Prometheus endpoints on webservice, Gitaly, and Workhorse. Alloy's `annotationAutodiscovery` scrapes them automatically.
- **Observability**: Grafana dashboard [GitLab Overview](https://grafana.com/grafana/dashboards/12341) (gnetId 12341) is pre-loaded.
- **Ingress**: Traefik handles all external access. TLS disabled in dev, enabled in prod.

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| CI/CD Runners | Not configured | Runners are deployed separately when needed; they scale independently |
| LDAP / SAML SSO | Not configured | Future work; default admin credentials in dev |
| Email (SMTP) | Not configured | Future work via `global.smtp` |
| Backup | Not explicit | Longhorn PVC is retained on deletion. Use GitLab's built-in backup task for application-level backups |
| Object storage (S3/MinIO) | Not configured | Gitaly handles repository storage; LFS and other object data use local storage |

## Resource Requirements

GitLab is the heaviest application in this setup:

| Environment | Approximate RAM |
|-------------|----------------|
| Dev (k3d) | ~6–8 GB across all pods |
| Prod (k3s) | ~8–12 GB recommended |

GitLab is deployed at sync wave `4` — last of all tools — to avoid putting excessive pressure on nodes while lighter apps are starting.

## Production Hardening

- Replace bundled PostgreSQL with an external PostgreSQL instance for better durability and independent scaling.
- Store `global.initialRootPassword` in a Kubernetes Secret.
- Enable TLS in `gitlab.values.prod.yaml` (already pre-configured, just update the domain and cert secret name).
- Pin the chart `targetRevision` to a specific GitLab version to control upgrade timing.
- Configure SMTP for email notifications.
