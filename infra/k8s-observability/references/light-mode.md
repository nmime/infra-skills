# Observability Light Mode

For minimal/small tiers with limited resources.

## Installation

```bash
#!/bin/bash
# scripts/install-observability-light.sh

set -euo pipefail

RETENTION="${1:-7d}"

echo "=== Installing Observability (Light Mode) ==="

helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -

# VictoriaMetrics Single (not cluster)
helm upgrade --install vmsingle vm/victoria-metrics-single \
  --namespace monitoring \
  --set server.retentionPeriod=${RETENTION} \
  --set server.resources.requests.cpu=100m \
  --set server.resources.requests.memory=256Mi \
  --set server.resources.limits.cpu=500m \
  --set server.resources.limits.memory=512Mi \
  --set server.persistentVolume.enabled=true \
  --set server.persistentVolume.size=10Gi \
  --wait

# Loki Single Binary (not scalable)
helm upgrade --install loki grafana/loki \
  --namespace monitoring \
  --set deploymentMode=SingleBinary \
  --set loki.auth_enabled=false \
  --set loki.limits_config.retention_period=168h \
  --set singleBinary.replicas=1 \
  --set singleBinary.resources.requests.cpu=100m \
  --set singleBinary.resources.requests.memory=256Mi \
  --set singleBinary.resources.limits.cpu=500m \
  --set singleBinary.resources.limits.memory=512Mi \
  --set singleBinary.persistence.enabled=true \
  --set singleBinary.persistence.size=10Gi \
  --set read.replicas=0 \
  --set write.replicas=0 \
  --set backend.replicas=0 \
  --set gateway.enabled=false \
  --wait

# Promtail
helm upgrade --install promtail grafana/promtail \
  --namespace monitoring \
  --set resources.requests.cpu=25m \
  --set resources.requests.memory=64Mi \
  --set resources.limits.cpu=100m \
  --set resources.limits.memory=128Mi \
  --set config.clients[0].url=http://loki:3100/loki/api/v1/push \
  --wait

# Grafana
helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --set replicas=1 \
  --set resources.requests.cpu=50m \
  --set resources.requests.memory=128Mi \
  --set resources.limits.cpu=200m \
  --set resources.limits.memory=256Mi \
  --set persistence.enabled=true \
  --set persistence.size=2Gi \
  --set 'datasources.datasources\.yaml.apiVersion=1' \
  --set 'datasources.datasources\.yaml.datasources[0].name=VictoriaMetrics' \
  --set 'datasources.datasources\.yaml.datasources[0].type=prometheus' \
  --set 'datasources.datasources\.yaml.datasources[0].url=http://vmsingle-victoria-metrics-single-server:8428' \
  --set 'datasources.datasources\.yaml.datasources[0].isDefault=true' \
  --set 'datasources.datasources\.yaml.datasources[1].name=Loki' \
  --set 'datasources.datasources\.yaml.datasources[1].type=loki' \
  --set 'datasources.datasources\.yaml.datasources[1].url=http://loki:3100' \
  --wait

echo "=== Observability Light Installed ==="
echo "Grafana password: kubectl get secret grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 -d"
```

## Resource Summary (Light Mode)

| Component | CPU | Memory | Storage |
|-----------|-----|--------|--------|
| VictoriaMetrics | 100-500m | 256-512Mi | 10Gi |
| Loki | 100-500m | 256-512Mi | 10Gi |
| Promtail (per node) | 25-100m | 64-128Mi | - |
| Grafana | 50-200m | 128-256Mi | 2Gi |
| **Total** | **~300m-1.3** | **~700Mi-1.4Gi** | **~22Gi** |