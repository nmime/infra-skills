# Percona MongoDB Operator

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| Operator | 1.21.2 |
| MongoDB | 8.0.17-6 / 7.0.28-15 |
| PBM | 2.8.x |

## Installation Script

```bash
#!/bin/bash
# scripts/install-mongo-operator.sh

set -euo pipefail

OPERATOR_VERSION="1.21.2"

echo "=== Installing Percona MongoDB Operator ==="

helm repo add percona https://percona.github.io/percona-helm-charts/
helm repo update

helm upgrade --install psmdb-operator percona/psmdb-operator \
  --namespace psmdb-operator \
  --create-namespace \
  --version ${OPERATOR_VERSION} \
  --wait

echo "=== Operator Installed ==="
kubectl get pods -n psmdb-operator
```

## MongoDB Cluster

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
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 20Gi
      resources:
        requests:
          cpu: 200m
          memory: 512Mi
        limits:
          cpu: 1000m
          memory: 2Gi
  
  sharding:
    enabled: false
  
  backup:
    enabled: true
    storages:
      s3-backup:
        type: s3
        s3:
          bucket: myapp-mongo-backups
          credentialsSecret: mongo-backup-s3
          region: eu-central-1
    tasks:
      - name: daily-backup
        enabled: true
        schedule: "0 2 * * *"
        storageName: s3-backup
        compressionType: gzip
```