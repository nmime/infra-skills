# Bastion Server Setup

## Modes

| Mode | Tier | Bastion Handles | Load Balancer |
|------|------|-----------------|---------------|
| **VPN Only** | small, medium, production | VPN + Admin | Yes (public apps) |
| **Full Proxy** | minimal | VPN + Admin + Public Apps | No (save €6) |

## Bastion VPN Only Mode (Recommended)

```bash
#!/bin/bash
# scripts/setup-bastion-vpn-only.sh

set -euo pipefail

DOMAIN="${1:-example.com}"
EMAIL="${2:-admin@example.com}"

echo "=== Setting up Bastion (VPN Only) ==="

# Install packages
apt update
apt install -y curl wget

# Install Caddy for TLS
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy.gpg
echo "deb [signed-by=/usr/share/keyrings/caddy.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main" > /etc/apt/sources.list.d/caddy.list
apt update && apt install -y caddy

# Install Headscale
HEADSCALE_VERSION="0.25.1"
wget -O /tmp/headscale.deb https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_amd64.deb
dpkg -i /tmp/headscale.deb

# Configure Headscale
mkdir -p /etc/headscale /var/lib/headscale
cat > /etc/headscale/config.yaml << EOF
server_url: https://vpn.${DOMAIN}
listen_addr: 127.0.0.1:8080
metrics_listen_addr: 127.0.0.1:9090

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10

derp:
  server:
    enabled: true
    region_id: 999
    stun_listen_addr: 0.0.0.0:3478

database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite

dns:
  magic_dns: true
  base_domain: internal
  nameservers:
    global:
      - 1.1.1.1
  extra_records:
    - name: gitlab.${DOMAIN}
      type: A
      value: 10.0.10.1
    - name: argocd.${DOMAIN}
      type: A
      value: 10.0.10.2
    - name: grafana.${DOMAIN}
      type: A
      value: 10.0.10.3
    - name: minio.${DOMAIN}
      type: A
      value: 10.0.10.4
    - name: vault.${DOMAIN}
      type: A
      value: 10.0.10.5

acl_policy_path: /etc/headscale/acl.json
EOF

# Caddy config (VPN only)
cat > /etc/caddy/Caddyfile << EOF
vpn.${DOMAIN} {
    reverse_proxy localhost:8080
    tls ${EMAIL}
}
EOF

# Start services
systemctl enable headscale caddy
systemctl start headscale caddy

# Setup Tailscale on bastion to advertise routes
curl -fsSL https://tailscale.com/install.sh | sh
headscale users create bastion
AUTH_KEY=$(headscale preauthkeys create --user bastion --reusable --expiration 87600h | tail -1)
tailscale up --login-server=https://vpn.${DOMAIN} --authkey=${AUTH_KEY} --advertise-routes=10.0.0.0/16
headscale routes enable --route 10.0.0.0/16

echo "=== Bastion Ready (VPN Only) ==="
echo "VPN: https://vpn.${DOMAIN}"
```

## Bastion Full Proxy Mode (Minimal Tier)

Handles ALL traffic - no Load Balancer needed.

```bash
#!/bin/bash
# scripts/setup-bastion-full-proxy.sh

set -euo pipefail

DOMAIN="${1:-example.com}"
EMAIL="${2:-admin@example.com}"
GATEWAY_IP="10.0.10.100"  # Cilium Gateway internal IP

echo "=== Setting up Bastion (Full Proxy) ==="

# Install Caddy
apt update
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy.gpg
echo "deb [signed-by=/usr/share/keyrings/caddy.gpg] https://dl.cloudsmith.io/public/caddy/stable/deb/debian any-version main" > /etc/apt/sources.list.d/caddy.list
apt update && apt install -y caddy

# Install Headscale
HEADSCALE_VERSION="0.25.1"
wget -O /tmp/headscale.deb https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_amd64.deb
dpkg -i /tmp/headscale.deb

# Headscale config (same as above)
mkdir -p /etc/headscale /var/lib/headscale
cat > /etc/headscale/config.yaml << EOF
server_url: https://vpn.${DOMAIN}
listen_addr: 127.0.0.1:8080
metrics_listen_addr: 127.0.0.1:9090
private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key
prefixes:
  v4: 100.64.0.0/10
derp:
  server:
    enabled: true
    region_id: 999
    stun_listen_addr: 0.0.0.0:3478
database:
  type: sqlite
  sqlite:
    path: /var/lib/headscale/db.sqlite
dns:
  magic_dns: true
  base_domain: internal
  nameservers:
    global:
      - 1.1.1.1
  extra_records:
    - name: gitlab.${DOMAIN}
      type: A
      value: 10.0.10.1
    - name: argocd.${DOMAIN}
      type: A  
      value: 10.0.10.2
    - name: grafana.${DOMAIN}
      type: A
      value: 10.0.10.3
acl_policy_path: /etc/headscale/acl.json
EOF

# Caddy config - FULL PROXY (VPN + Public Apps)
cat > /etc/caddy/Caddyfile << EOF
# VPN
vpn.${DOMAIN} {
    reverse_proxy localhost:8080
    tls ${EMAIL}
}

# PUBLIC APPS - Proxied to K8s Gateway
app.${DOMAIN} {
    reverse_proxy ${GATEWAY_IP}:443 {
        transport http {
            tls_insecure_skip_verify
        }
        header_up Host {host}
    }
    tls ${EMAIL}
}

api.${DOMAIN} {
    reverse_proxy ${GATEWAY_IP}:443 {
        transport http {
            tls_insecure_skip_verify
        }
        header_up Host {host}
    }
    tls ${EMAIL}
}

s3.${DOMAIN} {
    reverse_proxy ${GATEWAY_IP}:443 {
        transport http {
            tls_insecure_skip_verify
        }
        header_up Host {host}
    }
    tls ${EMAIL}
}

registry.${DOMAIN} {
    reverse_proxy ${GATEWAY_IP}:443 {
        transport http {
            tls_insecure_skip_verify
        }
        header_up Host {host}
    }
    tls ${EMAIL}
}
EOF

# Start services
systemctl enable headscale caddy
systemctl start headscale caddy

# Setup Tailscale
curl -fsSL https://tailscale.com/install.sh | sh
headscale users create bastion
AUTH_KEY=$(headscale preauthkeys create --user bastion --reusable --expiration 87600h | tail -1)
tailscale up --login-server=https://vpn.${DOMAIN} --authkey=${AUTH_KEY} --advertise-routes=10.0.0.0/16
headscale routes enable --route 10.0.0.0/16

echo "=== Bastion Ready (Full Proxy) ==="
echo "VPN: https://vpn.${DOMAIN}"
echo "Apps: https://app.${DOMAIN}, https://api.${DOMAIN}"
echo "S3: https://s3.${DOMAIN}"
```

## Comparison

| Feature | VPN Only | Full Proxy |
|---------|----------|------------|
| Cost | +€6 (LB) | €0 |
| Public apps | Via LB | Via Bastion |
| Redundancy | LB + Bastion | Single point |
| Performance | Better | Good |
| Best for | Production | Dev/Minimal |