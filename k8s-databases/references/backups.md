# Database Backups with MinIO

## Prerequisites

Install MinIO first from the `minio-storage` skill:

```bash
./scripts/install-minio.sh
./scripts/create-buckets.sh
```

## PostgreSQL Backups (Percona Operator)

### Create Backup Secret

```bash
#!/bin/bash
# scripts/create-postgres-backup-secret.sh

MINIO_NAMESPACE="minio"
DATABASES_NAMESPACE="databases"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

kubectl create namespace ${DATABASES_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

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

echo "PostgreSQL backup secret created!"
```

### Cluster with MinIO Backups

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: myapp-pg
  namespace: databases
spec:
  crVersion: "2.8.2"
  image: percona/percona-postgresql-operator:2.8.2-ppg17-postgres
  postgresVersion: 17
  
  instances:
    - name: instance1
      replicas: 3
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        resources:
          requests:
            storage: 20Gi
  
  users:
    - name: myapp
      databases:
        - myapp
  
  # MinIO Backup Configuration
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
            full: "0 1 * * 0"      # Sunday 1 AM
            incremental: "0 1 * * 1-6"  # Mon-Sat 1 AM
  
  proxy:
    pgBouncer:
      replicas: 2
```

## MongoDB Backups (Percona Operator)

### Create Backup Secret

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

echo "MongoDB backup secret created!"
```

### Cluster with MinIO Backups

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: myapp-mongo
  namespace: databases
spec:
  crVersion: "1.21.2"
  image: percona/percona-server-mongodb:8.0.17-6
  
  secrets:
    users: myapp-mongo-secrets
  
  replsets:
    - name: rs0
      size: 3
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 20Gi
  
  # MinIO Backup Configuration
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

## Manual Backup Commands

```bash
# PostgreSQL - Trigger manual backup
kubectl annotate perconapgcluster myapp-pg -n databases \
  postgres-operator.crunchydata.com/pgbackrest-backup="$(date +%s)" --overwrite

# PostgreSQL - Check backup status
kubectl exec -it myapp-pg-instance1-0 -n databases -- pgbackrest info

# MongoDB - Trigger manual backup
kubectl apply -f - <<EOF
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBBackup
metadata:
  name: manual-$(date +%Y%m%d-%H%M%S)
  namespace: databases
spec:
  clusterName: myapp-mongo
  storageName: minio-backup
EOF

# MongoDB - List backups
kubectl get psmdb-backup -n databases
```

## Restore from Backup

```bash
# PostgreSQL restore
kubectl annotate perconapgcluster myapp-pg -n databases \
  postgres-operator.crunchydata.com/pgbackrest-restore="$(date +%s)" --overwrite

# MongoDB restore
kubectl apply -f - <<EOF
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDBRestore
metadata:
  name: restore-$(date +%Y%m%d)
  namespace: databases
spec:
  clusterName: myapp-mongo
  backupName: daily-backup-XXXXX
EOF
```