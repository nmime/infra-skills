---
name: network-security
description: VPN access and firewall rules. Headscale VPN on bastion for admin access to private services.
---

# Network Security

Headscale v0.27.1 VPN and firewall for secure admin access. (Updated: January 2026). All scripts are **idempotent** - check state before applying changes.

**Run from**: Bastion server (scripts execute on bastion itself).

## Responsibility

| This Skill | Other Skills |
|------------|-------------|
| Headscale VPN setup | Servers → hetzner-infra |
| VPN user management | DNS → hetzner-infra |
| Firewall rules | TLS → k8s-cluster-management |
| Bastion hardening | LB → hetzner-infra |

## Architecture

```
INTERNET
    │
    ├─ PUBLIC (via LB) ───▶ app, api, s3, registry
    │
    └─ ADMIN (via VPN) ──▶ gitlab, argocd, grafana, vault, k8s
                │
                └──▶ Bastion + Headscale
```

## Scripts

Run on bastion server:

```bash
./scripts/setup-headscale.sh      # VPN server
./scripts/add-vpn-user.sh <name>  # Add user
./scripts/setup-firewall.sh       # Firewall rules
```

## VPN Client Access

Connect from any server or sandboxed environment:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
tailscale up --login-server https://vpn.example.com --authkey <KEY>
```

## Reference Files

- [references/headscale.md](references/headscale.md) - VPN server
- [references/netbird.md](references/netbird.md) - NetBird alternative
- [references/users.md](references/users.md) - User management
- [references/firewall.md](references/firewall.md) - Firewall rules
- [references/bastion.md](references/bastion.md) - Bastion hardening
- [references/architecture.md](references/architecture.md) - Network architecture
- [references/hetzner-network.md](references/hetzner-network.md) - Hetzner network setup
- [references/load-balancer.md](references/load-balancer.md) - Load balancer
- [references/dns.md](references/dns.md) - DNS configuration
- [references/tls.md](references/tls.md) - TLS certificates