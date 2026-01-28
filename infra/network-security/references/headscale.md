# Headscale VPN on Bastion

Run Headscale directly on Bastion server (not in K8s).

## Version Information

| Component | Version |
|-----------|---------|
| Headscale | 0.25.1 |
| Tailscale Client | 1.76.x |

## Install Headscale on Bastion

```bash
#!/bin/bash
# Run ON the Bastion server
# scripts/setup-bastion-vpn.sh

set -euo pipefail

HEADSCALE_VERSION="0.25.1"
DOMAIN="${1:-vpn.example.com}"
BASTION_PRIVATE_IP="10.0.0.2"

echo "=== Installing Headscale ${HEADSCALE_VERSION} ==="

# Download Headscale
wget -O /tmp/headscale.deb \
  https://github.com/juanfont/headscale/releases/download/v${HEADSCALE_VERSION}/headscale_${HEADSCALE_VERSION}_linux_amd64.deb

dpkg -i /tmp/headscale.deb

# Create config directory
mkdir -p /etc/headscale
mkdir -p /var/lib/headscale

# Create configuration
cat > /etc/headscale/config.yaml << EOF
server_url: https://${DOMAIN}
listen_addr: 0.0.0.0:443
metrics_listen_addr: 127.0.0.1:9090
grpc_listen_addr: 127.0.0.1:50443
grpc_allow_insecure: false

private_key_path: /var/lib/headscale/private.key
noise:
  private_key_path: /var/lib/headscale/noise_private.key

prefixes:
  v4: 100.64.0.0/10
  v6: fd7a:115c:a1e0::/48

derp:
  server:
    enabled: true
    region_id: 999
    region_code: "bastion"
    region_name: "Bastion DERP"
    stun_listen_addr: 0.0.0.0:3478
  urls: []
  auto_update_enabled: true
  update_frequency: 24h

disable_check_updates: false
ephemeral_node_inactivity_timeout: 30m

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
      - 8.8.8.8
  extra_records:
    - name: gitlab.example.com
      type: A
      value: 10.0.10.1
    - name: argocd.example.com
      type: A
      value: 10.0.10.2
    - name: grafana.example.com
      type: A
      value: 10.0.10.3

log:
  format: text
  level: info

acl_policy_path: /etc/headscale/acl.json
EOF

# Create ACL policy
cat > /etc/headscale/acl.json << 'EOF'
{
  "groups": {
    "group:admin": [],
    "group:dev": [],
    "group:readonly": []
  },
  "tagOwners": {
    "tag:k8s-node": ["group:admin"],
    "tag:bastion": ["group:admin"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["group:admin"],
      "dst": ["*:*"]
    },
    {
      "action": "accept",
      "src": ["group:dev"],
      "dst": [
        "10.0.0.0/16:443",
        "10.0.0.0/16:80",
        "10.0.0.0/16:6443",
        "10.0.0.0/16:22"
      ]
    },
    {
      "action": "accept",
      "src": ["group:readonly"],
      "dst": [
        "10.0.10.3:443"
      ]
    }
  ]
}
EOF

# Enable and start Headscale
systemctl enable headscale
systemctl start headscale

echo "=== Installing Tailscale on Bastion (as subnet router) ==="

curl -fsSL https://tailscale.com/install.sh | sh

headscale users create bastion
AUTH_KEY=$(headscale preauthkeys create --user bastion --reusable --expiration 87600h | tail -1)

tailscale up \
  --login-server=http://127.0.0.1:443 \
  --authkey=${AUTH_KEY} \
  --advertise-routes=10.0.0.0/16 \
  --accept-routes \
  --hostname=bastion

headscale routes enable --route 10.0.0.0/16

echo ""
echo "=== Headscale VPN Installed ==="
echo "Bastion is now advertising route to 10.0.0.0/16"
```

## TLS Certificate for Headscale

```bash
#!/bin/bash
# scripts/setup-bastion-tls.sh
# Run on: bastion server

set -euo pipefail

DOMAIN="${1:-vpn.example.com}"
EMAIL="${2:-admin@example.com}"

echo "=== Setting up TLS with Caddy ==="

apt install -y debian-keyring debian-archive-keyring apt-transport-https
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt update
apt install caddy

cat > /etc/caddy/Caddyfile << EOF
${DOMAIN} {
  reverse_proxy localhost:8080
  tls ${EMAIL}
}
EOF

sed -i 's/listen_addr: 0.0.0.0:443/listen_addr: 127.0.0.1:8080/' /etc/headscale/config.yaml

systemctl restart headscale
systemctl enable caddy
systemctl restart caddy

echo "=== TLS Configured ==="
echo "Headscale available at: https://${DOMAIN}"
```

## User Management

```bash
#!/bin/bash
# scripts/create-vpn-user.sh
# Run on: bastion server

USERNAME="$1"
GROUP="${2:-dev}"

if [[ -z "$USERNAME" ]]; then
  echo "Usage: $0 <username> [group]"
  echo "Groups: admin, dev, readonly"
  exit 1
fi

headscale users create ${USERNAME}

KEY=$(headscale preauthkeys create --user ${USERNAME} --reusable --expiration 168h | tail -1)

echo ""
echo "=== VPN User Created ==="
echo "Username: ${USERNAME}"
echo "Group: ${GROUP}"
echo "Auth Key: ${KEY}"
echo ""
echo "Connect via VPN:"
echo "  tailscale up --login-server https://vpn.example.com --authkey ${KEY}"
echo ""
echo "After connecting, accessible services:"
echo "  - https://gitlab.example.com"
echo "  - https://argocd.example.com"
echo "  - https://grafana.example.com"
echo "  - SSH to cluster nodes via private IPs"
```

## VPN Client Connection

Connect from any server or sandboxed environment:

```bash
# Install Tailscale
curl -fsSL https://tailscale.com/install.sh | sh

# Connect to VPN
tailscale up --login-server https://vpn.example.com --authkey <KEY>

# Verify connection
tailscale status

# Access private network
ping 10.0.1.1  # master-1
kubectl get nodes
```

## Listing Connected Nodes

```bash
# Run on bastion
headscale nodes list
headscale users list
headscale routes list
```