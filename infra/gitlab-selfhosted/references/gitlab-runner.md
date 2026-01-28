# GitLab Runner on Kubernetes

## Version Information (Latest - January 2025)

| Component | Version |
|-----------|---------|
| GitLab Runner | 18.7.2 |
| Helm Chart | 0.84.2 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-gitlab-runner.sh

set -euo pipefail

RUNNER_CHART_VERSION="0.84.2"
RUNNER_NAMESPACE="gitlab-runner"
GITLAB_URL="${1:-https://gitlab.example.com}"
RUNNER_TOKEN="${2:-}"

if [[ -z "$RUNNER_TOKEN" ]]; then
  echo "Usage: $0 <GITLAB_URL> <RUNNER_TOKEN>"
  echo ""
  echo "Get runner token from:"
  echo "  GitLab Admin > CI/CD > Runners > New instance runner"
  echo "  OR"
  echo "  Project > Settings > CI/CD > Runners > New project runner"
  exit 1
fi

echo "============================================"
echo "GitLab Runner Installation"
echo "============================================"
echo "Chart Version: ${RUNNER_CHART_VERSION}"
echo "GitLab URL: ${GITLAB_URL}"
echo "============================================"

# Add Helm repo
helm repo add gitlab https://charts.gitlab.io/
helm repo update

