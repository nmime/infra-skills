# MinIO Installation

All scripts are **idempotent** - safe to run multiple times. Uses `helm upgrade --install` for convergent behavior.

## ⚠️ Important: Image Source Change (October 2025)

MinIO no longer provides official Docker images. Use alternatives:

| Option | Image | Helm Repo |
|--------|-------|-----------|
| **Chainguard** | `cgr.dev/chainguard/minio` | N/A (use image override) |
| **Bitnami** | `bitnami/minio` | `bitnami/minio` |

## Version Information (January 2026)

| Component | Image |
|-----------|-------|
| MinIO (Chainguard) | `cgr.dev/chainguard/minio:latest` |
| MinIO (Bitnami) | `bitnami/minio:2024.12.18` |
| Bitnami Helm Chart | 17.0.21 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-minio.sh

set -euo pipefail

MINIO_NAMESPACE="minio"
STORAGE_CLASS="${1:-hcloud-volumes}"
STORAGE_SIZE="${2:-100Gi}"
REPLICAS="${3:-4}"

echo "============================================"
echo "MinIO Installation"
echo "============================================"
echo "Namespace: ${MINIO_NAMESPACE}"
echo "Storage Class: ${STORAGE_CLASS}"
echo "Storage Size: ${STORAGE_SIZE} per replica"
echo "Replicas: ${REPLICAS}"
echo "============================================"

# Add Helm repo (use Bitnami since official MinIO charts deprecated)
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Create namespace
kubectl create namespace ${MINIO_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Generate credentials
ROOT_USER="minio-admin"
ROOT_PASSWORD=$(openssl rand -base64 32 | tr -dc 'a-zA-Z0-9' | head -c 32)

# Create credentials secret
kubectl create secret generic minio-credentials \
  --namespace ${MINIO_NAMESPACE} \
  --from-literal=rootUser="${ROOT_USER}" \
  --from-literal=rootPassword="${ROOT_PASSWORD}" \
  --dry-run=client -o yaml | kubectl apply -f -

# Create values file
cat > /tmp/minio-values.yaml << EOF
# MinIO Distributed Mode
mode: distributed

# Replicas (must be >= 4 for distributed)
replicas: ${REPLICAS}

# Use existing secret for credentials
existingSecret: minio-credentials

# Persistence
persistence:
  enabled: true
  storageClass: "${STORAGE_CLASS}"
  size: ${STORAGE_SIZE}
  accessMode: ReadWriteOnce

# Resources
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 2Gi

# Service
service:
  type: ClusterIP
  port: 9000

# Console
consoleService:
  type: ClusterIP
  port: 9001

# Metrics for Prometheus/VictoriaMetrics
metrics:
  serviceMonitor:
    enabled: true
    namespace: monitoring
    interval: 30s

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  maxUnavailable: 1

# Anti-affinity for HA
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: minio
          topologyKey: kubernetes.io/hostname

# Liveness/Readiness
livenessProbe:
  enabled: true
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  enabled: true
  initialDelaySeconds: 10
  periodSeconds: 10

# Security Context
securityContext:
  enabled: true
  runAsUser: 1000
  runAsGroup: 1000
  fsGroup: 1000

# Environment variables
environment:
  MINIO_BROWSER: "on"
  MINIO_PROMETHEUS_AUTH_TYPE: "public"
  MINIO_UPDATE: "off"
EOF

echo ""
echo "=== Installing MinIO ==="

helm upgrade --install minio bitnami/minio \
  --namespace ${MINIO_NAMESPACE} \
  --values /tmp/minio-values.yaml \
  --timeout 10m \
  --wait

echo ""
echo "=== Waiting for MinIO pods ==="
kubectl rollout status statefulset/minio -n ${MINIO_NAMESPACE} --timeout=300s

echo ""
echo "============================================"
echo "MinIO Installation Complete!"
echo "============================================"
echo ""
echo "Credentials:"
echo "  Root User: ${ROOT_USER}"
echo "  Root Password: ${ROOT_PASSWORD}"
echo ""
echo "Internal Endpoints:"
echo "  API: http://minio.${MINIO_NAMESPACE}.svc.cluster.local:9000"
echo "  Console: http://minio-console.${MINIO_NAMESPACE}.svc.cluster.local:9001"
echo ""
echo "Access Console:"
echo "  kubectl port-forward svc/minio-console -n minio 9001:9001"
echo "  Open: http://localhost:9001"
echo ""
echo "Save credentials securely!"
echo "============================================"
```

## Gateway API Route (Optional)

```yaml
# minio-routes.yaml
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: minio-console
  namespace: minio
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "minio.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: minio-console
          port: 9001
---
# API endpoint (for external access if needed)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: minio-api
  namespace: minio
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "s3.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: minio
          port: 9000
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: minio-tls
  namespace: cilium-secrets
spec:
  secretName: minio-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - minio.example.com
    - s3.example.com
```

## Install MinIO Client (mc)

```bash
#!/bin/bash
# scripts/install-mc.sh

# Download mc
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/

# Configure alias
ROOT_USER=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootUser}' | base64 -d)
ROOT_PASSWORD=$(kubectl get secret minio-credentials -n minio -o jsonpath='{.data.rootPassword}' | base64 -d)

# Port forward in background for local access
kubectl port-forward svc/minio -n minio 9000:9000 &
PF_PID=$!
sleep 3

# Add alias
mc alias set myminio http://localhost:9000 ${ROOT_USER} ${ROOT_PASSWORD}

# Test connection
mc admin info myminio

# Kill port-forward
kill $PF_PID 2>/dev/null || true

echo ""
echo "mc configured! Use 'mc alias set' for permanent config"
```

## Verify Installation

```bash
# Check pods
kubectl get pods -n minio

# Check PVCs
kubectl get pvc -n minio

# Check service
kubectl get svc -n minio

# Check logs
kubectl logs -n minio -l app=minio --tail=50

# Test health
kubectl exec -n minio minio-0 -- curl -s http://localhost:9000/minio/health/live
```