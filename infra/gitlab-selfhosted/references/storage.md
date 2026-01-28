# GitLab Storage Configuration

## Recommended: MinIO on Kubernetes

For self-hosted setups, use MinIO from the `minio-storage` skill.

### Prerequisites

1. Install MinIO first:
```bash
# From minio-storage skill
./scripts/install-minio.sh hcloud-volumes 100Gi 4
./scripts/create-buckets.sh
```

2. Create GitLab secrets:
```bash
./scripts/create-gitlab-minio-secrets.sh
```

### Integration Steps

```bash
#!/bin/bash
# scripts/create-gitlab-minio-secrets.sh

set -euo pipefail

GITLAB_NAMESPACE="gitlab"
MINIO_NAMESPACE="minio"

MINIO_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
MINIO_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)
MINIO_ENDPOINT="http://minio.${MINIO_NAMESPACE}.svc.cluster.local:9000"

kubectl create namespace ${GITLAB_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Object storage secret
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

# Registry storage secret
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
EOF

echo "GitLab MinIO secrets created!"
```

## GitLab Helm Values for MinIO

```yaml
# Add to gitlab-values.yaml
global:
  appConfig:
    object_store:
      enabled: true
      proxy_download: true
      connection:
        secret: gitlab-object-storage
        key: connection
    
    artifacts:
      bucket: gitlab-artifacts
    lfs:
      bucket: gitlab-lfs
    uploads:
      bucket: gitlab-uploads
    packages:
      bucket: gitlab-packages
    terraformState:
      bucket: gitlab-terraform-state
    ciSecureFiles:
      bucket: gitlab-ci-secure-files
    dependencyProxy:
      bucket: gitlab-dependency-proxy
    backups:
      bucket: gitlab-backups
      tmpBucket: gitlab-tmp

registry:
  storage:
    secret: gitlab-registry-storage
    key: config

minio:
  install: false  # Using external MinIO
```

## Required MinIO Buckets

| Bucket | Purpose |
|--------|--------|
| `gitlab-registry` | Container images |
| `gitlab-artifacts` | CI/CD artifacts |
| `gitlab-lfs` | Git LFS objects |
| `gitlab-uploads` | User uploads |
| `gitlab-packages` | Package registry |
| `gitlab-backups` | Backups |
| `gitlab-terraform-state` | TF state |
| `gitlab-ci-secure-files` | Secure files |
| `gitlab-tmp` | Temporary files |
| `gitlab-dependency-proxy` | Dependency proxy |
| `gitlab-runner-cache` | Runner cache |

## Alternative: Cloud S3

### AWS S3

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-object-storage
  namespace: gitlab
stringData:
  connection: |
    provider: AWS
    region: eu-central-1
    aws_access_key_id: "AKIAXXXXXXXX"
    aws_secret_access_key: "xxxxxxxxxx"
```

### Cloudflare R2

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-object-storage
  namespace: gitlab
stringData:
  connection: |
    provider: AWS
    region: auto
    aws_access_key_id: "R2_ACCESS_KEY"
    aws_secret_access_key: "R2_SECRET_KEY"
    endpoint: "https://ACCOUNT_ID.r2.cloudflarestorage.com"
    path_style: true
```

## Verify Storage

```bash
# Test from GitLab toolbox
kubectl exec -n gitlab -it $(kubectl get pod -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}') -- \
  curl -s http://minio.minio.svc.cluster.local:9000/minio/health/live

# Check registry logs
kubectl logs -n gitlab -l app=registry --tail=20
```