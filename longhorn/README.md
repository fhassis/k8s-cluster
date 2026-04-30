# Longhorn Storage

Longhorn is a distributed block storage system for Kubernetes. In this setup, its primary purpose is **pod mobility**, not data redundancy: when a node is drained for OS maintenance, Longhorn detaches a volume from that node and reattaches it on whichever node the pod is rescheduled to. This allows rolling maintenance with near-zero downtime.

## Why Longhorn (not NFS, not local-path)

| Option | Problem |
|--------|---------|
| NFS (`/nfs/gbh_data`) | GitLab, Nexus, and PostgreSQL do not work reliably over NFS (file locking issues, performance). |
| `local-path` (k3s default) | PVs are pinned to the node where they were created via `nodeAffinity`. After a drain, the pod cannot schedule elsewhere because its volume is stuck. |
| Longhorn | Block storage with detach/reattach. No node pinning. Pod moves freely between nodes. |

## Why `replicaCount: 1`

Each volume has a single data replica. Longhorn still handles the detach-from-drained-node → reattach-on-new-node lifecycle. A single replica means no redundancy (if the disk fails, data is lost), but that is the same situation we had with docker-compose. Replication can be enabled later with `defaultReplicaCount: 3`.

## Dev Environment — Shared-hostPath Simulation

Longhorn requires `open-iscsi` and kernel modules (`iscsi_tcp`) that are not available inside k3d Docker containers. **Longhorn does not run in k3d.**

For dev (k3d), we simulate the same pod-mobility behavior using a shared host directory:

1. The k3d cluster mounts `~/k3d-storage` from the WSL host into **every node container** at `/data/longhorn`.
2. Because all containers share the same WSL filesystem, any node can read the same files.
3. Static PVs in `storage/pvs.yaml` use `hostPath: /data/longhorn/{app}` with **no `nodeAffinity`**.
4. The StorageClass is named `longhorn` in both dev and prod — Helm values files are identical.

When a node is drained in dev:
- The pod is evicted and rescheduled on another node.
- The new node finds the data at `/data/longhorn/{app}` (same physical directory on WSL).
- The pod starts up with all its previous data intact — exactly what Longhorn does in prod.

## Prod Prerequisites (SLES + k3s)

Run these commands on **every k3s node** before deploying Longhorn:

```bash
# 1. Install open-iscsi
zypper install -y open-iscsi

# 2. SLES ships kernel-default-base by default, which does NOT include iscsi_tcp.
#    Replace it with the full kernel package:
zypper install -y kernel-default
zypper remove -y kernel-default-base

# 3. Enable the iSCSI daemon
systemctl enable --now iscsid

# 4. Create the Longhorn data directory on the dedicated partition
mkdir -p /data/longhorn

# 5. (Optional) Verify Longhorn environment readiness after deploying
kubectl apply -f https://raw.githubusercontent.com/longhorn/longhorn/master/scripts/environment_check.sh
```

## Deploying Longhorn (prod via ArgoCD)

Longhorn is deployed by ArgoCD from `argocd/apps/prod/longhorn.yaml`. It runs at sync wave `-1`, so it is always installed before any on-premises application that needs a PVC.

To deploy manually (outside ArgoCD):

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update

helm upgrade --install longhorn longhorn/longhorn \
  --namespace longhorn-system \
  --create-namespace \
  -f longhorn/longhorn.values.yaml \
  -f longhorn/longhorn.values.prod.yaml
```

## Accessing the Longhorn UI

- **Prod (via ingress):** `https://longhorn.fhassis.top`
- **Port-forward:** `kubectl port-forward -n longhorn-system svc/longhorn-frontend 8000:80`

From the UI you can see volume health, replicas, and snapshot schedules.

## Volume Snapshots (Basic Backup)

Longhorn supports recurring snapshot schedules per volume. This provides a simple backup strategy without external object storage:

```bash
# Create a snapshot manually
kubectl -n longhorn-system exec -it <longhorn-manager-pod> -- \
  longhorn-manager snapshot create --volume jenkins-pvc --name manual-snap
```

Recurring schedules can be configured from the Longhorn UI under **Recurring Jobs**.
