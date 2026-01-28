# Troubleshooting Guide

## Common Issues

### Pods Not Starting

```bash
# Check pod status
kubectl describe pod <pod-name> -n <namespace>

# Common causes:
# - Insufficient resources → scale up nodes
# - PVC not bound → check storage class
# - Image pull error → check registry access
# - ConfigMap/Secret missing → check dependencies
```

### Storage Issues

```bash
# Check PVCs
kubectl get pvc -A

# Check storage class
kubectl get sc

# Check MinIO health
kubectl exec -n minio minio-0 -- curl -s http://localhost:9000/minio/health/live
```

### Network Issues

```bash
# Check Cilium
cilium status

# Test connectivity
cilium connectivity test

# Check Gateway
kubectl get gateway -A
kubectl get httproute -A
```

### GitLab Issues

```bash
# Check all GitLab pods
kubectl get pods -n gitlab

# Check migrations
kubectl get jobs -n gitlab

# Rails console
kubectl exec -n gitlab -it $(kubectl get pod -n gitlab -l app=toolbox -o name | head -1) -- gitlab-rails console

# Check registry
kubectl logs -n gitlab -l app=registry
```

### Database Issues

```bash
# PostgreSQL
kubectl get perconapgcluster -n databases
kubectl describe perconapgcluster myapp-pg -n databases

# Connect to PostgreSQL
kubectl exec -n databases -it myapp-pg-instance1-0 -- psql

# MongoDB
kubectl get psmdb -n databases
kubectl exec -n databases -it myapp-mongo-rs0-0 -- mongosh
```

## Reset Components

```bash
# Reset GitLab
helm uninstall gitlab -n gitlab
kubectl delete pvc --all -n gitlab
helm upgrade --install gitlab gitlab/gitlab -n gitlab -f values.yaml

# Reset MinIO
helm uninstall minio -n minio
kubectl delete pvc --all -n minio
./platform.sh deploy minio
```

## Logs Collection

```bash
# Collect all logs
mkdir -p /tmp/platform-logs
for ns in minio vault gitlab argocd monitoring; do
  kubectl logs -n $ns --all-containers --tail=1000 > /tmp/platform-logs/${ns}.log 2>&1
done
tar -czvf platform-logs.tar.gz /tmp/platform-logs/
```