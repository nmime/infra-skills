# Loki Logging Stack

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| Loki | 3.6.3 |
| Promtail | 3.6.3 |
| Loki Helm Chart | 6.29.0 |

## Storage Backend Options

### Option 1: MinIO (Recommended for Self-Hosted)

Use MinIO from the `minio-storage` skill.

#### Prerequisites

```bash
# Install MinIO first (from minio-storage skill)
./scripts/install-minio.sh
./scripts/create-buckets.sh
```

#### Create Loki MinIO Secret

```bash
#!/bin/bash
# scripts/create-loki-minio-secret.sh

MINIO_NAMESPACE="minio"
MONITORING_NAMESPACE="monitoring"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

kubectl create namespace ${MONITORING_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: loki-minio-credentials
  namespace: ${MONITORING_NAMESPACE}
type: Opaque
stringData:
  MINIO_ACCESS_KEY_ID: "${MINIO_USER}"
  MINIO_SECRET_ACCESS_KEY: "${MINIO_PASSWORD}"
EOF

echo "Loki MinIO secret created!"
```

#### Install Loki with MinIO Backend

```bash
#!/bin/bash
# scripts/install-loki-minio.sh

set -euo pipefail

LOKI_CHART_VERSION="6.29.0"

echo "=== Installing Loki with MinIO Backend ==="

# Create secret first
./scripts/create-loki-minio-secret.sh

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

cat > /tmp/loki-minio-values.yaml << 'EOF'
deploymentMode: SimpleScalable

loki:
  auth_enabled: false
  
  schemaConfig:
    configs:
      - from: 2024-01-01
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h
  
  storage:
    type: s3
    s3:
      endpoint: minio.minio.svc.cluster.local:9000
      bucketnames: loki-chunks
      region: us-east-1
      insecure: true
      s3forcepathstyle: true
  
  commonConfig:
    replication_factor: 2
  
  limits_config:
    retention_period: 336h
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32

# Read credentials from secret
extraEnvFrom:
  - secretRef:
      name: loki-minio-credentials

read:
  replicas: 2
  
write:
  replicas: 2

backend:
  replicas: 2

singleBinary:
  replicas: 0

gateway:
  enabled: true

monitoring:
  selfMonitoring:
    enabled: false
  lokiCanary:
    enabled: false

test:
  enabled: false
EOF

helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --create-namespace \
  --version ${LOKI_CHART_VERSION} \
  --values /tmp/loki-minio-values.yaml \
  --wait

echo "=== Installing Promtail ==="

helm upgrade --install promtail grafana/promtail \
  --namespace monitoring \
  --version 6.16.6 \
  --set config.clients[0].url=http://loki-gateway/loki/api/v1/push \
  --wait

echo "=== Loki Stack Installed with MinIO ==="
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki
```

### Option 2: Filesystem (Simple/Dev)

```bash
#!/bin/bash
# scripts/install-loki-filesystem.sh

helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --create-namespace \
  --version 6.29.0 \
  --set deploymentMode=SimpleScalable \
  --set loki.auth_enabled=false \
  --set loki.commonConfig.replication_factor=2 \
  --set 'loki.schemaConfig.configs[0].from=2024-01-01' \
  --set 'loki.schemaConfig.configs[0].store=tsdb' \
  --set 'loki.schemaConfig.configs[0].object_store=filesystem' \
  --set 'loki.schemaConfig.configs[0].schema=v13' \
  --set read.replicas=2 \
  --set write.replicas=2 \
  --set write.persistence.enabled=true \
  --set write.persistence.size=20Gi \
  --set write.persistence.storageClass=hcloud-volumes \
  --set backend.replicas=2 \
  --set backend.persistence.enabled=true \
  --set backend.persistence.size=20Gi \
  --set backend.persistence.storageClass=hcloud-volumes \
  --set singleBinary.replicas=0 \
  --set gateway.enabled=true \
  --wait
```

## LogQL Query Examples

```promql
# All logs from namespace
{namespace="myapp"}

# Logs from specific pod
{namespace="myapp", pod=~"backend-.*"}

# Error logs only
{namespace="myapp"} |= "error" | json | level="error"

# Count errors per pod
sum by (pod) (count_over_time({namespace="myapp"} |= "error" [5m]))

# Rate of logs
sum(rate({namespace="myapp"}[1m])) by (pod)
```

## Verify Installation

```bash
# Check pods
kubectl get pods -n monitoring -l app.kubernetes.io/name=loki

# Check logs
kubectl logs -n monitoring -l app.kubernetes.io/name=loki --tail=20

# Test query
kubectl port-forward -n monitoring svc/loki-gateway 3100:80 &
curl -s "http://localhost:3100/loki/api/v1/labels" | jq
```