# Create namespace
kubectl create namespace ${RUNNER_NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -

# Create runner values
cat > /tmp/runner-values.yaml << EOF
#=============================================
# GitLab Runner Helm Values - Production
#=============================================

# GitLab connection
gitlabUrl: ${GITLAB_URL}
runnerToken: ${RUNNER_TOKEN}

# Concurrent jobs
concurrent: 20

# Check interval
checkInterval: 3

# Log level
logLevel: info

# RBAC
rbac:
  create: true
  rules:
    - apiGroups: [""]
      resources: ["pods", "pods/exec", "secrets", "configmaps"]
      verbs: ["get", "list", "watch", "create", "delete", "update"]
    - apiGroups: [""]
      resources: ["pods/log"]
      verbs: ["get"]

# Service account
serviceAccount:
  create: true
  name: gitlab-runner

# Metrics
metrics:
  enabled: true
  portName: metrics
  port: 9252
  serviceMonitor:
    enabled: true

# Runner replicas (manager pods)
replicas: 2

# Runner manager resources
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Pod disruption budget
podDisruptionBudget:
  minAvailable: 1

# Affinity
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: gitlab-runner
          topologyKey: kubernetes.io/hostname

# Runner configuration
runners:
  # Runner name
  name: "k8s-runner"
  
  # Tags
  tags: "kubernetes,docker,k8s"
  
  # Run untagged jobs
  runUntagged: true
  
  # Protected branches
  protected: false
  
  # Lock to project
  locked: false
  
  # Privileged (required for Docker-in-Docker)
  privileged: true
  
  # Namespace for job pods
  namespace: ${RUNNER_NAMESPACE}
  
  # Service account for jobs
  serviceAccountName: gitlab-runner
  
  # Pod labels
  podLabels:
    gitlab-runner: "job"
  
  # Pod annotations
  podAnnotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9252"
  
  # Poll timeout
  pollTimeout: 3600
  
  # Output limit
  outputLimit: 16384
  
  # Build container resources
  builds:
    cpuLimit: 2000m
    cpuLimitOverwriteMaxAllowed: 4000m
    memoryLimit: 4Gi
    memoryLimitOverwriteMaxAllowed: 8Gi
    cpuRequests: 500m
    cpuRequestsOverwriteMaxAllowed: 2000m
    memoryRequests: 1Gi
    memoryRequestsOverwriteMaxAllowed: 4Gi
  
  # Service container resources
  services:
    cpuLimit: 1000m
    memoryLimit: 2Gi
    cpuRequests: 200m
    memoryRequests: 512Mi
  
  # Helper container resources
  helpers:
    cpuLimit: 500m
    memoryLimit: 512Mi
    cpuRequests: 100m
    memoryRequests: 128Mi
  
  # Cache configuration
  cache:
    cacheType: s3
    s3BucketName: gitlab-runner-cache
    s3BucketLocation: eu-central-1
    s3ServerAddress: s3.amazonaws.com
    secretName: gitlab-runner-cache-secret
    cacheShared: true
  
  # Config template
  config: |
    [[runners]]
      name = "Kubernetes Runner"
      executor = "kubernetes"
      
      [runners.kubernetes]
        namespace = "${RUNNER_NAMESPACE}"
        image = "alpine:3.19"
        privileged = true
        
        # Pull policy
        pull_policy = ["if-not-present", "always"]
        
        # Pod cleanup
        poll_interval = 3
        poll_timeout = 3600
        
        # Resource limits
        cpu_request = "500m"
        cpu_limit = "2000m"
        memory_request = "1Gi"
        memory_limit = "4Gi"
        
        # Service account
        service_account = "gitlab-runner"
        
        # Node selector (optional)
        # [runners.kubernetes.node_selector]
        #   workload = "ci"
        
        # Tolerations (optional)
        # [[runners.kubernetes.node_tolerations]]
        #   key = "ci-only"
        #   operator = "Equal"
        #   value = "true"
        #   effect = "NoSchedule"
        
        # Pod security context
        [runners.kubernetes.pod_security_context]
          run_as_non_root = false
          run_as_user = 0
        
        # Volumes for Docker-in-Docker
        [[runners.kubernetes.volumes.empty_dir]]
          name = "docker-certs"
          mount_path = "/certs/client"
          medium = "Memory"
        
        [[runners.kubernetes.volumes.empty_dir]]
          name = "docker-graph-storage"
          mount_path = "/var/lib/docker"
      
      [runners.cache]
        Type = "s3"
        Shared = true
        [runners.cache.s3]
          ServerAddress = "s3.amazonaws.com"
          BucketName = "gitlab-runner-cache"
          BucketLocation = "eu-central-1"
EOF

echo ""
echo "=== Creating Cache Secret ==="
if ! kubectl get secret gitlab-runner-cache-secret -n ${RUNNER_NAMESPACE} &>/dev/null; then
  read -p "S3 Access Key for runner cache: " CACHE_ACCESS_KEY
  read -sp "S3 Secret Key for runner cache: " CACHE_SECRET_KEY
  echo ""
  
  kubectl create secret generic gitlab-runner-cache-secret \
    --namespace ${RUNNER_NAMESPACE} \
    --from-literal=accesskey="${CACHE_ACCESS_KEY}" \
    --from-literal=secretkey="${CACHE_SECRET_KEY}"
fi

echo ""
echo "=== Installing GitLab Runner ==="

helm upgrade --install gitlab-runner gitlab/gitlab-runner \
  --namespace ${RUNNER_NAMESPACE} \
  --version ${RUNNER_CHART_VERSION} \
  --values /tmp/runner-values.yaml \
  --wait

echo ""
echo "============================================"
echo "GitLab Runner Installation Complete!"
echo "============================================"
echo ""
kubectl get pods -n ${RUNNER_NAMESPACE}
echo ""
echo "Verify runner in GitLab:"
echo "  Admin > CI/CD > Runners"
echo "  Should show runner as 'online'"
```

## Kaniko Build (No Privileged Mode)

```yaml
# .gitlab-ci.yml - Using Kaniko instead of Docker-in-Docker
variables:
  CI_REGISTRY_IMAGE: registry.example.com/myorg/myapp

stages:
  - build
  - deploy

build:image:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:v1.23.2-debug
    entrypoint: [""]
  script:
    - |
      /kaniko/executor \
        --context "${CI_PROJECT_DIR}" \
        --dockerfile "${CI_PROJECT_DIR}/Dockerfile" \
        --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_SHA}" \
        --destination "${CI_REGISTRY_IMAGE}:${CI_COMMIT_REF_SLUG}" \
        --cache=true \
        --cache-repo="${CI_REGISTRY_IMAGE}/cache"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'
```

## Docker-in-Docker Build

```yaml
# .gitlab-ci.yml - Using Docker-in-Docker
variables:
  DOCKER_TLS_CERTDIR: "/certs"
  DOCKER_HOST: tcp://docker:2376
  DOCKER_DRIVER: overlay2

build:docker:
  stage: build
  image: docker:26
  services:
    - docker:26-dind
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Verify Runner

```bash
# Check runner pods
kubectl get pods -n gitlab-runner

# Check logs
kubectl logs -n gitlab-runner -l app=gitlab-runner -f

# Check runner registration
kubectl exec -it -n gitlab-runner $(kubectl get pods -n gitlab-runner -l app=gitlab-runner -o jsonpath='{.items[0].metadata.name}') -- gitlab-runner list

# Verify in GitLab UI
# Admin > CI/CD > Runners > Should show runner online
```