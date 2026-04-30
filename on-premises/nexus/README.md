# Nexus Repository Manager OSS

Nexus OSS is the artifact repository that serves Maven, npm, Docker, PyPI, and other package formats to internal teams. It is deployed via the official [Sonatype Helm chart](https://github.com/sonatype/nxrm3-helm-repository).

## Chart Choice

The official `sonatype/nexus-repository-manager` chart from `sonatype.github.io/helm3-charts` is the only chart maintained by the Nexus vendor. Third-party alternatives exist but are not kept up-to-date with Nexus releases. Using the official chart ensures compatibility and access to security patches.

## Dev: Embedded H2 Database

In dev, Nexus runs with its embedded H2 database. This is acceptable for drain/migration testing and learning. H2 is a file-based database stored inside the Nexus data directory on the Longhorn PVC, so it moves with the pod when a node is drained.

**Warning**: H2 is not safe for production under concurrent write load — it can corrupt data. Do not use it for the real production deployment.

## Prod: External PostgreSQL

Sonatype's official recommendation for containerized production deployments is to use PostgreSQL. The `nexus.values.prod.yaml` file contains a commented-out example of the JVM parameters needed to connect Nexus to an external PostgreSQL instance.

Steps for prod setup:
1. Deploy a PostgreSQL instance (Bitnami chart or external server).
2. Create a `nexus` database and user.
3. Uncomment and fill in the `extraEnvVars` block in `nexus.values.prod.yaml`.

## Repository Formats Available

The following repository formats are available out-of-the-box and can be configured via the Nexus UI or provisioning scripts:
- Maven 2 (hosted, proxy, group)
- npm (hosted, proxy, group)
- Docker (hosted, proxy, group) — requires additional port/ingress configuration
- PyPI (hosted, proxy, group)
- Raw (hosted, proxy, group)

Repository creation is not automated in this Helm setup — configure repositories via the Nexus UI after first startup.

## What Is Configured

- **Persistence**: blob store on a Longhorn PVC (`/nexus-data`). Moves with the pod on drain.
- **Metrics**: pod annotations expose `/service/metrics/prometheus` for Alloy scraping. Grafana dashboard [Nexus Repository Metrics](https://grafana.com/grafana/dashboards/16457) (gnetId 16457) is pre-loaded.
- **Ingress**: Traefik handles access via `nexus.localhost` (dev) or `nexus.fhassis.top` (prod).

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| Docker registry port | Not configured | Docker pull/push requires a separate port (e.g., 5000). Add a second ingress rule or NodePort when needed. |
| LDAP | Not configured | Future work; use the Nexus UI Security → LDAP section |
| Repository cleanup policies | Not configured | Manual via UI; schedule cleanup tasks after repositories are created |
| S3 blob store | Not configured | Using Longhorn PVC; S3 is an alternative for large artifact volumes |
| SSL termination (internal) | Not configured | TLS terminated at Traefik |

## Accessing Nexus

- **Dev**: http://nexus.localhost:8080
- **Prod**: https://nexus.fhassis.top
- **Default credentials**: `admin` / retrieve initial password from `/nexus-data/admin.password` inside the pod:
  ```bash
  kubectl exec -n nexus <nexus-pod> -- cat /nexus-data/admin.password
  ```

## Production Hardening

- Switch to external PostgreSQL (uncomment and configure `extraEnvVars` in prod values).
- Increase `persistence.size` based on artifact volume.
- Configure LDAP for user management.
- Add repository cleanup scheduled tasks to manage disk usage.
- Set up Docker registry port and ingress when Docker hosting is needed.
