# cert-manager + TLS Setup

Automatic TLS with Let's Encrypt using DNS-01 challenge.

## Version

| Component | Version |
|-----------|---------|
| cert-manager | v1.19.2 |
| Hetzner webhook | latest |

## Installation Script

```bash
#!/bin/bash
# scripts/install-cert-manager.sh

set -euo pipefail

echo "=== Installing cert-manager v1.19.2 ==="

helm repo add jetstack https://charts.jetstack.io
helm repo update

helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.19.2 \
  --set crds.enabled=true \
  --wait

echo "cert-manager installed!"
```

## TLS Setup with Hetzner DNS

```bash
#!/bin/bash
# scripts/setup-tls.sh <domain> <email>

set -euo pipefail

DOMAIN="$1"
EMAIL="${2:-admin@$DOMAIN}"

[[ -z "${HCLOUD_TOKEN:-}" ]] && { echo "HCLOUD_TOKEN required"; exit 1; }

echo "=== Setting up TLS for $DOMAIN ==="

# Install Hetzner webhook for DNS-01
helm repo add cert-manager-webhook-hetzner \
  https://vadimkim.github.io/cert-manager-webhook-hetzner 2>/dev/null || true
helm upgrade --install cert-manager-webhook-hetzner \
  cert-manager-webhook-hetzner/cert-manager-webhook-hetzner \
  --namespace cert-manager --wait

# Create secret with Hetzner token
kubectl create secret generic hetzner-secret \
  --namespace cert-manager \
  --from-literal=api-key="$HCLOUD_TOKEN" \
  --dry-run=client -o yaml | kubectl apply -f -

# ClusterIssuer with DNS-01
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: ${EMAIL}
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - dns01:
          webhook:
            groupName: acme.hetzner.cloud
            solverName: hetzner
            config:
              secretName: hetzner-secret
              apiUrl: https://dns.hetzner.com/api/v1
EOF

# Wildcard certificate
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard
  namespace: cert-manager
spec:
  secretName: wildcard-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "${DOMAIN}"
    - "*.${DOMAIN}"
EOF

echo "TLS setup complete!"
echo "Check: kubectl get certificate -n cert-manager"
```

## Verify

```bash
# Check certificate status
kubectl get certificate -n cert-manager

# Check secret
kubectl get secret wildcard-tls -n cert-manager

# View cert details
kubectl get secret wildcard-tls -n cert-manager \
  -o jsonpath='{.data.tls\.crt}' | base64 -d | \
  openssl x509 -text -noout | head -20
```

## Use in Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main
spec:
  gatewayClassName: cilium
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        certificateRefs:
          - name: wildcard-tls
            namespace: cert-manager
```