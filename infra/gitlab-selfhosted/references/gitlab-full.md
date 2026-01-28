# GitLab Full Mode

For medium/production tiers with HA requirements.

## Installation

```bash
#!/bin/bash
# scripts/install-gitlab-full.sh

set -euo pipefail

DOMAIN="${1:-example.com}"
REPLICAS="${2:-2}"

echo "=== Installing GitLab (Full Mode) ==="

helm repo add gitlab https://charts.gitlab.io/
helm repo update

cat > /tmp/gitlab-full-values.yaml << 'EOF'
global:
  edition: ce
  hosts:
    domain: DOMAIN_PLACEHOLDER
  ingress:
    enabled: false
  psql:
    host: postgresql.databases.svc.cluster.local
    password:
      secret: gitlab-postgresql-password
      key: password

postgresql:
  install: false
redis:
  install: true
  architecture: standalone
  master:
    persistence:
      size: 8Gi
minio:
  install: false
prometheus:
  install: false
nginx-ingress:
  enabled: false
certmanager:
  install: false

gitlab:
  webservice:
    replicaCount: REPLICAS_PLACEHOLDER
    minReplicas: REPLICAS_PLACEHOLDER
    maxReplicas: 10
    resources:
      requests:
        cpu: 500m
        memory: 3Gi
      limits:
        cpu: 3000m
        memory: 6Gi
    workerProcesses: 4
    hpa:
      cpu:
        targetAverageUtilization: 75
  
  sidekiq:
    replicaCount: REPLICAS_PLACEHOLDER
    minReplicas: REPLICAS_PLACEHOLDER
    maxReplicas: 10
    resources:
      requests:
        cpu: 300m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 4Gi
  
  gitlab-shell:
    replicaCount: 2
    minReplicas: 2
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
  
  gitaly:
    resources:
      requests:
        cpu: 300m
        memory: 512Mi
      limits:
        cpu: 2000m
        memory: 2Gi
    persistence:
      size: 50Gi
  
  kas:
    enabled: true
    minReplicas: 2

registry:
  enabled: true
  replicaCount: 2
  hpa:
    minReplicas: 2
    maxReplicas: 5
  resources:
    requests:
      cpu: 100m
      memory: 128Mi

gitlab-runner:
  install: false
EOF

sed -i "s/DOMAIN_PLACEHOLDER/${DOMAIN}/g" /tmp/gitlab-full-values.yaml
sed -i "s/REPLICAS_PLACEHOLDER/${REPLICAS}/g" /tmp/gitlab-full-values.yaml

helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab \
  --create-namespace \
  --values /tmp/gitlab-full-values.yaml \
  --timeout 30m \
  --wait

echo "=== GitLab Full Installed ==="
```