# Observability Stack

Single-replica, non-HA Grafana stack deployed as five Helm releases into the `observability` namespace. Collection is owned by `grafana/k8s-monitoring`; each backend (Prometheus, Loki, Tempo, Grafana) is its own independent chart.

Target environments:

- `k3d-dev` (local single node)
- Single-node VPS running k3s
- 5-node k3s cluster

All rely on the `local-path` StorageClass (default in k3s/k3d) and single-replica filesystem storage.

## Components

| Release | Chart | Role |
|---|---|---|
| `prometheus` | `prometheus-community/prometheus` | Metrics storage — plain server, remote-write receiver enabled |
| `loki` | `grafana-community/loki` (SingleBinary) | Log storage + query |
| `tempo` | `grafana-community/tempo` (single binary, **deprecated** — see note in [tempo.values.yaml](tempo.values.yaml)) | Trace storage + query, OTLP receiver on 4317/4318 |
| `grafana` | `grafana-community/grafana` | UI — datasources pre-wired for Prometheus, Loki, Tempo |
| `k8s-monitoring` | `grafana/k8s-monitoring` | Unified collection: Alloy DaemonSet + singleton, node-exporter, kube-state-metrics, OTLP receiver for apps |

`k8s-monitoring` installs the Alloy Operator as a dependency and deploys node-exporter + kube-state-metrics via its own subcharts — no separate charts needed for those.

## Install

```bash
# 1. Add Helm repos (once per machine)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana-community https://grafana-community.github.io/helm-charts
helm repo update

# 2. Create namespace
kubectl create namespace observability

# 3. Install backends first
helm install prometheus prometheus-community/prometheus \
  -n observability -f observability/prometheus.values.yaml

helm install loki grafana-community/loki \
  -n observability -f observability/loki.values.yaml

helm install tempo grafana-community/tempo \
  -n observability -f observability/tempo.values.yaml

helm install grafana grafana-community/grafana \
  -n observability -f observability/grafana.values.yaml

# 4. Install collection last (backends must be reachable)
helm install k8s-monitoring grafana/k8s-monitoring \
  -n observability -f observability/k8s-monitoring.values.yaml

# 5. Watch pods come up
kubectl -n observability get pods -w
```

## Access Grafana

```bash
kubectl -n observability port-forward svc/grafana 3000:80
```

Open <http://localhost:3000> — log in as `admin` / `admin`.

Go to **Connections → Data sources** to confirm three healthy datasources: Prometheus, Loki, Tempo.

Quick smoke tests in **Explore**:

- Prometheus: `count(up)` should return ≥ 3 series.
- Loki: `{namespace="kube-system"}` should return logs with `namespace`, `pod`, `container` labels.
- Tempo: empty until an app sends a trace.

## Send a trace from a workload

Apps send OTLP to the Alloy singleton Service (DaemonSet handles node-local scraping, singleton handles OTLP reception):

| Protocol | Endpoint |
|---|---|
| OTLP gRPC | `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4317` |
| OTLP HTTP | `k8s-monitoring-alloy-singleton.observability.svc.cluster.local:4318` |

Set `OTEL_EXPORTER_OTLP_ENDPOINT` in your workload to one of those.

Alloy enriches every span with `k8s.cluster.name`, `k8s.namespace.name`, `k8s.pod.name`, and `k8s.node.name` via `k8sattributes` processor.

## Uninstall

```bash
helm uninstall k8s-monitoring grafana tempo loki prometheus -n observability
kubectl delete namespace observability
```

The Alloy Operator CRDs installed by k8s-monitoring remain after uninstall. Remove them for a full cleanup:

```bash
kubectl get crd -o name | grep alloy | xargs kubectl delete
```

## Grafana Cloud portability

To switch any environment to Grafana Cloud, edit only `k8s-monitoring.values.yaml`:

1. Replace the three `destinations` URLs with Grafana Cloud endpoints.
2. Add `auth` blocks with your Grafana Cloud credentials.
3. Skip installing (or uninstall) prometheus / loki / tempo / grafana — the Cloud provides those.
4. App workloads need no changes.

```yaml
# Example Grafana Cloud destinations (replace in-cluster URLs with these)
destinations:
  - name: prometheus
    type: prometheus
    url: https://prometheus-prod-XX.grafana.net/api/prom/push
    auth:
      type: basic
      username: "<cloud-prometheus-id>"
      password: "<cloud-api-key>"
  - name: loki
    type: loki
    url: https://logs-prod-XX.grafana.net/loki/api/v1/push
    auth:
      type: basic
      username: "<cloud-loki-id>"
      password: "<cloud-api-key>"
  - name: tempo
    type: otlp
    url: tempo-prod-XX.grafana.net:443
    protocol: grpc
    auth:
      type: basic
      username: "<cloud-tempo-id>"
      password: "<cloud-api-key>"
    traces:
      enabled: true
```

## Secrets

The Grafana admin password is committed as `admin` in [grafana.values.yaml](grafana.values.yaml) — a throwaway dev default.

**Switch to a real secrets tool when any of these become true:**

- This repo goes public.
- Grafana is exposed on a public IP.
- Real credentials enter the stack (Grafana Cloud API keys, OAuth, SMTP).

Migration path: **SOPS + [age](https://github.com/FiloSottile/age) + [`helm-secrets`](https://github.com/jkroepke/helm-secrets)**. Encrypt the sensitive values file, commit it, decrypt at apply time. The base `*.values.yaml` files stay committable.

## Adapting for VPS / work cluster

The base values are environment-agnostic. Likely additions per environment (out of scope for now):

- Ingress + TLS for Grafana.
- Real Grafana admin password (see **Secrets** above).
- Larger PVC sizes and longer Prometheus retention.
- `alertmanager.enabled: true` in prometheus.values.yaml once alerting rules exist.
- Object storage for Loki/Tempo when retention grows beyond a single PVC.
- Grafana Cloud migration for the VPS (see **Grafana Cloud portability** above).
