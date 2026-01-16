# ArgoCD Installation

All scripts are **idempotent** - safe to run multiple times. Uses `kubectl apply` and `helm upgrade --install` for convergent behavior.

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| ArgoCD | v3.2.5 |
| ArgoCD CLI | v3.2.5 |
| Image Updater | v0.15.1 |

## Installation Script

```bash
#!/bin/bash
# scripts/install-argocd.sh

set -euo pipefail

ARGOCD_VERSION="v3.2.5"
ARGOCD_NAMESPACE="argocd"

echo "=== Installing ArgoCD ${ARGOCD_VERSION} ==="

kubectl create namespace $ARGOCD_NAMESPACE --dry-run=client -o yaml | kubectl apply -f -

# HA installation for production
kubectl apply -n $ARGOCD_NAMESPACE -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/${ARGOCD_VERSION}/manifests/ha/install.yaml

kubectl wait --for=condition=Available deployment --all -n $ARGOCD_NAMESPACE --timeout=300s

# Install CLI
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/download/${ARGOCD_VERSION}/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64

ARGOCD_PASSWORD=$(kubectl -n $ARGOCD_NAMESPACE get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d)

echo ""
echo "=== ArgoCD Installed ==="
echo "Admin: admin"
echo "Password: $ARGOCD_PASSWORD"
echo "Access: kubectl port-forward svc/argocd-server -n argocd 8080:443"
```

## Helm Installation

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --version 7.8.5 \
  --set global.image.tag=v3.2.5 \
  --set server.replicas=2 \
  --set controller.replicas=2 \
  --set repoServer.replicas=2 \
  --set redis-ha.enabled=true \
  --set configs.params.server\.insecure=true \
  --wait
```

## Application Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://gitlab.example.com/myorg/manifests.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```