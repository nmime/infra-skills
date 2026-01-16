# Scaling Between Tiers

## Upgrade Path

```
Minimal (€18) → Small (€30) → Medium (€50) → Production (€75)
```

## Minimal → Small

```bash
# 1. Add dedicated master
hcloud server create --name master-1 --type cx22 --network k8s-private --without-ipv4

# 2. Migrate control plane
kubeadm join ... --control-plane

# 3. Add load balancer
hcloud load-balancer create --name k8s-lb --type lb11

# 4. Add second worker
hcloud server create --name worker-2 --type cx32 --network k8s-private --without-ipv4

# 5. Update config
cp profiles/small.yaml platform.yaml
./platform.sh reconcile
```

## Small → Medium

```bash
# 1. Add 2 more masters for HA
hcloud server create --name master-2 --type cx22 --network k8s-private --without-ipv4
hcloud server create --name master-3 --type cx22 --network k8s-private --without-ipv4

# 2. Join as control plane
kubeadm join ... --control-plane

# 3. Scale MinIO to distributed
kubectl scale statefulset minio --replicas=4 -n minio

# 4. Enable Vault HA
helm upgrade vault hashicorp/vault --set server.ha.enabled=true --set server.ha.replicas=3

# 5. Update config
cp profiles/medium.yaml platform.yaml
./platform.sh reconcile
```

## Medium → Production

```bash
# 1. Upgrade master nodes
for i in 1 2 3; do
  hcloud server change-type master-$i cx32
done

# 2. Add third worker
hcloud server create --name worker-3 --type cx32 --network k8s-private --without-ipv4

# 3. Scale services
kubectl scale deployment gitlab-webservice --replicas=2 -n gitlab
kubectl scale deployment argocd-server --replicas=2 -n argocd

# 4. Update config
cp profiles/production.yaml platform.yaml
./platform.sh reconcile
```

## Downgrade (Cost Saving)

```bash
# Production → Medium: Remove workers, downgrade nodes
# Medium → Small: Remove HA masters
# Small → Minimal: Remove LB, combine master+worker

# Always backup first!
./platform.sh backup create
```