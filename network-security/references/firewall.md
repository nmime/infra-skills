# Firewall Configuration

## Hetzner Firewall

```bash
#!/bin/bash
# scripts/setup-firewall.sh

set -euo pipefail

echo "=== Setting up Hetzner Firewall ==="

# Firewall for Bastion (public facing)
hcloud firewall create --name bastion-fw

# Allow HTTPS (Headscale)
hcloud firewall add-rule bastion-fw \
  --direction in --protocol tcp --port 443 \
  --source-ips 0.0.0.0/0 --source-ips ::/0 \
  --description "HTTPS/Headscale"

# Allow STUN/DERP
hcloud firewall add-rule bastion-fw \
  --direction in --protocol udp --port 3478 \
  --source-ips 0.0.0.0/0 --source-ips ::/0 \
  --description "STUN/DERP"

# Allow SSH (optional, can restrict to your IP)
hcloud firewall add-rule bastion-fw \
  --direction in --protocol tcp --port 22 \
  --source-ips 0.0.0.0/0 \
  --description "SSH"

# Apply to bastion
hcloud firewall apply-to-resource bastion-fw --type server --server bastion

# Firewall for K8s nodes (private only)
hcloud firewall create --name k8s-private-fw

# Allow ALL from private network
hcloud firewall add-rule k8s-private-fw \
  --direction in --protocol tcp --port any \
  --source-ips 10.0.0.0/8 \
  --description "Private TCP"

hcloud firewall add-rule k8s-private-fw \
  --direction in --protocol udp --port any \
  --source-ips 10.0.0.0/8 \
  --description "Private UDP"

hcloud firewall add-rule k8s-private-fw \
  --direction in --protocol icmp \
  --source-ips 10.0.0.0/8 \
  --description "Private ICMP"

# Allow Tailscale VPN subnet
hcloud firewall add-rule k8s-private-fw \
  --direction in --protocol tcp --port any \
  --source-ips 100.64.0.0/10 \
  --description "Tailscale TCP"

hcloud firewall add-rule k8s-private-fw \
  --direction in --protocol udp --port any \
  --source-ips 100.64.0.0/10 \
  --description "Tailscale UDP"

# Apply to K8s nodes
for server in master-1 master-2 master-3 worker-1 worker-2 worker-3; do
  hcloud firewall apply-to-resource k8s-private-fw --type server --server ${server}
done

echo "=== Firewalls Configured ==="
hcloud firewall list
```

## Summary of Firewall Rules

### Bastion Server (Public)

| Port | Protocol | Source | Purpose |
|------|----------|--------|--------|
| 443 | TCP | 0.0.0.0/0 | Headscale VPN |
| 3478 | UDP | 0.0.0.0/0 | STUN/DERP |
| 22 | TCP | Your IP | SSH (optional) |

### K8s Nodes (Private Only)

| Port | Protocol | Source | Purpose |
|------|----------|--------|--------|
| * | TCP/UDP | 10.0.0.0/8 | Private network |
| * | TCP/UDP | 100.64.0.0/10 | VPN clients |

**Result**: K8s nodes have **ZERO** exposure to internet!