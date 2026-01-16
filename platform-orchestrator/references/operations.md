# Platform Operations

Run all commands from **bastion server** or via VPN.

## Daily Operations

### Check Status

```bash
# Overall status
./platform.sh status

# Detailed pod status
kubectl get pods -A | grep -v Running

# Check events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

### View Logs

```bash
# GitLab webservice
kubectl logs -n gitlab -l app=webservice -c webservice --tail=100

# ArgoCD
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=100

# MinIO
kubectl logs -n minio -l app=minio --tail=100
```

### Access Services

Services are accessible via VPN:

```bash
# Connect to VPN first
tailscale up --login-server https://vpn.example.com --authkey <KEY>

# Then access directly
curl https://gitlab.example.com
curl https://argocd.example.com
curl https://grafana.example.com
```

Or via kubectl port-forward from bastion:

```bash
# Grafana
kubectl port-forward svc/grafana -n monitoring 3000:80 --address 0.0.0.0

# ArgoCD
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address 0.0.0.0

# MinIO Console
kubectl port-forward svc/minio-console -n minio 9001:9001 --address 0.0.0.0

# Vault
kubectl port-forward svc/vault -n vault 8200:8200 --address 0.0.0.0
```

## Backup Operations

### Manual Backup

```bash
# GitLab backup
kubectl exec -n gitlab -it $(kubectl get pod -n gitlab -l app=toolbox -o name | head -1) -- backup-utility

# PostgreSQL backup
kubectl annotate perconapgcluster myapp-pg -n databases \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date +%s)" --overwrite
```

### List Backups

```bash
# GitLab
kubectl exec -n gitlab -it $(kubectl get pod -n gitlab -l app=toolbox -o name | head -1) -- backup-utility --list

# PostgreSQL
kubectl exec -n databases myapp-pg-instance1-0 -- pgbackrest info
```

## Scaling

### Scale GitLab

```bash
kubectl scale deployment -n gitlab gitlab-webservice-default --replicas=4
kubectl scale deployment -n gitlab gitlab-sidekiq-all-in-1-v2 --replicas=4
```

### Scale Databases

```bash
kubectl patch perconapgcluster myapp-pg -n databases --type=merge \
  -p '{"spec":{"instances":[{"name":"instance1","replicas":5}]}}'
```

## Troubleshooting

### Common Issues

```bash
# Pods stuck in Pending
kubectl describe pod <pod-name> -n <namespace>

# Check PVC issues
kubectl get pvc -A | grep -v Bound

# Check node resources
kubectl top nodes
kubectl describe nodes | grep -A5 "Allocated resources"

# Check Cilium
cilium status
cilium connectivity test
```

### Recovery

```bash
# Auto-heal
./platform.sh heal

# Restart deployment
kubectl rollout restart deployment -n gitlab gitlab-webservice-default

# Force delete stuck pod
kubectl delete pod <pod> -n <ns> --force --grace-period=0
```