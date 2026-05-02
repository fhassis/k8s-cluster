# Multipass Staging Cluster

Three-node k3s cluster on Ubuntu VMs for staging — drain tests, Longhorn, and ArgoCD GitOps.

| Node | Role | Cloud-init |
|------|------|-----------|
| node1 | k3s server | `cloud-init-server.yaml` |
| node2 | k3s agent | `cloud-init-agent.yaml` |
| node3 | k3s agent | `cloud-init-agent.yaml` |

## Create the cluster

### 0. Disable Traefik in cloud-init

k3s ships with Traefik as its default ingress. This cluster uses `ingress-nginx` instead. Before launching node1, edit `cloud-init-server.yaml` to pass `--disable=traefik` to the k3s server:

```yaml
# In cloud-init-server.yaml, under the k3s install command, add:
--disable=traefik
```

Or append it to the `K3S_EXEC` / `INSTALL_K3S_EXEC` environment variable in the cloud-init file if that pattern is used. Disabling Traefik at install time is cleaner than removing it after the fact.

If you are **migrating an existing cluster** that already has Traefik running:

```bash
# On node1 (server): add disable flag to k3s config, then restart
multipass exec node1 -- sudo bash -c 'echo "disable: [traefik]" >> /etc/rancher/k3s/config.yaml'
multipass exec node1 -- sudo systemctl restart k3s

# Clean up Traefik HelmChart objects left behind
kubectl -n kube-system delete helmchart traefik traefik-crd --ignore-not-found
```

### 1. Launch node1 (server)

```bash
multipass launch --name node1 --cpus 4 --memory 8G --disk 30G \
  --cloud-init clusters/multipass/cloud-init-server.yaml 26.04
```

Wait for cloud-init to finish (~2 min):

```bash
multipass exec node1 -- cloud-init status --wait
```

### 2. Get node1's IP and cluster token

```bash
NODE1_IP=$(multipass exec node1 -- hostname -I | awk '{print $1}')
TOKEN=$(multipass exec node1 -- sudo cat /var/lib/rancher/k3s/server/node-token)
echo "IP: $NODE1_IP  TOKEN: $TOKEN"
```

### 3. Prepare agent cloud-init

Edit `cloud-init-agent.yaml` replacing the three placeholders:

```bash
sed -e "s/NODE_NAME/node2/" \
    -e "s/SERVER_IP/$NODE1_IP/" \
    -e "s/CLUSTER_TOKEN/$TOKEN/" \
    clusters/multipass/cloud-init-agent.yaml > /tmp/agent-node2.yaml

sed -e "s/NODE_NAME/node3/" \
    -e "s/SERVER_IP/$NODE1_IP/" \
    -e "s/CLUSTER_TOKEN/$TOKEN/" \
    clusters/multipass/cloud-init-agent.yaml > /tmp/agent-node3.yaml
```

### 4. Launch node2 and node3

```bash
multipass launch --name node2 --cpus 2 --memory 4G --disk 30G \
  --cloud-init /tmp/agent-node2.yaml 26.04

multipass launch --name node3 --cpus 2 --memory 4G --disk 30G \
  --cloud-init /tmp/agent-node3.yaml 26.04
```

### 5. Get kubeconfig

```bash
multipass exec node1 -- sudo cat /etc/rancher/k3s/k3s.yaml \
  | sed "s/127.0.0.1/$NODE1_IP/" > ~/.kube/staging-config

export KUBECONFIG=~/.kube/staging-config
kubectl get nodes
# Expected: node1 (control-plane), node2, node3 — all Ready
```

## TLS pre-flight (run once before bootstrap)

Generate a self-signed wildcard cert for `*.staging` and load it as a Secret:

```bash
openssl req -x509 -nodes -days 3650 -newkey rsa:4096 \
  -keyout wildcard.key -out wildcard.crt \
  -subj "/CN=*.staging" \
  -addext "subjectAltName=DNS:*.staging,DNS:staging"

kubectl create secret tls ssl-certificate \
  --cert=wildcard.crt --key=wildcard.key \
  -n kube-system
```

## DNS (Windows hosts file)

Add to `C:\Windows\System32\drivers\etc\hosts` (replace with actual node1 IP):

```
192.168.x.x   argocd.staging grafana.staging prometheus.staging loki.staging
192.168.x.x   jenkins.staging nexus.staging xwiki.staging gitlab.staging
192.168.x.x   sonarqube.staging drawio.staging excalidraw.staging umami.staging
```

## Install ArgoCD and bootstrap GitOps

```bash
helm repo add argo https://argoproj.github.io/argo-helm && helm repo update
kubectl create namespace argocd
helm install argocd argo/argo-cd -n argocd -f argocd/argocd.values.yaml
kubectl -n argocd rollout status deployment/argocd-server

kubectl apply -f argocd/bootstrap.staging.yaml
kubectl -n argocd get applications -w
```

## Destroy the cluster

```bash
multipass delete node1 node2 node3 && multipass purge
```
