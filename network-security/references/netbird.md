# NetBird VPN (Alternative)

Open-source alternative to Tailscale with built-in management UI.

## Version Information (January 2026)

| Component | Version |
|-----------|---------|
| NetBird | 0.31.x |

## Why NetBird?

- ✅ Modern web UI for management
- ✅ Built-in user management
- ✅ WireGuard-based (like Tailscale)
- ✅ Easier setup than Headscale
- ✅ Active development

## Installation

```bash
#!/bin/bash
# scripts/install-netbird.sh

set -euo pipefail

NETBIRD_DOMAIN="${1:-vpn.example.com}"

echo "=== Installing NetBird ==="

helm repo add netbird https://netbirdio.github.io/helm-charts
helm repo update

helm upgrade --install netbird netbird/netbird \
  --namespace netbird \
  --create-namespace \
  --set management.domain=${NETBIRD_DOMAIN} \
  --set dashboard.enabled=true \
  --set signal.enabled=true \
  --set relay.enabled=true \
  --wait

echo "=== NetBird installed ==="
echo "Dashboard: https://${NETBIRD_DOMAIN}"
```

## Comparison: Headscale vs NetBird

| Feature | Headscale | NetBird |
|---------|-----------|----------|
| Protocol | WireGuard | WireGuard |
| Client | Tailscale app | NetBird app |
| Web UI | No (CLI only) | Yes (built-in) |
| User Auth | Manual/OIDC | Built-in/OIDC |
| ACLs | JSON file | Web UI |
| Maturity | Stable | Growing |
| Community | Large | Medium |