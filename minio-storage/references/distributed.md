# MinIO Distributed Mode

HA cluster for medium/production tiers.

## Installation

```bash
#!/bin/bash
# scripts/install-minio-distributed.sh

set -euo pipefail

REPLICAS="${1:-4}"
STORAGE_SIZE="${2:-100Gi}"
STORAGE_CLASS="${3:-hcloud-volumes}"

echo "=== Installing MinIO Distributed (${REPLICAS} replicas) ==="

helm repo add minio https://charts.min.io/
helm repo update

MINIO_USER="minio-admin"
MINIO_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)

kubectl create namespace minio --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic minio-credentials \
  --namespace minio \
  --from-literal=rootUser="${MINIO_USER}" \
  --from-literal=rootPassword="${MINIO_PASSWORD}" \
  --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install minio minio/minio \
  --namespace minio \
  --set mode=distributed \
  --set replicas=${REPLICAS} \
  --set existingSecret=minio-credentials \
  --set persistence.enabled=true \
  --set persistence.storageClass=${STORAGE_CLASS} \
  --set persistence.size=${STORAGE_SIZE} \
  --set resources.requests.cpu=250m \
  --set resources.requests.memory=512Mi \
  --set resources.limits.cpu=1000m \
  --set resources.limits.memory=1Gi \
  --set podDisruptionBudget.enabled=true \
  --set podDisruptionBudget.maxUnavailable=1 \
  --wait

echo "=== MinIO Distributed Installed ==="
echo "Replicas: ${REPLICAS}"
echo "Total Storage: $((${REPLICAS} * ${STORAGE_SIZE%Gi}))Gi"
```

## Resource Usage

| Resource | Per Pod | Total (4 pods) |
|----------|---------|----------------|
| CPU | 250m-1000m | 1-4 cores |
| Memory | 512Mi-1Gi | 2-4Gi |
| Storage | As configured | 4x size |