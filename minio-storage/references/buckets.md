# MinIO Bucket Configuration

## Required Buckets

| Bucket | Purpose | Retention | Versioning |
|--------|---------|-----------|------------|
| `gitlab-registry` | Container images | 90 days untagged | No |
| `gitlab-artifacts` | CI/CD artifacts | 30 days | No |
| `gitlab-lfs` | Git LFS objects | Permanent | Yes |
| `gitlab-uploads` | User uploads | Permanent | Yes |
| `gitlab-packages` | Package registry | Permanent | Yes |
| `gitlab-backups` | GitLab backups | 30 days | Yes |
| `gitlab-terraform-state` | TF state | Permanent | Yes |
| `gitlab-ci-secure-files` | Secure files | Permanent | Yes |
| `loki-chunks` | Loki logs | 14 days | No |
| `loki-ruler` | Loki rules | Permanent | No |
| `postgres-backups` | PostgreSQL backups | 30 days | Yes |
| `mongodb-backups` | MongoDB backups | 30 days | Yes |
| `velero-backups` | Cluster backups | 30 days | Yes |

## Create Buckets Script

```bash
#!/bin/bash
# scripts/create-buckets.sh

set -euo pipefail

MINIO_NAMESPACE="minio"

echo "=== Creating MinIO Buckets ==="

# Get credentials
ROOT_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
ROOT_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

# Create job to run mc commands
cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: minio-bucket-setup
  namespace: ${MINIO_NAMESPACE}
spec:
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: mc
          image: minio/mc:latest
          command:
            - /bin/sh
            - -c
            - |
              set -e
              
              # Configure mc
              mc alias set myminio http://minio.${MINIO_NAMESPACE}.svc.cluster.local:9000 \${ROOT_USER} \${ROOT_PASSWORD}
              
              echo "=== Creating GitLab buckets ==="
              mc mb --ignore-existing myminio/gitlab-registry
              mc mb --ignore-existing myminio/gitlab-artifacts
              mc mb --ignore-existing myminio/gitlab-lfs
              mc mb --ignore-existing myminio/gitlab-uploads
              mc mb --ignore-existing myminio/gitlab-packages
              mc mb --ignore-existing myminio/gitlab-backups
              mc mb --ignore-existing myminio/gitlab-terraform-state
              mc mb --ignore-existing myminio/gitlab-ci-secure-files
              mc mb --ignore-existing myminio/gitlab-pages
              mc mb --ignore-existing myminio/gitlab-tmp
              mc mb --ignore-existing myminio/gitlab-dependency-proxy
              
              echo "=== Creating Loki buckets ==="
              mc mb --ignore-existing myminio/loki-chunks
              mc mb --ignore-existing myminio/loki-ruler
              mc mb --ignore-existing myminio/loki-admin
              
              echo "=== Creating backup buckets ==="
              mc mb --ignore-existing myminio/postgres-backups
              mc mb --ignore-existing myminio/mongodb-backups
              mc mb --ignore-existing myminio/velero-backups
              mc mb --ignore-existing myminio/gitlab-runner-cache
              
              echo "=== Setting versioning ==="
              mc version enable myminio/gitlab-lfs
              mc version enable myminio/gitlab-uploads
              mc version enable myminio/gitlab-packages
              mc version enable myminio/gitlab-backups
              mc version enable myminio/gitlab-terraform-state
              mc version enable myminio/postgres-backups
              mc version enable myminio/mongodb-backups
              mc version enable myminio/velero-backups
              
              echo "=== Setting lifecycle policies ==="
              
              # GitLab artifacts - 30 days
              mc ilm rule add --expire-days 30 myminio/gitlab-artifacts
              
              # GitLab tmp - 7 days
              mc ilm rule add --expire-days 7 myminio/gitlab-tmp
              
              # Loki chunks - 14 days
              mc ilm rule add --expire-days 14 myminio/loki-chunks
              
              # Backups - 30 days for old versions
              mc ilm rule add --noncurrent-expire-days 30 myminio/gitlab-backups
              mc ilm rule add --noncurrent-expire-days 30 myminio/postgres-backups
              mc ilm rule add --noncurrent-expire-days 30 myminio/mongodb-backups
              mc ilm rule add --noncurrent-expire-days 30 myminio/velero-backups
              
              echo "=== Bucket setup complete ==="
              mc ls myminio/
          env:
            - name: ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-credentials
                  key: rootUser
            - name: ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-credentials
                  key: rootPassword
EOF

echo "Waiting for bucket setup job..."
kubectl wait --for=condition=complete job/minio-bucket-setup -n ${MINIO_NAMESPACE} --timeout=120s

echo ""
echo "=== Bucket Setup Complete ==="
kubectl logs -n ${MINIO_NAMESPACE} job/minio-bucket-setup
```

