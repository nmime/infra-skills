# HashiCorp Vault Installation

All scripts are **idempotent** - safe to run multiple times. Uses `helm upgrade --install` for convergent behavior.

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| Vault | 1.21.2 |
| Vault Helm Chart | 0.29.0 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-vault.sh

set -euo pipefail

VAULT_CHART_VERSION="0.29.0"
VAULT_NAMESPACE="vault"

echo "=== Installing HashiCorp Vault ==="

helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update

kubectl create namespace $VAULT_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install vault hashicorp/vault \
  --namespace $VAULT_NAMESPACE \
  --version $VAULT_CHART_VERSION \
  --set server.ha.enabled=true \
  --set server.ha.replicas=3 \
  --set server.ha.raft.enabled=true \
  --set server.ha.raft.setNodeId=true \
  --set server.dataStorage.enabled=true \
  --set server.dataStorage.storageClass=hcloud-volumes \
  --set server.dataStorage.size=10Gi \
  --set server.auditStorage.enabled=true \
  --set server.auditStorage.storageClass=hcloud-volumes \
  --set server.auditStorage.size=10Gi \
  --set injector.enabled=true \
  --set ui.enabled=true \
  --wait

echo "=== Vault pods starting ==="
kubectl get pods -n $VAULT_NAMESPACE
echo ""
echo "Next: Initialize with 'kubectl exec -it vault-0 -n vault -- vault operator init'"
```