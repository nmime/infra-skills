# GitLab Cleanup Scripts

## Complete GitLab Uninstall

```bash
#!/bin/bash
# scripts/cleanup-gitlab.sh
# WARNING: This will delete ALL GitLab data!

set -euo pipefail

GITLAB_NAMESPACE="gitlab"
RUNNER_NAMESPACE="gitlab-runner"

echo "⚠️  WARNING: This will completely remove GitLab and all data!"
echo "Namespaces to delete: ${GITLAB_NAMESPACE}, ${RUNNER_NAMESPACE}"
read -p "Type 'DELETE' to confirm: " confirm

if [[ "$confirm" != "DELETE" ]]; then
  echo "Aborted."
  exit 1
fi

echo ""
echo "=== Uninstalling GitLab Runner ==="
helm uninstall gitlab-runner -n ${RUNNER_NAMESPACE} 2>/dev/null || echo "Runner not found"

echo ""
echo "=== Uninstalling GitLab ==="
helm uninstall gitlab -n ${GITLAB_NAMESPACE} 2>/dev/null || echo "GitLab not found"

echo ""
echo "=== Deleting PVCs ==="
kubectl delete pvc --all -n ${GITLAB_NAMESPACE} --wait=false 2>/dev/null || true
kubectl delete pvc --all -n ${RUNNER_NAMESPACE} --wait=false 2>/dev/null || true

echo ""
echo "=== Deleting Secrets ==="
kubectl delete secret --all -n ${GITLAB_NAMESPACE} 2>/dev/null || true
kubectl delete secret --all -n ${RUNNER_NAMESPACE} 2>/dev/null || true

echo ""
echo "=== Deleting ConfigMaps ==="
kubectl delete configmap --all -n ${GITLAB_NAMESPACE} 2>/dev/null || true

echo ""
echo "=== Deleting Jobs ==="
kubectl delete jobs --all -n ${GITLAB_NAMESPACE} 2>/dev/null || true

echo ""
echo "=== Waiting for pods to terminate ==="
kubectl wait --for=delete pod --all -n ${GITLAB_NAMESPACE} --timeout=300s 2>/dev/null || true

echo ""
echo "=== Deleting Namespaces ==="
kubectl delete namespace ${GITLAB_NAMESPACE} --wait=false 2>/dev/null || true
kubectl delete namespace ${RUNNER_NAMESPACE} --wait=false 2>/dev/null || true

echo ""
echo "=== Cleaning up Gateway API Routes ==="
kubectl delete httproute -n ${GITLAB_NAMESPACE} --all 2>/dev/null || true
kubectl delete httproute gitlab-webservice gitlab-registry gitlab-kas -n ${GITLAB_NAMESPACE} 2>/dev/null || true

echo ""
echo "=== Cleaning up Certificates ==="
kubectl delete certificate gitlab-tls gitlab-registry-tls -n cilium-secrets 2>/dev/null || true

echo ""
echo "=== Verifying cleanup ==="
echo "Remaining resources in ${GITLAB_NAMESPACE}:"
kubectl get all -n ${GITLAB_NAMESPACE} 2>/dev/null || echo "Namespace deleted"

echo ""
echo "✅ GitLab cleanup complete!"
echo ""
echo "Note: S3 buckets and external PostgreSQL data are NOT deleted."
echo "To delete those, run:"
echo "  - Delete S3 buckets manually"
echo "  - kubectl delete perconapgcluster gitlab-pg -n databases"
```

## Cleanup GitLab Runner Only

```bash
#!/bin/bash
# scripts/cleanup-gitlab-runner.sh

set -euo pipefail

RUNNER_NAMESPACE="gitlab-runner"

echo "=== Uninstalling GitLab Runner ==="

# Stop all running jobs first
kubectl delete pods -n ${RUNNER_NAMESPACE} -l app=gitlab-runner-gitlab-runner --force --grace-period=0 2>/dev/null || true

# Uninstall helm release
helm uninstall gitlab-runner -n ${RUNNER_NAMESPACE} 2>/dev/null || echo "Not found"

# Delete remaining resources
kubectl delete pvc --all -n ${RUNNER_NAMESPACE} 2>/dev/null || true
kubectl delete secret gitlab-runner-cache-secret -n ${RUNNER_NAMESPACE} 2>/dev/null || true
kubectl delete secret gitlab-runner-secret -n ${RUNNER_NAMESPACE} 2>/dev/null || true

# Optionally delete namespace
read -p "Delete namespace ${RUNNER_NAMESPACE}? (y/n): " delete_ns
if [[ "$delete_ns" == "y" ]]; then
  kubectl delete namespace ${RUNNER_NAMESPACE}
fi

echo "✅ GitLab Runner cleanup complete!"
```

## Cleanup Registry Data (Garbage Collection)

```bash
#!/bin/bash
# scripts/cleanup-registry-gc.sh
# Run registry garbage collection to reclaim space

set -euo pipefail

GITLAB_NAMESPACE="gitlab"

echo "=== Running Registry Garbage Collection ==="

# Find the registry pod
REGISTRY_POD=$(kubectl get pods -n ${GITLAB_NAMESPACE} -l app=registry -o jsonpath='{.items[0].metadata.name}')

if [[ -z "$REGISTRY_POD" ]]; then
  echo "Registry pod not found!"
  exit 1
fi

echo "Registry pod: ${REGISTRY_POD}"

# Run garbage collection (dry-run first)
echo ""
echo "=== Dry run (showing what would be deleted) ==="
kubectl exec -n ${GITLAB_NAMESPACE} ${REGISTRY_POD} -- \
  /bin/registry garbage-collect /etc/docker/registry/config.yml --dry-run

read -p "Proceed with actual garbage collection? (y/n): " proceed
if [[ "$proceed" != "y" ]]; then
  echo "Aborted."
  exit 0
fi

echo ""
echo "=== Running garbage collection ==="
kubectl exec -n ${GITLAB_NAMESPACE} ${REGISTRY_POD} -- \
  /bin/registry garbage-collect /etc/docker/registry/config.yml --delete-untagged

echo "✅ Registry garbage collection complete!"
```

