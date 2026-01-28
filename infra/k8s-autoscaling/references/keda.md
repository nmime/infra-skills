# KEDA Installation

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| KEDA | 2.18.2 |
| Helm Chart | 2.18.2 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-keda.sh

set -euo pipefail

KEDA_VERSION="2.18.2"

echo "=== Installing KEDA ${KEDA_VERSION} ==="

helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm upgrade --install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --version ${KEDA_VERSION} \
  --set metricsServer.useHostNetwork=false \
  --set prometheus.metricServer.enabled=true \
  --set prometheus.operator.enabled=true \
  --set prometheus.operator.serviceMonitor.enabled=true \
  --wait

echo "=== KEDA Installed ==="
kubectl get pods -n keda
```

## Basic ScaledObject

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: backend
  minReplicaCount: 2
  maxReplicaCount: 20
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "70"
```

## Scale to Zero

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-scaler
  namespace: myapp
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0
  maxReplicaCount: 50
  pollingInterval: 5
  cooldownPeriod: 30
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://vmselect-vmcluster.monitoring.svc:8481/select/0/prometheus
        metricName: pending_jobs
        threshold: "5"
        query: sum(redis_queue_length{queue="jobs"})
```