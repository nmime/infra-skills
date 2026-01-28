# GitLab + MinIO Integration

## Create GitLab Secrets for MinIO

```bash
#!/bin/bash
# scripts/create-gitlab-minio-secrets.sh

set -euo pipefail

GITLAB_NAMESPACE="gitlab"
MINIO_NAMESPACE="minio"

# Get MinIO credentials
MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)
MINIO_ENDPOINT="http://minio.${MINIO_NAMESPACE}.svc.cluster.local:9000"

echo "=== Creating GitLab MinIO Secrets ==="

# Create namespace if needed
kubectl create namespace ${GITLAB_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Object storage connection secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-object-storage
  namespace: ${GITLAB_NAMESPACE}
type: Opaque
stringData:
  connection: |
    provider: AWS
    region: us-east-1
    aws_access_key_id: "${MINIO_USER}"
    aws_secret_access_key: "${MINIO_PASSWORD}"
    endpoint: "${MINIO_ENDPOINT}"
    path_style: true
EOF

# Registry storage secret (different format)
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-registry-storage
  namespace: ${GITLAB_NAMESPACE}
type: Opaque
stringData:
  config: |
    s3:
      bucket: gitlab-registry
      accesskey: "${MINIO_USER}"
      secretkey: "${MINIO_PASSWORD}"
      region: us-east-1
      regionendpoint: "${MINIO_ENDPOINT}"
      v4auth: true
      pathstyle: true
      rootdirectory: /
    cache:
      blobdescriptor: inmemory
    delete:
      enabled: true
    maintenance:
      uploadpurging:
        enabled: true
        age: 168h
        interval: 24h
        dryrun: false
EOF

# Runner cache secret
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-runner-cache-secret
  namespace: gitlab-runner
type: Opaque
stringData:
  accesskey: "${MINIO_USER}"
  secretkey: "${MINIO_PASSWORD}"
EOF

echo "=== Secrets Created ==="
kubectl get secrets -n ${GITLAB_NAMESPACE} | grep -E "gitlab-(object|registry)"
```

## GitLab Helm Values for MinIO

```yaml
# gitlab-minio-values.yaml
global:
  # MinIO endpoints
  minio:
    enabled: false  # Don't install bundled MinIO
  
  # Object storage configuration
  appConfig:
    object_store:
      enabled: true
      proxy_download: true
      connection:
        secret: gitlab-object-storage
        key: connection
    
    # Individual bucket configs
    artifacts:
      bucket: gitlab-artifacts
      connection:
        secret: gitlab-object-storage
        key: connection
    
    lfs:
      bucket: gitlab-lfs
      connection:
        secret: gitlab-object-storage
        key: connection
    
    uploads:
      bucket: gitlab-uploads
      connection:
        secret: gitlab-object-storage
        key: connection
    
    packages:
      bucket: gitlab-packages
      connection:
        secret: gitlab-object-storage
        key: connection
    
    terraformState:
      bucket: gitlab-terraform-state
      connection:
        secret: gitlab-object-storage
        key: connection
    
    ciSecureFiles:
      bucket: gitlab-ci-secure-files
      connection:
        secret: gitlab-object-storage
        key: connection
    
    dependencyProxy:
      bucket: gitlab-dependency-proxy
      connection:
        secret: gitlab-object-storage
        key: connection
    
    backups:
      bucket: gitlab-backups
      tmpBucket: gitlab-tmp

# Container Registry with MinIO
registry:
  storage:
    secret: gitlab-registry-storage
    key: config

# Don't install bundled MinIO
minio:
  install: false
```

## GitLab Runner Cache Configuration

```yaml
# In gitlab-runner values.yaml
runners:
  cache:
    cacheType: s3
    s3BucketName: gitlab-runner-cache
    s3BucketLocation: us-east-1
    s3ServerAddress: minio.minio.svc.cluster.local:9000
    s3CachePath: runner-cache
    s3CacheInsecure: true  # Internal cluster, no TLS needed
    secretName: gitlab-runner-cache-secret
```

## Verify Integration

```bash
# Check secrets exist
kubectl get secret gitlab-object-storage -n gitlab
kubectl get secret gitlab-registry-storage -n gitlab

# Test connectivity from GitLab
kubectl exec -n gitlab -it $(kubectl get pod -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://minio.minio.svc.cluster.local:9000/minio/health/live

# Check registry can access storage
kubectl logs -n gitlab -l app=registry --tail=20
```