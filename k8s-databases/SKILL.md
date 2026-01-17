---
name: k8s-databases
description: PostgreSQL and MongoDB on Kubernetes via Percona Operators. Use when deploying databases, configuring HA clusters, setting up backups, monitoring, or performing database operations.
---

# K8s Databases

Percona Operators for PostgreSQL and MongoDB. (Updated: January 2026). All deployments are **idempotent** - operators reconcile to desired state.

**Run from**: Bastion server or any machine with kubectl access.

## Percona Operators

| Operator | Version | Database |
|----------|---------|----------|
| Percona PostgreSQL | v2.8.2 | PostgreSQL 18.x |
| Percona Server MongoDB | v1.21.2 | MongoDB 8.0.x |

> Always use latest versions. Operators include built-in backup, monitoring, and HA management.

## Deployment Tiers

| Tier | PostgreSQL | MongoDB | HA |
|------|------------|---------|-----|
| minimal/small | 1 replica | 1 replica | No |
| medium/production | 3 replicas + PgBouncer | 3 replicas (ReplicaSet) | Yes |

## Installation

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

# PostgreSQL Operator
helm upgrade --install pg-operator percona/pg-operator \
  --namespace pg-operator --create-namespace \
  --version 2.8.2 --wait

# MongoDB Operator
helm upgrade --install psmdb-operator percona/psmdb-operator \
  --namespace psmdb-operator --create-namespace \
  --version 1.21.2 --wait
```

## PostgreSQL HA Cluster

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

  users:
    - name: myapp
      databases:
        - myapp

  proxy:
    pgBouncer:
      replicas: 2

  backups:
    pgbackrest:
      repos:
        - name: repo1
          schedules:
            full: "0 1 * * 0"
            incremental: "0 1 * * 1-6"
          volume:
            volumeClaimSpec:
              storageClassName: hcloud-volumes
              resources:
                requests:
                  storage: 30Gi
```

## MongoDB ReplicaSet

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
        schedule: "0 2 * * *"
        storageName: s3-backup
```

## Get Connections

```bash
# PostgreSQL
kubectl get secret myapp-pg-pguser-myapp -n databases \
  -o jsonpath='{.data.uri}' | base64 -d

# MongoDB
kubectl get secret myapp-mongo-secrets -n databases \
  -o jsonpath='{.data.MONGODB_DATABASE_ADMIN_URI}' | base64 -d
```

## Monitoring with PMM

Percona Monitoring and Management (PMM) provides Query Analytics, metrics, and alerting.

```yaml
# PostgreSQL with PMM
spec:
  pmm:
    enabled: true
    image: percona/pmm-client:2.44.0
    serverHost: pmm-server.monitoring.svc.cluster.local

# MongoDB with PMM
spec:
  pmm:
    enabled: true
    image: percona/pmm-client:2.44.0
    serverHost: pmm-server.monitoring.svc.cluster.local
```

See [references/monitoring.md](references/monitoring.md) for PMM Server deployment and configuration.

## Reference Files

- [references/postgresql.md](references/postgresql.md) - PostgreSQL HA cluster
- [references/postgresql-single.md](references/postgresql-single.md) - PostgreSQL single instance
- [references/mongodb.md](references/mongodb.md) - MongoDB ReplicaSet
- [references/backups.md](references/backups.md) - Backup with MinIO
- [references/monitoring.md](references/monitoring.md) - Metrics and alerting
