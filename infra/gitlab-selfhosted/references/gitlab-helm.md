# GitLab Helm Installation

## Version Information (Latest - January 2025)

| Component | Version |
|-----------|---------|
| GitLab | 18.7.1 |
| Helm Chart | 9.7.1 |
| Container Registry | 4.14.0-gitlab |
| GitLab Shell | 14.39.0 |
| Gitaly | 18.7.1 |
| KAS | 18.7.0 |

## Prerequisites

1. **Kubernetes cluster** with:
   - Cilium CNI + Gateway API (k8s-cluster-management)
   - cert-manager for TLS
   - StorageClass (hcloud-volumes)
   - Minimum 3 worker nodes, 8GB RAM each

2. **External services**:
   - PostgreSQL 14+ (Percona Operator recommended)
   - S3-compatible storage (MinIO or cloud)

3. **DNS records** pointing to Gateway IP:
   - `gitlab.example.com`
   - `registry.example.com`
   - `kas.example.com` (for GitLab Agent)

## Installation Script

```bash
#!/bin/bash
# scripts/install-gitlab.sh

set -euo pipefail

GITLAB_CHART_VERSION="9.7.1"
GITLAB_NAMESPACE="gitlab"
DOMAIN="${1:-example.com}"
EMAIL="${2:-admin@example.com}"

echo "============================================"
echo "GitLab Installation"
echo "============================================"
echo "Chart Version: ${GITLAB_CHART_VERSION}"
echo "Domain: ${DOMAIN}"
echo "Email: ${EMAIL}"
echo "============================================"

# Add Helm repo
helm repo add gitlab https://charts.gitlab.io/
helm repo update

# Create namespace
kubectl create namespace ${GITLAB_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Create values file
cat > /tmp/gitlab-values.yaml << 'EOF'
#=============================================
# GitLab Helm Chart Values - Production
#=============================================

global:
  # GitLab version
  gitlabVersion: "18.7.1"
  
  # Hosts configuration
  hosts:
    domain: DOMAIN_PLACEHOLDER
    gitlab:
      name: gitlab.DOMAIN_PLACEHOLDER
      https: true
    registry:
      name: registry.DOMAIN_PLACEHOLDER
      https: true
    kas:
      name: kas.DOMAIN_PLACEHOLDER
      https: true
    minio:
      name: minio.DOMAIN_PLACEHOLDER
      https: true
  
  # Ingress - Disabled (using Gateway API)
  ingress:
    enabled: false
    configureCertmanager: false
  
  # TLS configuration
  certificates:
    customCAs: []
  
  # Time zone
  time_zone: UTC
  
  # Email configuration
  email:
    from: gitlab@DOMAIN_PLACEHOLDER
    display_name: GitLab
    reply_to: noreply@DOMAIN_PLACEHOLDER
  
  # External PostgreSQL (Percona Operator)
  psql:
    host: gitlab-pg-pgbouncer.databases.svc.cluster.local
    port: 5432
    database: gitlabhq_production
    username: gitlab
    password:
      secret: gitlab-postgresql-password
      key: postgresql-password
  
  # Registry configuration
  registry:
    enabled: true
    bucket: gitlab-registry
  
  # Object storage for all components
  appConfig:
    # Enable container registry
    registry:
      enabled: true
    
    # Object storage
    object_store:
      enabled: true
      proxy_download: true
      connection:
        secret: gitlab-object-storage
        key: connection
    
    # LFS
    lfs:
      enabled: true
      bucket: gitlab-lfs
    
    # Artifacts
    artifacts:
      enabled: true
      bucket: gitlab-artifacts
    
    # Uploads
    uploads:
      enabled: true
      bucket: gitlab-uploads
    
    # Packages
    packages:
      enabled: true
      bucket: gitlab-packages
    
    # Terraform state
    terraformState:
      enabled: true
      bucket: gitlab-terraform-state
    
    # CI Secure Files
    ciSecureFiles:
      enabled: true
      bucket: gitlab-ci-secure-files
    
    # Dependency proxy
    dependencyProxy:
      enabled: true
      bucket: gitlab-dependency-proxy
    
    # Backups
    backups:
      bucket: gitlab-backups
      tmpBucket: gitlab-tmp

# Disable bundled PostgreSQL
postgresql:
  install: false

# Internal Redis (can be external for HA)
redis:
  install: true
  architecture: standalone
  master:
    persistence:
      enabled: true
      size: 8Gi
      storageClass: hcloud-volumes

# GitLab Webservice
gitlab:
  webservice:
    replicaCount: 2
    minReplicas: 2
    maxReplicas: 10
    resources:
      requests:
        cpu: 300m
        memory: 2.5Gi
      limits:
        cpu: 2000m
        memory: 5Gi
    workerProcesses: 2
    hpa:
      cpu:
        targetAverageValue: 400m
  
  # Sidekiq (background jobs)
  sidekiq:
    replicaCount: 2
    minReplicas: 1
    maxReplicas: 10
    resources:
      requests:
        cpu: 200m
        memory: 1.5Gi
      limits:
        cpu: 1500m
        memory: 3Gi
  
  # GitLab Shell (SSH)
  gitlab-shell:
    replicaCount: 2
    minReplicas: 2
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 256Mi
  
  # Gitaly (Git storage)
  gitaly:
    persistence:
      enabled: true
      size: 50Gi
      storageClass: hcloud-volumes
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 1500m
        memory: 2Gi
  
  # KAS (Kubernetes Agent Server)
  kas:
    enabled: true
    minReplicas: 2
  
  # Toolbox (for backups, rails console)
  toolbox:
    enabled: true
    backups:
      cron:
        enabled: true
        schedule: "0 2 * * *"
        extraArgs: ""
      objectStorage:
        config:
          secret: gitlab-object-storage
          key: connection

# Container Registry
registry:
  enabled: true
  replicaCount: 2
  hpa:
    minReplicas: 2
    maxReplicas: 5
  storage:
    secret: gitlab-registry-storage
    key: config
  maintenance:
    gc:
      disabled: false
      schedule: "0 4 * * 0"  # Weekly on Sunday at 4 AM
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 256Mi

# Disable components we don't need
prometheus:
  install: false  # Using VictoriaMetrics

grafana:
  install: false  # Using external Grafana

certmanager:
  install: false  # Already installed

nginx-ingress:
  enabled: false  # Using Cilium Gateway API

gitlab-runner:
  install: false  # Installing separately

minio:
  install: false  # Using external S3

shared-secrets:
  enabled: true
  rbac:
    create: true
EOF

# Replace domain placeholder
sed -i "s/DOMAIN_PLACEHOLDER/${DOMAIN}/g" /tmp/gitlab-values.yaml

echo ""
echo "=== Creating Secrets ==="

# Check if PostgreSQL secret exists
if ! kubectl get secret gitlab-postgresql-password -n ${GITLAB_NAMESPACE} &>/dev/null; then
  echo "Creating PostgreSQL password secret..."
  echo "Please get the password from Percona PG secret:"
  echo "  kubectl get secret gitlab-pg-pguser-gitlab -n databases -o jsonpath='{.data.password}' | base64 -d"
  read -sp "Enter PostgreSQL password: " PG_PASSWORD
  echo ""
  
  kubectl create secret generic gitlab-postgresql-password \
    --namespace ${GITLAB_NAMESPACE} \
    --from-literal=postgresql-password="${PG_PASSWORD}"
fi

# Check if object storage secret exists
if ! kubectl get secret gitlab-object-storage -n ${GITLAB_NAMESPACE} &>/dev/null; then
  echo ""
  echo "Creating object storage secret..."
  echo "Please provide S3 credentials:"
  read -p "S3 Access Key: " S3_ACCESS_KEY
  read -sp "S3 Secret Key: " S3_SECRET_KEY
  echo ""
  read -p "S3 Endpoint (e.g., https://s3.amazonaws.com): " S3_ENDPOINT
  read -p "S3 Region (e.g., eu-central-1): " S3_REGION
  
  cat <<EOFS | kubectl apply -n ${GITLAB_NAMESPACE} -f -
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-object-storage
type: Opaque
stringData:
  connection: |
    provider: AWS
    region: ${S3_REGION}
    aws_access_key_id: "${S3_ACCESS_KEY}"
    aws_secret_access_key: "${S3_SECRET_KEY}"
    endpoint: "${S3_ENDPOINT}"
    path_style: true
EOFS
fi

# Registry storage secret
if ! kubectl get secret gitlab-registry-storage -n ${GITLAB_NAMESPACE} &>/dev/null; then
  echo "Creating registry storage secret..."
  S3_ACCESS_KEY=$(kubectl get secret gitlab-object-storage -n ${GITLAB_NAMESPACE} -o jsonpath='{.data.connection}' | base64 -d | grep aws_access_key_id | cut -d'"' -f2)
  S3_SECRET_KEY=$(kubectl get secret gitlab-object-storage -n ${GITLAB_NAMESPACE} -o jsonpath='{.data.connection}' | base64 -d | grep aws_secret_access_key | cut -d'"' -f2)
  S3_ENDPOINT=$(kubectl get secret gitlab-object-storage -n ${GITLAB_NAMESPACE} -o jsonpath='{.data.connection}' | base64 -d | grep endpoint | cut -d'"' -f2)
  S3_REGION=$(kubectl get secret gitlab-object-storage -n ${GITLAB_NAMESPACE} -o jsonpath='{.data.connection}' | base64 -d | grep region | head -1 | cut -d' ' -f2)
  
  cat <<EOFS | kubectl apply -n ${GITLAB_NAMESPACE} -f -
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-registry-storage
type: Opaque
stringData:
  config: |
    s3:
      bucket: gitlab-registry
      accesskey: "${S3_ACCESS_KEY}"
      secretkey: "${S3_SECRET_KEY}"
      region: ${S3_REGION}
      regionendpoint: "${S3_ENDPOINT}"
      v4auth: true
      pathstyle: true
      rootdirectory: /registry
EOFS
fi

echo ""
echo "=== Installing GitLab ==="
echo "This will take 10-20 minutes..."
echo ""

helm upgrade --install gitlab gitlab/gitlab \
  --namespace ${GITLAB_NAMESPACE} \
  --version ${GITLAB_CHART_VERSION} \
  --values /tmp/gitlab-values.yaml \
  --timeout 30m \
  --wait

echo ""
echo "============================================"
echo "GitLab Installation Complete!"
echo "============================================"
echo ""
echo "Get root password:"
echo "  kubectl get secret gitlab-gitlab-initial-root-password -n gitlab -o jsonpath='{.data.password}' | base64 -d; echo"
echo ""
echo "Now setup Gateway API routes:"
echo "  kubectl apply -f gitlab-routes.yaml"
echo ""
echo "Access GitLab:"
echo "  https://gitlab.${DOMAIN}"
echo ""
echo "Access Registry:"
echo "  https://registry.${DOMAIN}"
```

## Gateway API Routes

```yaml
# gitlab-routes.yaml
---
# GitLab Webservice Route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gitlab-webservice
  namespace: gitlab
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "gitlab.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: gitlab-webservice-default
          port: 8181
---
# Container Registry Route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gitlab-registry
  namespace: gitlab
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "registry.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: gitlab-registry
          port: 5000
---
# KAS (Kubernetes Agent Server) Route
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: gitlab-kas
  namespace: gitlab
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "kas.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: gitlab-kas
          port: 8150
---
# TLS Certificates
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: gitlab-tls
  namespace: cilium-secrets
spec:
  secretName: gitlab-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - gitlab.example.com
    - registry.example.com
    - kas.example.com
```

## Verify Installation

```bash
# Check all pods
kubectl get pods -n gitlab

# Check services
kubectl get svc -n gitlab

# Check PVCs
kubectl get pvc -n gitlab

# Get root password
kubectl get secret gitlab-gitlab-initial-root-password -n gitlab \
  -o jsonpath='{.data.password}' | base64 -d; echo

# Check webservice logs
kubectl logs -n gitlab -l app=webservice -c webservice --tail=50

# Check migrations completed
kubectl get jobs -n gitlab

# Test registry
docker login registry.example.com
```