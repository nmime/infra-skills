# GitLab Light Mode

For minimal/small tiers with limited resources.

## Installation

```bash
#!/bin/bash
# scripts/install-gitlab-light.sh

set -euo pipefail

DOMAIN="${1:-example.com}"

echo "=== Installing GitLab (Light Mode) ==="

helm repo add gitlab https://charts.gitlab.io/
helm repo update

cat > /tmp/gitlab-light-values.yaml << 'EOF'
global:
  edition: ce
  hosts:
    domain: DOMAIN_PLACEHOLDER
  ingress:
    enabled: false
  # Use external PostgreSQL (Percona) - lighter
  psql:
    host: postgresql.databases.svc.cluster.local
    password:
      secret: gitlab-postgresql-password
      key: password

# Disable bundled components
postgresql:
  install: false
redis:
  install: true
  master:
    persistence:
      size: 2Gi
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
minio:
  install: false  # Use external MinIO
prometheus:
  install: false  # Use external monitoring
nginx-ingress:
  enabled: false  # Use Cilium Gateway
certmanager:
  install: false

# Light resource allocation
gitlab:
  webservice:
    replicaCount: 1
    minReplicas: 1
    maxReplicas: 2
    resources:
      requests:
        cpu: 200m
        memory: 1.5Gi
      limits:
        cpu: 1500m
        memory: 3Gi
    workerProcesses: 2
  
  sidekiq:
    replicaCount: 1
    minReplicas: 1
    maxReplicas: 2
    resources:
      requests:
        cpu: 100m
        memory: 1Gi
      limits:
        cpu: 1000m
        memory: 2Gi
  
  gitlab-shell:
    replicaCount: 1
    minReplicas: 1
    resources:
      requests:
        cpu: 25m
        memory: 32Mi
  
  gitaly:
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 1Gi
    persistence:
      size: 20Gi
  
  toolbox:
    resources:
      requests:
        cpu: 25m
        memory: 64Mi

registry:
  enabled: true
  replicaCount: 1
  resources:
    requests:
      cpu: 25m
      memory: 32Mi

gitlab-runner:
  install: false  # Install separately
EOF

sed -i "s/DOMAIN_PLACEHOLDER/${DOMAIN}/g" /tmp/gitlab-light-values.yaml

# Create secrets first
./scripts/create-gitlab-minio-secrets.sh
./scripts/create-gitlab-db-secret.sh

helm upgrade --install gitlab gitlab/gitlab \
  --namespace gitlab \
  --create-namespace \
  --values /tmp/gitlab-light-values.yaml \
  --timeout 20m \
  --wait

echo "=== GitLab Light Installed ==="
echo "Get password: kubectl get secret gitlab-gitlab-initial-root-password -n gitlab -o jsonpath='{.data.password}' | base64 -d"
```

## Resource Summary (Light Mode)

| Component | CPU | Memory |
|-----------|-----|--------|
| Webservice | 200m-1.5 | 1.5-3Gi |
| Sidekiq | 100m-1 | 1-2Gi |
| Gitaly | 100m-1 | 256Mi-1Gi |
| Registry | 25m | 32Mi |
| Redis | 50m | 64Mi |
| **Total** | **~500m-5** | **~3-7Gi** |