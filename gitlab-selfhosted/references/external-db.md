# External Database Configuration

## PostgreSQL via Percona Operator

First, create PostgreSQL cluster using k8s-databases skill:

```yaml
# gitlab-postgres.yaml (using Percona PG Operator)
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: gitlab-pg
  namespace: databases
spec:
  crVersion: "2.4.1"
  image: percona/percona-postgresql-operator:2.4.1-ppg16.3-postgres
  postgresVersion: 16
  
  instances:
    - name: instance1
      replicas: 3
      resources:
        requests:
          cpu: 500m
          memory: 1Gi
        limits:
          cpu: 2000m
          memory: 4Gi
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 50Gi
  
  users:
    - name: gitlab
      databases:
        - gitlabhq_production
      options: "SUPERUSER"
  
  proxy:
    pgBouncer:
      replicas: 2
      config:
        global:
          pool_mode: transaction
          max_client_conn: "1000"
          default_pool_size: "50"
  
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
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 100Gi
```

## Get PostgreSQL Connection Details

```bash
# Get password
export PGPASSWORD=$(kubectl get secret gitlab-pg-pguser-gitlab -n databases \
  -o jsonpath='{.data.password}' | base64 -d)

# Connection string
postgresql://gitlab:${PGPASSWORD}@gitlab-pg-pgbouncer.databases.svc:5432/gitlabhq_production
```

## Create GitLab Database Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-postgresql-password
  namespace: gitlab
type: Opaque
stringData:
  postgresql-password: "<password-from-percona-secret>"
```

## Configure GitLab for External PostgreSQL

```yaml
# In gitlab-values.yaml
postgresql:
  install: false  # Don't install bundled PostgreSQL

global:
  psql:
    host: gitlab-pg-pgbouncer.databases.svc.cluster.local
    port: 5432
    database: gitlabhq_production
    username: gitlab
    password:
      secret: gitlab-postgresql-password
      key: postgresql-password
```

## External Redis (Optional)

For high availability, use external Redis:

```yaml
redis:
  install: false

global:
  redis:
    host: redis.databases.svc.cluster.local
    port: 6379
    password:
      enabled: true
      secret: gitlab-redis-password
      key: password
```