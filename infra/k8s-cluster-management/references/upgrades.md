# Cluster Upgrades

All upgrades run from bastion server.

## Kubernetes Upgrade (via Kubespray)

```bash
# 1. Update Kubespray
cd kubespray
git fetch --tags
git checkout v2.29.1  # Target version

# 2. Update Python dependencies
source venv/bin/activate
pip install -r requirements.txt

# 3. Update kube_version in group_vars (optional - defaults to v1.34.3)
vim inventory/mycluster/group_vars/k8s_cluster/k8s-cluster.yml
# Change: kube_version: v1.34.3

# 4. Run upgrade playbook
ansible-playbook -i inventory/mycluster/hosts.yaml upgrade-cluster.yml \
  --become --become-user=root

# 5. Verify
kubectl get nodes
kubectl version
```

## Cilium Upgrade

```bash
# Check current version
cilium version

# Upgrade via Helm
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --version 1.18.6 \
  --reuse-values

# Or via CLI
cilium upgrade --version 1.18.6

# Verify
cilium status --wait
cilium connectivity test
```

## cert-manager Upgrade

```bash
# Upgrade via Helm
helm upgrade cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.19.2 \
  --set crds.enabled=true

# Verify certificates still work
kubectl get certificates -A
```

## Gateway API Upgrade

```bash
# Check current version
kubectl get crd gateways.gateway.networking.k8s.io -o jsonpath='{.metadata.labels.gateway\.networking\.k8s\.io/bundle-version}'

# Upgrade to latest GA
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Verify
kubectl get gatewayclass
```

## Hetzner CCM Upgrade

```bash
# Upgrade to latest
kubectl apply -f https://github.com/hetznercloud/hcloud-cloud-controller-manager/releases/latest/download/ccm-networks.yaml

# Wait for rollout
kubectl -n kube-system rollout status deployment/hcloud-cloud-controller-manager

# Verify
kubectl get pods -n kube-system -l app=hcloud-cloud-controller-manager
```

## Pre-Upgrade Checklist

- [ ] Backup etcd: `etcdctl snapshot save backup.db`
- [ ] Review changelog for breaking changes
- [ ] Test upgrade on staging/minimal tier first
- [ ] Ensure PodDisruptionBudgets are set
- [ ] Schedule maintenance window
- [ ] Verify all nodes healthy: `kubectl get nodes`
- [ ] Check cluster resources: `kubectl top nodes`

## Rollback (Kubespray)

```bash
# If upgrade fails, restore from etcd backup
# On master node:
etcdctl snapshot restore backup.db

# Or re-run Kubespray with previous version
git checkout v2.28.0
ansible-playbook -i inventory/mycluster/hosts.yaml cluster.yml
```