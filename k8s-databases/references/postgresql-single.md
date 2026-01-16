# PostgreSQL Single Instance

For minimal/small tiers.

## Installation

```bash
#!/bin/bash
# scripts/install-postgresql-single.sh

set -euo pipefail

STORAGE_SIZE="${1:-10Gi}"

echo "=== Installing PostgreSQL (Single Instance) ==="

helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

# Install operator
helm upgrade --install pg-operator percona/pg-operator \
  --namespace pg-operator \
  --create-namespace \
  --set resources.requests.cpu=50m \
  --set resources.requests.memory=64Mi \
  --wait

# Create single instance cluster
cat <<EOF | kubectl apply -f -
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: postgresql
  namespace: databases
spec:
  crVersion: "2.8.2"
  postgresVersion: 17
  
  instances:
    - name: instance1
      replicas: 1  # Single instance!
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 500m
          memory: 512Mi
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: ${STORAGE_SIZE}
  
  users:
    - name: app
      databases:
        - app
    - name: gitlab
      databases:
        - gitlabhq_production
  
  # Simple backup to MinIO
  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              storageClassName: hcloud-volumes
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 5Gi
  
  # No PgBouncer for minimal - direct connection
  proxy:
    pgBouncer:
      replicas: 1
      resources:
        requests:
          cpu: 25m
          memory: 32Mi
      config:
        global:
          pool_mode: session
          max_client_conn: "100"
EOF

echo "=== PostgreSQL Single Installed ==="
echo "Connection: kubectl get secret postgresql-pguser-app -n databases -o jsonpath='{.data.uri}' | base64 -d"
```

## Resource Summary

| Component | CPU | Memory |
|-----------|-----|--------|
| Operator | 50m | 64Mi |
| PostgreSQL | 100-500m | 256-512Mi |
| PgBouncer | 25m | 32Mi |
| **Total** | **~175m-575m** | **~350-600Mi** |