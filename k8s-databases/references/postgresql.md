# PostgreSQL HA Cluster

Percona PostgreSQL Operator v2.8.2 with PostgreSQL 18.x.

## Install Operator

```bash
helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install pg-operator percona/pg-operator \
  --namespace pg-operator --create-namespace \
  --version 2.8.2 --wait

kubectl get pods -n pg-operator
```

## HA Cluster

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
      resources:
        requests:
          cpu: 200m
          memory: 512Mi
        limits:
          cpu: 1000m
          memory: 2Gi
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    postgres-operator.crunchydata.com/cluster: myapp-pg
                topologyKey: kubernetes.io/hostname
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        accessModes:
          - ReadWriteOnce
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
      config:
        global:
          pool_mode: transaction
          max_client_conn: "1000"
          default_pool_size: "25"

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
                  storage: 30Gi
```

## Get Connection

```bash
kubectl get secret myapp-pg-pguser-myapp -n databases \
  -o jsonpath='{.data.uri}' | base64 -d
```

## Enable Monitoring

```yaml
spec:
  monitoring:
    pgmonitor:
      exporter:
        image: percona/postgres_exporter:0.15.0
```
