# External Secrets Operator

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| External Secrets Operator | 1.2.0 |
| Helm Chart | 0.15.0 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-eso.sh

set -euo pipefail

ESO_VERSION="0.15.0"

echo "=== Installing External Secrets Operator ==="

helm repo add external-secrets https://charts.external-secrets.io
helm repo update

helm upgrade --install external-secrets external-secrets/external-secrets \
  --namespace external-secrets \
  --create-namespace \
  --version ${ESO_VERSION} \
  --set installCRDs=true \
  --set webhook.port=9443 \
  --set certController.requeueInterval=5m \
  --set serviceMonitor.enabled=true \
  --wait

kubectl wait --for=condition=Available deployment/external-secrets \
  -n external-secrets --timeout=120s

echo "=== ESO Installed ==="
kubectl get pods -n external-secrets
```

## ClusterSecretStore

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
```

## ExternalSecret Example

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: myapp
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: database-credentials
    template:
      data:
        DATABASE_URL: "postgresql://{{ .username }}:{{ .password }}@postgres:5432/myapp"
  data:
    - secretKey: username
      remoteRef:
        key: secret/data/myapp/database
        property: username
    - secretKey: password
      remoteRef:
        key: secret/data/myapp/database
        property: password
```