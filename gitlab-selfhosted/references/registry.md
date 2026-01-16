# GitLab Container Registry

## Overview

GitLab Container Registry is included in the GitLab Helm chart and provides:
- Docker/OCI image storage
- Integration with GitLab CI/CD
- Image vulnerability scanning (Ultimate)
- Cleanup policies
- Geo replication (Premium)

## Registry Configuration

The registry is configured in `gitlab-values.yaml`:

```yaml
global:
  registry:
    enabled: true
    bucket: gitlab-registry
    
  hosts:
    registry:
      name: registry.example.com
      https: true

registry:
  enabled: true
  replicaCount: 2
  
  # HPA configuration
  hpa:
    minReplicas: 2
    maxReplicas: 5
    cpu:
      targetAverageUtilization: 75
  
  # Storage backend (S3)
  storage:
    secret: gitlab-registry-storage
    key: config
  
  # Garbage collection
  maintenance:
    gc:
      disabled: false
      schedule: "0 4 * * 0"  # Weekly Sunday 4 AM
  
  # Debug mode
  debug:
    addr:
      port: 5001
    prometheus:
      enabled: true
      path: /metrics
  
  # Resources
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 256Mi
```

## Registry Storage Secret

```yaml
# gitlab-registry-storage.yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-registry-storage
  namespace: gitlab
type: Opaque
stringData:
  config: |
    s3:
      bucket: gitlab-registry
      accesskey: "YOUR_ACCESS_KEY"
      secretkey: "YOUR_SECRET_KEY"
      region: eu-central-1
      regionendpoint: "https://s3.amazonaws.com"
      v4auth: true
      pathstyle: true
      rootdirectory: /registry
      # Optional: encryption
      # encrypt: true
      # keyid: "alias/my-key"
    # Maintenance mode (optional)
    maintenance:
      uploadpurging:
        enabled: true
        age: 168h  # 7 days
        interval: 24h
        dryrun: false
    # Cache configuration
    cache:
      blobdescriptor: inmemory
    # Delete enabled
    delete:
      enabled: true
    # Redirect (optional, for CDN)
    # redirect:
    #   disable: false
```

## Using the Registry

### Login to Registry

```bash
# Using Docker
docker login registry.example.com
# Username: your-gitlab-username
# Password: your-personal-access-token (with read_registry, write_registry scope)

# Using personal access token directly
echo $GITLAB_TOKEN | docker login registry.example.com -u oauth2 --password-stdin
```

### Push Images

```bash
# Tag image
docker tag myapp:latest registry.example.com/mygroup/myproject/myapp:latest

# Push
docker push registry.example.com/mygroup/myproject/myapp:latest
```

### Pull Images

```bash
docker pull registry.example.com/mygroup/myproject/myapp:latest
```

### CI/CD Integration

```yaml
# .gitlab-ci.yml
variables:
  CI_REGISTRY: registry.example.com
  CI_REGISTRY_IMAGE: $CI_REGISTRY/$CI_PROJECT_PATH

build:
  stage: build
  image: docker:26
  services:
    - docker:26-dind
  before_script:
    # Auto-login using CI job token
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
```

## Cleanup Policies

### Project-Level Cleanup (GitLab UI)

1. Go to Project > Settings > Packages & Registries > Container Registry
2. Set cleanup rules:
   - Keep most recent: 5 tags
   - Remove tags older than: 90 days
   - Remove tags matching: `.*-dev`, `.*-test`

### Garbage Collection

```bash
#!/bin/bash
# scripts/registry-gc.sh

REGISTRY_POD=$(kubectl get pods -n gitlab -l app=registry -o jsonpath='{.items[0].metadata.name}')

# Dry run first
echo "=== Dry Run ==="
kubectl exec -n gitlab ${REGISTRY_POD} -- \
  /bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run

read -p "Proceed with garbage collection? (y/n): " proceed
if [[ "$proceed" == "y" ]]; then
  echo "=== Running GC ==="
  kubectl exec -n gitlab ${REGISTRY_POD} -- \
    /bin/registry garbage-collect /etc/docker/registry/config.yml --delete-untagged
fi
```

## Registry Monitoring

### Prometheus Metrics

```yaml
# VMServiceScrape for registry
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: gitlab-registry
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: registry
  namespaceSelector:
    matchNames:
      - gitlab
  endpoints:
    - port: http-metrics
      path: /metrics
```

### Key Metrics

| Metric | Description |
|--------|-------------|
| `registry_storage_action_seconds` | Storage operation latency |
| `registry_http_requests_total` | Total HTTP requests |
| `registry_http_in_flight_requests` | Current requests |
| `registry_storage_blob_upload_bytes` | Upload bytes |

## Troubleshooting

```bash
# Check registry pods
kubectl get pods -n gitlab -l app=registry

# Check registry logs
kubectl logs -n gitlab -l app=registry -f

# Test registry health
curl -k https://registry.example.com/v2/
# Should return: {}

# Check storage connectivity
kubectl exec -n gitlab -l app=registry -- \
  /bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run

# Debug mode
kubectl port-forward -n gitlab svc/gitlab-registry 5001:5001
curl http://localhost:5001/debug/health
```