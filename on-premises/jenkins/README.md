# Jenkins

Jenkins is the CI/CD automation server used by internal teams. It is deployed via the official [jenkinsci/helm-charts](https://github.com/jenkinsci/helm-charts) chart, maintained by the Jenkins project, and runs as a single controller pod backed by a Longhorn PVC.

## Chart Choice

The official chart (`charts.jenkins.io/jenkins`) is the only maintained, production-grade option for Jenkins on Kubernetes. It supports JCasC (Jenkins Configuration as Code) natively, making the entire controller configuration version-controlled and reproducible without any manual UI steps.

## What Is Configured

### JCasC — Configuration as Code

All Jenkins configuration is declared in `jenkins.values.yaml` under `controller.JCasC.configScripts`. On first startup, the controller reads these scripts and configures itself automatically:

- **Agent protocols**: `JNLP4-connect` and `Ping` enabled. Agents connect over WebSocket (`webSocket: true`), which goes through port 80/443 — no extra NodePort or service needed.
- **WSL permanent agent**: a permanent agent named `wsl-agent` is pre-declared. It connects from the WSL host to the Jenkins controller via WebSocket. Retrieve the generated secret from: **Manage Jenkins → Nodes → wsl-agent → Status**.

### Plugins

| Plugin | Why |
|--------|-----|
| `workflow-aggregator` | Pipeline (Declarative and Scripted) support |
| `workflow-durable-task-step` | `sh`/`bat` steps keep running on the agent even if the controller restarts — critical for the drain survival test |
| `git` | Git SCM integration |
| `configuration-as-code` | Reads JCasC scripts from `configScripts` at startup |
| `prometheus` | Exposes `/prometheus` endpoint for Grafana scraping |

### Persistence

Jenkins home (`/var/jenkins_home`) is stored on a Longhorn PVC. This means:
- In **dev**, the PVC is backed by the shared WSL host directory — any k3d node can mount it after a drain.
- In **prod**, Longhorn detaches and reattaches the block volume when the controller pod moves to a new node.

### Observability

- **Metrics**: the `prometheus` plugin exposes `/prometheus` on the controller pod. Pod annotations (`prometheus.io/scrape: "true"`) let Alloy's `annotationAutodiscovery` scrape it automatically.
- **Logs**: collected by Alloy `podLogsViaLoki` from the `jenkins` namespace, visible in Grafana → Explore → Loki.
- **Dashboard**: Grafana dashboard [Jenkins Performance and Health Overview](https://grafana.com/grafana/dashboards/9964) (gnetId 9964) is pre-loaded via `grafana.values.yaml`.

## What Is NOT Configured (and Why)

| Feature | Status | Reason |
|---------|--------|--------|
| Kubernetes dynamic agents | Disabled | Deliberately skipped to keep the drain test simple. Re-enable via `agent.enabled: true` when needed. |
| LDAP / SSO | Not configured | Future work. Default admin credentials in dev. Use a Kubernetes Secret in prod. |
| HTTPS termination (internal) | Not configured | TLS is terminated at Traefik. Jenkins runs HTTP inside the cluster. |
| Backup | Not explicit | Longhorn PVC is retained on deletion (`reclaimPolicy: Retain`). Longhorn snapshot schedules can be added via the Longhorn UI. |
| Email notifications | Not configured | Future work via JCasC `mailer` section. |

## WSL Agent — How to Connect

1. Install Java 17+ on WSL.
2. Get the agent secret from Jenkins UI: **Manage Jenkins → Nodes → wsl-agent → Status**.
3. Download `agent.jar`:
   ```bash
   curl -sO http://jenkins.localhost:8080/jnlpJars/agent.jar
   ```
4. Run the agent (reconnect loop):
   ```bash
   while true; do
     java -jar agent.jar \
       -url http://jenkins.localhost:8080 \
       -secret <secret-from-step-2> \
       -name wsl-agent \
       -webSocket \
       -workDir /tmp/jenkins-agent
     echo "Agent disconnected, retrying in 10s..."
     sleep 10
   done
   ```

The reconnect loop ensures the agent recovers automatically if the controller restarts during a drain.

## Production Hardening

- Store the admin password in a Kubernetes Secret and reference it via `controller.adminSecret`.
- Pin `targetRevision` in `argocd/apps/base/on-premises.yaml` to a specific chart semver.
- Adjust `controller.persistence.size` based on build history volume.
- Consider enabling the Kubernetes plugin for ephemeral build agents when the cluster has spare capacity.
