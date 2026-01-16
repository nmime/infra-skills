# Percona PostgreSQL Operator

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| Operator | 2.8.2 |
| PostgreSQL | 18.x / 17.x / 16.x |
| PgBouncer | 1.24.x |
| pgBackRest | 2.54 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-pg-operator.sh

set -euo pipefail

OPERATOR_VERSION="2.8.2"

echo "=== Installing Percona PostgreSQL Operator ==="

helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install pg-operator percona/pg-operator \
  --namespace pg-operator \
  --create-namespace \
  --version ${OPERATOR_VERSION} \
  --wait

echo "=== Operator Installed ==="
kubectl get pods -n pg-operator
```

## PostgreSQL Cluster

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
  
  proxy:
    pgBouncer:
      replicas: 2
      config:
        global:
          pool_mode: transaction
          max_client_conn: "1000"
          default_pool_size: "25"
```

## Get Connection

```bash
kubectl get secret myapp-pg-pguser-myapp -n databases \
  -o jsonpath='{.data.uri}' | base64 -d
```