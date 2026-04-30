# XWiki

XWiki is the internal wiki and knowledge base platform. It is deployed via the community-maintained [xwiki-contrib/xwiki-helm](https://github.com/xwiki-contrib/xwiki-helm) chart, which is the most active and up-to-date Helm chart available for XWiki.

## Chart Choice

No official vendor chart exists for XWiki. The `xwiki-contrib/xwiki-helm` chart is maintained by the XWiki open-source community (same organization that publishes the XWiki Docker image) and supports both standalone and HA configurations with multiple database backends. It includes a Bitnami PostgreSQL subchart for convenience.

## What Is Configured

- **Persistence**: wiki data stored on a Longhorn PVC. Moves with the pod on drain.
- **PostgreSQL subchart**: a Bitnami PostgreSQL instance is included and also backed by a Longhorn PVC. Both the app and its database move together when a node is drained.
- **Ingress**: Traefik handles access via `xwiki.localhost` (dev) or `xwiki.fhassis.top` (prod).
- **Logs**: collected automatically by Alloy `podLogsViaLoki` from the `xwiki` namespace, visible in Grafana → Explore → Loki.

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| Prometheus metrics | Not available | XWiki has no native Prometheus endpoint. JMX is available but requires a JMX exporter sidecar — noted as future work. |
| Clustering | Not configured | XWiki clustering requires shared storage and a load balancer. Not needed for single-container mobility (our goal). |
| LDAP | Not configured | Future work; configure via XWiki Administration → LDAP. |
| Glowroot APM | Not configured | XWiki ships with Glowroot support for tracing. Can be enabled as a future observability improvement. |
| Email (SMTP) | Not configured | Future work via XWiki Administration → Email. |

## Major Version Upgrades

XWiki requires a **migration wizard** when upgrading across major versions (e.g., 14.x → 15.x). The wizard runs automatically on first startup after an image tag change, but it can take several minutes and temporarily makes the wiki unavailable. Always take a Longhorn snapshot before upgrading.

Upgrade process:
1. Take a Longhorn snapshot of both `xwiki-pv` and `xwiki-postgresql-pv`.
2. Update the `image.tag` in `xwiki.values.yaml`.
3. ArgoCD syncs → XWiki pod restarts → migration wizard runs automatically.
4. Verify the wiki is functional before deleting the snapshot.

## Accessing XWiki

- **Dev**: http://xwiki.localhost:8080
- **Prod**: https://xwiki.fhassis.top
- **First run**: XWiki presents an installation wizard on first startup. Complete it to create the admin account and initialize the database.

## Production Hardening

- Store the PostgreSQL password in a Kubernetes Secret (not in values files).
- Increase `persistence.size` and `postgresql.primary.persistence.size` based on content volume.
- Pin `image.tag` to a specific XWiki LTS version to control upgrade timing.
- Consider a dedicated external PostgreSQL if content volume grows significantly.
- Enable Glowroot for application performance monitoring.