## Cleanup Old CI/CD Artifacts

```bash
#!/bin/bash
# scripts/cleanup-artifacts.sh
# Clean up old CI/CD artifacts from S3

set -euo pipefail

echo "=== Cleaning up old CI/CD artifacts ==="

# This triggers GitLab's built-in cleanup
TOOLBOX_POD=$(kubectl get pods -n gitlab -l app=toolbox -o jsonpath='{.items[0].metadata.name}')

if [[ -z "$TOOLBOX_POD" ]]; then
  echo "Toolbox pod not found!"
  exit 1
fi

# Run artifact cleanup via rails console
kubectl exec -n gitlab ${TOOLBOX_POD} -- \
  gitlab-rails runner "Ci::JobArtifact.expired.find_each { |a| a.destroy }"

# Run LFS cleanup
kubectl exec -n gitlab ${TOOLBOX_POD} -- \
  gitlab-rails runner "LfsObject.unreferenced.find_each { |o| o.destroy }"

echo "✅ Artifact cleanup complete!"
```

## Cleanup Stuck/Failed Jobs

```bash
#!/bin/bash
# scripts/cleanup-stuck-jobs.sh

set -euo pipefail

RUNNER_NAMESPACE="gitlab-runner"

echo "=== Cleaning up stuck runner pods ==="

# Delete pods stuck in Terminating
kubectl get pods -n ${RUNNER_NAMESPACE} --field-selector=status.phase=Terminating -o name | \
  xargs -r kubectl delete -n ${RUNNER_NAMESPACE} --force --grace-period=0

# Delete completed/failed pods older than 1 hour
kubectl get pods -n ${RUNNER_NAMESPACE} -o json | \
  jq -r '.items[] | select(.status.phase=="Succeeded" or .status.phase=="Failed") | .metadata.name' | \
  xargs -r kubectl delete pod -n ${RUNNER_NAMESPACE}

# Delete evicted pods
kubectl get pods -n ${RUNNER_NAMESPACE} --field-selector=status.phase=Failed -o name | \
  xargs -r kubectl delete -n ${RUNNER_NAMESPACE}

echo "✅ Stuck jobs cleanup complete!"
echo ""
kubectl get pods -n ${RUNNER_NAMESPACE}
```

## Reset GitLab to Fresh State

```bash
#!/bin/bash
# scripts/reset-gitlab.sh
# Reset GitLab while keeping infrastructure

set -euo pipefail

GITLAB_NAMESPACE="gitlab"

echo "⚠️  This will reset GitLab to fresh state (keeps PG, S3)"
read -p "Type 'RESET' to confirm: " confirm

if [[ "$confirm" != "RESET" ]]; then
  echo "Aborted."
  exit 1
fi

# Scale down
echo "=== Scaling down GitLab ==="
kubectl scale deployment -n ${GITLAB_NAMESPACE} --all --replicas=0
kubectl scale statefulset -n ${GITLAB_NAMESPACE} --all --replicas=0

# Wait
sleep 30

# Reset database (via toolbox)
echo "=== This requires manual database reset ==="
echo "Connect to PostgreSQL and run:"
echo "  DROP DATABASE gitlabhq_production;"
echo "  CREATE DATABASE gitlabhq_production;"
echo ""
read -p "Press Enter after database reset..."

# Delete secrets (will be regenerated)
kubectl delete secret -n ${GITLAB_NAMESPACE} -l app=gitlab --all

# Restart
echo "=== Restarting GitLab ==="
helm upgrade gitlab gitlab/gitlab -n ${GITLAB_NAMESPACE} --reuse-values

echo "✅ GitLab reset initiated. Check pods:"
kubectl get pods -n ${GITLAB_NAMESPACE} -w
```

## Scheduled Cleanup CronJob

```yaml
# cleanup-cronjob.yaml
# Deploy for automatic cleanup
apiVersion: batch/v1
kind: CronJob
metadata:
  name: gitlab-cleanup
  namespace: gitlab
spec:
  schedule: "0 3 * * 0"  # Weekly on Sunday at 3 AM
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: gitlab-toolbox
          containers:
            - name: cleanup
              image: registry.gitlab.com/gitlab-org/build/cng/gitlab-toolbox-ee:v18.7.1
              command:
                - /bin/sh
                - -c
                - |
                  # Clean expired artifacts
                  gitlab-rails runner "Ci::JobArtifact.expired.find_each { |a| a.destroy }"
                  
                  # Clean orphaned LFS objects
                  gitlab-rails runner "LfsObject.unreferenced.find_each { |o| o.destroy }"
                  
                  # Clean old job logs
                  gitlab-rails runner "Ci::Build.where('created_at < ?', 90.days.ago).find_each { |b| b.erase }"
                  
                  echo "Cleanup complete"
              envFrom:
                - configMapRef:
                    name: gitlab-toolbox
                - secretRef:
                    name: gitlab-rails-secret
          restartPolicy: OnFailure
```