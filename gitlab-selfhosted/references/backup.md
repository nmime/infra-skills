# GitLab Backup & Restore

## Automated Backups

```yaml
# In gitlab-values.yaml
global:
  appConfig:
    backups:
      bucket: gitlab-backups
      tmpBucket: gitlab-tmp

gitlab:
  toolbox:
    backups:
      cron:
        enabled: true
        schedule: "0 2 * * *"  # Daily at 2 AM
        extraArgs: "--skip registry"  # Registry backed up separately
      objectStorage:
        config:
          secret: gitlab-s3-credentials
          key: connection
```

## Manual Backup

```bash
# Trigger manual backup
kubectl exec -it $(kubectl get pods -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}') \
  -n gitlab -- backup-utility

# List backups
kubectl exec -it $(kubectl get pods -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}') \
  -n gitlab -- backup-utility --list
```

## Restore from Backup

```bash
# Scale down
kubectl scale deploy -n gitlab -l app=sidekiq --replicas=0
kubectl scale deploy -n gitlab -l app=webservice --replicas=0

# Restore
kubectl exec -it $(kubectl get pods -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}') \
  -n gitlab -- backup-utility --restore -t <BACKUP_TIMESTAMP>

# Scale up
kubectl scale deploy -n gitlab -l app=sidekiq --replicas=1
kubectl scale deploy -n gitlab -l app=webservice --replicas=2
```

## Database Backup (Percona)

See k8s-databases skill for PostgreSQL backup via Percona Operator.