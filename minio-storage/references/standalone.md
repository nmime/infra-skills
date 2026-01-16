# MinIO Standalone Mode

Single instance for minimal/small tiers.

## Installation

```bash
#!/bin/bash
# scripts/install-minio-standalone.sh

set -euo pipefail

STORAGE_SIZE="${1:-50Gi}"
STORAGE_CLASS="${2:-hcloud-volumes}"

echo "=== Installing MinIO Standalone ==="

helm repo add minio https://charts.min.io/
helm repo update

# Generate credentials
MINIO_USER="minio-admin"
MINIO_PASSWORD=$(openssl rand -base64 24 | tr -dc 'a-zA-Z0-9' | head -c 24)

kubectl create namespace minio --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic minio-credentials \
  --namespace minio \
  --from-literal=rootUser="${MINIO_USER}" \
  --from-literal=rootPassword="${MINIO_PASSWORD}" \
  --dry-run=client -o yaml | kubectl apply -f -

# Install standalone (single replica)
helm upgrade --install minio minio/minio \
  --namespace minio \
  --set mode=standalone \
  --set replicas=1 \
  --set existingSecret=minio-credentials \
  --set persistence.enabled=true \
  --set persistence.storageClass=${STORAGE_CLASS} \
  --set persistence.size=${STORAGE_SIZE} \
  --set resources.requests.cpu=100m \
  --set resources.requests.memory=256Mi \
  --set resources.limits.cpu=500m \
  --set resources.limits.memory=512Mi \
  --set consoleService.type=ClusterIP \
  --wait

echo "=== MinIO Standalone Installed ==="
echo "User: ${MINIO_USER}"
echo "Password: ${MINIO_PASSWORD}"
```

## Resource Usage

| Resource | Standalone | Notes |
|----------|------------|-------|
| CPU | 100-500m | Single pod |
| Memory | 256-512Mi | Light |
| Storage | As configured | Single PVC |