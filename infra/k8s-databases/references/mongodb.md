# MongoDB ReplicaSet

Percona Server MongoDB Operator v1.21.2 with MongoDB 8.0.x.

## Install Operator

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install psmdb-operator percona/psmdb-operator \
  --namespace psmdb-operator --create-namespace \
  --version 1.21.2 --wait

kubectl get pods -n psmdb-operator
```

## ReplicaSet Cluster

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
      affinity:
        antiAffinityTopologyKey: kubernetes.io/hostname
      resources:
        requests:
          cpu: 200m
          memory: 512Mi
        limits:
          cpu: 1000m
          memory: 2Gi
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 20Gi

  sharding:
    enabled: false

  backup:
    enabled: true
    storages:
      s3-backup:
        type: s3
        s3:
          bucket: mongodb-backups
          credentialsSecret: mongo-backup-s3
    tasks:
      - name: daily
        enabled: true
        schedule: "0 2 * * *"
        storageName: s3-backup
        compressionType: gzip
```

## Get Connection

```bash
kubectl get secret myapp-mongo-secrets -n databases \
  -o jsonpath='{.data.MONGODB_DATABASE_ADMIN_URI}' | base64 -d
```

## Enable Monitoring

```yaml
spec:
  pmm:
    enabled: true
    serverHost: monitoring-service
```

## Single Instance (Minimal Tier)

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: mongodb
  namespace: databases
spec:
  crVersion: "1.21.2"
  image: percona/percona-server-mongodb:8.0.17-6

  replsets:
    - name: rs0
      size: 1
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 500m
          memory: 512Mi
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 10Gi

  sharding:
    enabled: false
```
