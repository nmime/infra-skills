# Database Backups

Percona operators include built-in backup support. Use MinIO for S3-compatible storage.

## PostgreSQL Backup Secret

```bash
MINIO_USER=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootPassword}' | base64 -d)

kubectl create namespace databases --dry-run=client -o yaml | kubectl apply -f -

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: postgres-backup-s3
  namespace: databases
type: Opaque
stringData:
  s3.conf: |
    [global]
    repo1-s3-key=${MINIO_USER}
    repo1-s3-key-secret=${MINIO_PASSWORD}
EOF
```

## PostgreSQL with S3 Backup

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: myapp-pg
  namespace: databases
spec:
  crVersion: "2.8.2"
  postgresVersion: 18

  instances:
    - name: instance1
      replicas: 3
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        resources:
          requests:
            storage: 20Gi

  backups:
    pgbackrest:
      global:
        repo1-path: /pgbackrest/myapp-pg/repo1
        repo1-retention-full: "7"
        repo1-retention-full-type: count
        repo1-s3-uri-style: path
      configuration:
        - secret:
            name: postgres-backup-s3
      repos:
        - name: repo1
          s3:
            bucket: postgres-backups
            endpoint: minio.minio.svc.cluster.local:9000
            region: us-east-1
          schedules:
            full: "0 1 * * 0"
            incremental: "0 1 * * 1-6"
```

## MongoDB Backup Secret

```bash
MINIO_USER=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootPassword}' | base64 -d)

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-backup-s3
  namespace: databases
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "${MINIO_USER}"
  AWS_SECRET_ACCESS_KEY: "${MINIO_PASSWORD}"
EOF
```

## MongoDB with S3 Backup

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: myapp-mongo
  namespace: databases
spec:
  crVersion: "1.21.2"
  image: percona/percona-server-mongodb:8.0.17-6

  replsets:
    - name: rs0
      size: 3
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 20Gi

  backup:
    enabled: true
    storages:
      minio:
        type: s3
        s3:
          bucket: mongodb-backups
          credentialsSecret: mongodb-backup-s3
          region: us-east-1
          endpointUrl: http://minio.minio.svc.cluster.local:9000
          insecureSkipTLSVerify: true
    tasks:
      - name: daily
        enabled: true
        schedule: "0 2 * * *"
        storageName: minio
        compressionType: gzip
```

## Manual Backup

```bash
# PostgreSQL
kubectl annotate perconapgcluster myapp-pg -n databases \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date +%s)" --overwrite

# PostgreSQL - check status
kubectl exec -it myapp-pg-instance1-0 -n databases -- pgbackrest info

# MongoDB
cat <<EOF | kubectl apply -f -
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBBackup
metadata:
  name: manual-$(date +%Y%m%d-%H%M%S)
  namespace: databases
spec:
  clusterName: myapp-mongo
  storageName: minio
EOF

# MongoDB - list backups
kubectl get psmdb-backup -n databases
```

## Restore

```bash
# PostgreSQL
kubectl annotate perconapgcluster myapp-pg -n databases \
  postgres-operator.crunchydata.com/pgbackrest-restore="$(date +%s)" --overwrite

# MongoDB
cat <<EOF | kubectl apply -f -
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBRestore
metadata:
  name: restore-$(date +%Y%m%d)
  namespace: databases
spec:
  clusterName: myapp-mongo
  backupName: <backup-name>
EOF
```