## Create Service Accounts

```bash
#!/bin/bash
# scripts/create-service-accounts.sh

set -euo pipefail

MINIO_NAMESPACE="minio"

ROOT_USER=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootUser}' | base64 -d)
ROOT_PASSWORD=$(kubectl get secret minio-credentials -n ${MINIO_NAMESPACE} -o jsonpath='{.data.rootPassword}' | base64 -d)

cat <<EOF | kubectl apply -f -
apiVersion: batch/v1
kind: Job
metadata:
  name: minio-create-users
  namespace: ${MINIO_NAMESPACE}
spec:
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: mc
          image: minio/mc:latest
          command:
            - /bin/sh
            - -c
            - |
              set -e
              mc alias set myminio http://minio.${MINIO_NAMESPACE}.svc.cluster.local:9000 \${ROOT_USER} \${ROOT_PASSWORD}
              
              # Create GitLab user
              GITLAB_KEY=\$(openssl rand -hex 16)
              GITLAB_SECRET=\$(openssl rand -hex 32)
              mc admin user add myminio gitlab-user \${GITLAB_SECRET}
              
              # Create GitLab policy
              cat > /tmp/gitlab-policy.json << 'POLICY'
              {
                "Version": "2012-10-17",
                "Statement": [
                  {
                    "Effect": "Allow",
                    "Action": ["s3:*"],
                    "Resource": [
                      "arn:aws:s3:::gitlab-*",
                      "arn:aws:s3:::gitlab-*/*"
                    ]
                  }
                ]
              }
              POLICY
              mc admin policy create myminio gitlab-policy /tmp/gitlab-policy.json
              mc admin policy attach myminio gitlab-policy --user gitlab-user
              
              # Create Loki user
              LOKI_SECRET=\$(openssl rand -hex 32)
              mc admin user add myminio loki-user \${LOKI_SECRET}
              
              cat > /tmp/loki-policy.json << 'POLICY'
              {
                "Version": "2012-10-17",
                "Statement": [
                  {
                    "Effect": "Allow",
                    "Action": ["s3:*"],
                    "Resource": [
                      "arn:aws:s3:::loki-*",
                      "arn:aws:s3:::loki-*/*"
                    ]
                  }
                ]
              }
              POLICY
              mc admin policy create myminio loki-policy /tmp/loki-policy.json
              mc admin policy attach myminio loki-policy --user loki-user
              
              # Create backup user
              BACKUP_SECRET=\$(openssl rand -hex 32)
              mc admin user add myminio backup-user \${BACKUP_SECRET}
              
              cat > /tmp/backup-policy.json << 'POLICY'
              {
                "Version": "2012-10-17",
                "Statement": [
                  {
                    "Effect": "Allow",
                    "Action": ["s3:*"],
                    "Resource": [
                      "arn:aws:s3:::*-backups",
                      "arn:aws:s3:::*-backups/*",
                      "arn:aws:s3:::velero-*",
                      "arn:aws:s3:::velero-*/*"
                    ]
                  }
                ]
              }
              POLICY
              mc admin policy create myminio backup-policy /tmp/backup-policy.json
              mc admin policy attach myminio backup-policy --user backup-user
              
              echo "Users created. Get credentials from MinIO console."
          env:
            - name: ROOT_USER
              valueFrom:
                secretKeyRef:
                  name: minio-credentials
                  key: rootUser
            - name: ROOT_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: minio-credentials
                  key: rootPassword
EOF

kubectl wait --for=condition=complete job/minio-create-users -n ${MINIO_NAMESPACE} --timeout=120s
echo "Users created! Get access keys from MinIO console."
```