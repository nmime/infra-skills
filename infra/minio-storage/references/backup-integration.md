# Backup Integration with MinIO

## PostgreSQL Backups (Percona Operator)

```bash
#!/bin/bash
# scripts/create-postgres-backup-secret.sh

MINIO_NAMESPACE="minio"
DATABASES_NAMESPACE="databases"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: postgres-backup-s3
  namespace: ${DATABASES_NAMESPACE}
type: Opaque
stringData:
  s3.conf: |
    [global]
    repo1-s3-key=${MINIO_USER}
    repo1-s3-key-secret=${MINIO_PASSWORD}
EOF
```

### Percona PG Cluster with MinIO Backups

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: myapp-pg
  namespace: databases
spec:
  # ... other config ...
  
  backups:
    pgbackrest:
      global:
        repo1-path: /pgbackrest/myapp-pg/repo1
        repo1-retention-full: "7"
        repo1-retention-full-type: count
        repo1-s3-uri-style: path
      
      repos:
        - name: repo1
          s3:
            bucket: postgres-backups
            endpoint: minio.minio.svc.cluster.local:9000
            region: us-east-1
          schedules:
            full: "0 1 * * 0"      # Weekly
            incremental: "0 1 * * 1-6"  # Daily
```

## MongoDB Backups (Percona Operator)

```bash
#!/bin/bash
# scripts/create-mongodb-backup-secret.sh

MINIO_NAMESPACE="minio"
DATABASES_NAMESPACE="databases"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-backup-s3
  namespace: ${DATABASES_NAMESPACE}
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "${MINIO_USER}"
  AWS_SECRET_ACCESS_KEY: "${MINIO_PASSWORD}"
EOF
```

### Percona MongoDB Cluster with MinIO Backups

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: myapp-mongo
  namespace: databases
spec:
  # ... other config ...
  
  backup:
    enabled: true
    image: percona/percona-backup-mongodb:2.8.0
    storages:
      minio-backup:
        type: s3
        s3:
          bucket: mongodb-backups
          credentialsSecret: mongodb-backup-s3
          region: us-east-1
          endpointUrl: http://minio.minio.svc.cluster.local:9000
          prefix: mongodb
          insecureSkipTLSVerify: true
    tasks:
      - name: daily-backup
        enabled: true
        schedule: "0 2 * * *"
        storageName: minio-backup
        compressionType: gzip
```

## Velero Cluster Backups

```bash
#!/bin/bash
# scripts/install-velero-with-minio.sh

MINIO_NAMESPACE="minio"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

# Create credentials file
cat > /tmp/velero-credentials << EOF
[default]
aws_access_key_id=${MINIO_USER}
aws_secret_access_key=${MINIO_PASSWORD}
EOF

# Install Velero
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.10.0 \
  --bucket velero-backups \
  --secret-file /tmp/velero-credentials \
  --backup-location-config region=us-east-1,s3ForcePathStyle=true,s3Url=http://minio.minio.svc.cluster.local:9000 \
  --snapshot-location-config region=us-east-1 \
  --use-volume-snapshots=false

rm /tmp/velero-credentials

echo "Velero installed with MinIO backend!"
```