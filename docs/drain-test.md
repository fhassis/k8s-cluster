# Drain Test — Pod Migration Without Data Loss

This test demonstrates the core goal of the Longhorn + on-premises setup: draining a Kubernetes node for OS maintenance while an application moves to another node and resumes with all its data intact.

The test uses Jenkins, but the exact same steps apply to GitLab, Nexus, and XWiki — just change the namespace and the app-specific verification step.

## Prerequisites

1. k3d dev cluster is running: `k3d cluster list` should show `dev` with 4 nodes.
2. ArgoCD is synced: all on-premises apps are `Healthy` and `Synced`.
3. Jenkins is accessible at http://jenkins.localhost:8080.

## Step 1 — Identify Which Node Jenkins Is Running On

```bash
kubectl get pod -n jenkins -o wide
```

Example output:
```
NAME                       READY   STATUS    NODE
jenkins-6b9d7d4f5c-x8k2p   2/2     Running   k3d-dev-agent-0
```

Note the node name. We will drain **that** node.

## Step 2 — Create Test Data

Before draining, create something that we can verify is still there after the migration.

1. Open http://jenkins.localhost:8080 in your browser.
2. Create a new Freestyle job named `drain-test`.
3. Add a build step: **Execute shell** → `echo "drain test: $(date)"`.
4. Click **Save**, then **Build Now**.
5. Wait for the build to complete and note the build number (e.g., `#1`).

## Step 3 — Drain the Node

Open a terminal and watch the pods in real time:

```bash
kubectl get pod -n jenkins -w
```

In a second terminal, drain the node that Jenkins is running on:

```bash
# Replace k3d-dev-agent-0 with the node name from Step 1
kubectl drain k3d-dev-agent-0 \
  --ignore-daemonsets \
  --delete-emptydir-data
```

Expected output from `kubectl drain`:
```
node/k3d-dev-agent-0 cordoned
evicting pod jenkins/jenkins-6b9d7d4f5c-x8k2p
pod/jenkins-6b9d7d4f5c-x8k2p evicted
node/k3d-dev-agent-0 drained
```

## Step 4 — Watch the Pod Reschedule

In the first terminal (watching pods), you should see:

```
NAME                       READY   STATUS        NODE
jenkins-6b9d7d4f5c-x8k2p   2/2     Terminating   k3d-dev-agent-0
jenkins-6b9d7d4f5c-9m3nt   0/2     Pending       <none>
jenkins-6b9d7d4f5c-9m3nt   0/2     ContainerCreating   k3d-dev-agent-1
jenkins-6b9d7d4f5c-9m3nt   2/2     Running       k3d-dev-agent-1
```

The pod has moved to `k3d-dev-agent-1`. Confirm the new node:

```bash
kubectl get pod -n jenkins -o wide
```

## Step 5 — Verify Data Persistence

Wait for Jenkins to become fully ready (the `2/2` Ready count):

```bash
kubectl wait pod -n jenkins -l app.kubernetes.io/component=jenkins-controller \
  --for=condition=Ready --timeout=120s
```

Open http://jenkins.localhost:8080 again. Verify:
- The `drain-test` job still exists.
- Build `#1` is still in the build history with its console output.

This confirms that the Jenkins home directory (jobs, build history, configuration) survived the node drain and pod migration intact.

## Step 6 — Restore the Node

Uncordon the drained node to make it available for scheduling again:

```bash
kubectl uncordon k3d-dev-agent-0
```

Verify all nodes are schedulable:

```bash
kubectl get nodes
```

All nodes should show `Ready` with no `SchedulingDisabled` taint.

---

## How This Works

In **dev (k3d)**:
- All k3d node containers mount `~/k3d-storage → /data/longhorn` from the WSL host.
- The Jenkins PVC is a static PV pointing to `/data/longhorn/jenkins` with no `nodeAffinity`.
- Any node can mount the same physical directory — the data is always accessible.
- When the pod moves to `k3d-dev-agent-1`, it finds Jenkins home exactly where it left it.

In **prod (k3s + Longhorn)**:
- Longhorn detects that the draining node holds a volume replica.
- It detaches the replica from the draining node.
- Once the pod is scheduled on the new node, Longhorn reattaches the volume there.
- The process is the same from the operator's perspective — drain, watch, uncordon.

---

## Repeating the Test for Other Apps

| App | Namespace | Verification |
|-----|-----------|--------------|
| GitLab | `gitlab` | Git repository and issues still accessible after pod moves |
| Nexus | `nexus` | Browse repositories; artifacts still present |
| XWiki | `xwiki` | Wiki pages and attachments still accessible |

Replace `jenkins` with the target namespace and adjust the verification step accordingly.
