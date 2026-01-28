# MinIO Operations

## Health Check

```bash
# Check cluster health
kubectl exec -n minio minio-0 -- curl -s http://localhost:9000/minio/health/cluster

# Check individual node
kubectl exec -n minio minio-0 -- curl -s http://localhost:9000/minio/health/live

# Using mc
mc admin info myminio
```

## Scaling

```bash
# Scale replicas (must maintain minimum 4 for distributed)
kubectl scale statefulset minio -n minio --replicas=6

# Add storage (expand PVC - if storage class supports)
kubectl patch pvc export-minio-0 -n minio -p '{"spec":{"resources":{"requests":{"storage":"200Gi"}}}}'
```

## Backup & Restore

```bash
# Mirror bucket to another MinIO
mc mirror myminio/gitlab-registry myminio-backup/gitlab-registry

# Export bucket contents
mc cp --recursive myminio/gitlab-backups /local/backup/
```

## Cleanup

```bash
# Remove old versions
mc rm --recursive --versions --older-than 30d myminio/gitlab-backups

# List bucket usage
mc du myminio/

# Remove empty buckets
mc rb myminio/unused-bucket
```

## Disaster Recovery

```bash
#!/bin/bash
# scripts/minio-disaster-recovery.sh

# 1. Stop applications writing to MinIO
kubectl scale deployment -n gitlab --all --replicas=0

# 2. Create snapshot of all PVCs
for i in 0 1 2 3; do
  kubectl exec -n minio minio-$i -- mc admin service freeze myminio
done

# 3. Backup PVCs (using your backup tool)
# velero backup create minio-backup --include-namespaces minio

# 4. Unfreeze
for i in 0 1 2 3; do
  kubectl exec -n minio minio-$i -- mc admin service unfreeze myminio
done

# 5. Restart applications
kubectl scale deployment -n gitlab --all --replicas=2
```

## Complete Uninstall

```bash
#!/bin/bash
# scripts/cleanup-minio.sh

echo "WARNING: This will delete ALL MinIO data!"
read -p "Type 'DELETE' to confirm: " confirm

if [[ "$confirm" != "DELETE" ]]; then
  echo "Aborted."
  exit 1
fi

helm uninstall minio -n minio
kubectl delete pvc --all -n minio
kubectl delete namespace minio

echo "MinIO removed!"
```