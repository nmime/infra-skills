# VictoriaMetrics Installation

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| VictoriaMetrics | v1.133.0 |
| VM Operator | v0.52.0 |
| Grafana | 12.4.0 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-victoriametrics.sh

set -euo pipefail

VM_OPERATOR_VERSION="0.52.0"
GRAFANA_CHART_VERSION="8.10.0"

echo "=== Installing VictoriaMetrics Operator ==="

helm repo add vm https://victoriametrics.github.io/helm-charts/
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install vm-operator vm/victoria-metrics-operator \
  --namespace monitoring \
  --create-namespace \
  --version ${VM_OPERATOR_VERSION} \
  --wait

echo "=== Creating VMCluster ==="

cat <<EOF | kubectl apply -f -
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMCluster
metadata:
  name: vmcluster
  namespace: monitoring
spec:
  retentionPeriod: "30d"
  replicationFactor: 2
  vmstorage:
    replicaCount: 2
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 50Gi
  vmselect:
    replicaCount: 2
    cacheMountPath: /cache
  vminsert:
    replicaCount: 2
EOF

echo "=== Creating VMAgent ==="

cat <<EOF | kubectl apply -f -
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAgent
metadata:
  name: vmagent
  namespace: monitoring
spec:
  selectAllByDefault: true
  replicaCount: 2
  remoteWrite:
    - url: http://vminsert-vmcluster.monitoring.svc:8480/insert/0/prometheus
EOF

echo "=== Installing Grafana ==="

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --version ${GRAFANA_CHART_VERSION} \
  --set persistence.enabled=true \
  --set persistence.storageClassName=hcloud-volumes \
  --set persistence.size=5Gi \
  --set adminPassword=admin \
  --set 'datasources.datasources\.yaml.apiVersion=1' \
  --set 'datasources.datasources\.yaml.datasources[0].name=VictoriaMetrics' \
  --set 'datasources.datasources\.yaml.datasources[0].type=prometheus' \
  --set 'datasources.datasources\.yaml.datasources[0].url=http://vmselect-vmcluster:8481/select/0/prometheus' \
  --set 'datasources.datasources\.yaml.datasources[0].isDefault=true' \
  --wait

echo "=== VictoriaMetrics Stack Installed ==="
kubectl get pods -n monitoring
```