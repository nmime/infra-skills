# Loki + MinIO Integration

## Create Loki MinIO Secret

```bash
#!/bin/bash
# scripts/create-loki-minio-secret.sh

set -euo pipefail

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

## Loki Helm Values for MinIO

```yaml
# loki-minio-values.yaml
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
      access_key_id: ${MINIO_ACCESS_KEY_ID}
      secret_access_key: ${MINIO_SECRET_ACCESS_KEY}
      insecure: true
      s3forcepathstyle: true
    
  storage_config:
    tsdb_shipper:
      active_index_directory: /var/loki/tsdb-index
      cache_location: /var/loki/tsdb-cache
  
  commonConfig:
    replication_factor: 2
  
  limits_config:
    retention_period: 336h  # 14 days
    ingestion_rate_mb: 16
    ingestion_burst_size_mb: 32

# Read from environment secret
extraEnvFrom:
  - secretRef:
      name: loki-minio-credentials
```

## Complete Loki Installation with MinIO

```bash
#!/bin/bash
# scripts/install-loki-with-minio.sh

set -euo pipefail

# Create secret first
./scripts/create-loki-minio-secret.sh

# Install Loki with MinIO backend
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --create-namespace \
  --version 6.29.0 \
  --set deploymentMode=SimpleScalable \
  --set loki.auth_enabled=false \
  --set 'loki.storage.type=s3' \
  --set 'loki.storage.s3.endpoint=minio.minio.svc.cluster.local:9000' \
  --set 'loki.storage.s3.bucketnames=loki-chunks' \
  --set 'loki.storage.s3.region=us-east-1' \
  --set 'loki.storage.s3.insecure=true' \
  --set 'loki.storage.s3.s3forcepathstyle=true' \
  --set 'loki.schemaConfig.configs[0].from=2024-01-01' \
  --set 'loki.schemaConfig.configs[0].store=tsdb' \
  --set 'loki.schemaConfig.configs[0].object_store=s3' \
  --set 'loki.schemaConfig.configs[0].schema=v13' \
  --set 'loki.schemaConfig.configs[0].index.prefix=loki_index_' \
  --set 'loki.schemaConfig.configs[0].index.period=24h' \
  --set 'extraEnvFrom[0].secretRef.name=loki-minio-credentials' \
  --set read.replicas=2 \
  --set write.replicas=2 \
  --set backend.replicas=2 \
  --set singleBinary.replicas=0 \
  --set gateway.enabled=true \
  --wait

echo "Loki installed with MinIO backend!"
```