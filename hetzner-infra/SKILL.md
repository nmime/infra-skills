---
name: hetzner-infra
description: Hetzner cloud infrastructure provisioning for Kubernetes. Use when provisioning servers, networks, load balancers, firewalls, DNS zones, or storage on Hetzner Cloud via hcloud CLI.
---

# Cloud Infrastructure

**Infrastructure patterns for Kubernetes clusters.** Implementation via hcloud CLI. All scripts are **idempotent**.

## Core Components

| Component | Purpose | hcloud Command |
|-----------|---------|----------------|
| Compute | VM instances for nodes | `hcloud server` |
| Network | Private connectivity | `hcloud network` |
| Load Balancer | Traffic distribution | `hcloud load-balancer` |
| Firewall | Network security | `hcloud firewall` |
| DNS | Name resolution | `hcloud zone` |
| Storage | Block storage | `hcloud volume` |

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Private Network (10.0.0.0/16)            │
│                                                             │
│  ┌─────────┐    ┌─────────────────────────────────────┐    │
│  │ Bastion │    │         Control Plane               │    │
│  │10.0.0.1 │    │  master-1   master-2   master-3    │    │
│  │   SSH   │    │  10.0.1.1   10.0.1.2   10.0.1.3    │    │
│  └─────────┘    └─────────────────────────────────────┘    │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │                   Workers                            │   │
│  │  worker-1   worker-2   worker-3   ...   worker-N    │   │
│  │  10.0.2.1   10.0.2.2   10.0.2.3         10.0.2.N    │   │
│  └─────────────────────────────────────────────────────┘   │
│                            │                                │
│                     ┌──────┴──────┐                        │
│                     │ Load Balancer│                        │
│                     │  10.0.0.10   │                        │
│                     └──────────────┘                        │
└─────────────────────────────────────────────────────────────┘
                            │
                      Public Internet
                       (:80, :443)
```

## IP Allocation

| Range | Purpose |
|-------|---------|
| `10.0.0.0/24` | Infrastructure (bastion, LB) |
| `10.0.1.0/24` | Control plane nodes |
| `10.0.2.0/24` | Worker nodes |
| `10.0.10.0/24` | Service IPs (MetalLB) |
| `100.64.0.0/10` | VPN overlay (Tailscale/Headscale) |

## Security Architecture

**Bastion** (public-facing):
- SSH (22/tcp) - from anywhere
- VPN (41641/udp) - Tailscale/WireGuard
- ICMP - connectivity testing

**Control Plane** (private):
- SSH (22/tcp) - from private net + VPN
- K8s API (6443/tcp) - from private net + VPN
- etcd (2379-2380/tcp) - from control plane only
- Kubelet (10250-10252/tcp) - from private net + VPN
- Cilium (4240/tcp, 8472/udp) - from private net

**Workers** (private):
- SSH (22/tcp) - from private net + VPN
- Kubelet (10250/tcp) - from private net + VPN
- NodePorts (30000-32767/tcp) - from private net + LB
- Cilium (4240/tcp, 8472/udp) - from private net

**Load Balancer** (public-facing):
- HTTP (80/tcp), HTTPS (443/tcp) - from anywhere

## Deployment Tiers

| Tier | Masters | Workers | HA | Cost |
|------|---------|---------|----|----|
| Minimal | 1 (schedulable) | 0 | No | ~€12/mo |
| Small | 1 | 2 | No | ~€21/mo |
| Medium | 3 | 2 | Yes | ~€34/mo |
| Production | 3 | 3+ | Yes | ~€48/mo |

## Quick Start

```bash
# Install hcloud CLI
curl -sL https://github.com/hetznercloud/cli/releases/latest/download/hcloud-linux-amd64.tar.gz | tar xz
sudo mv hcloud /usr/local/bin/

# Set token
export HCLOUD_TOKEN="your-token"

# Verify
hcloud server list
```

## hcloud Reference

| Resource | Reference |
|----------|-----------|
| Servers | [hcloud-server.md](references/hcloud-server.md) |
| Networks | [hcloud-network.md](references/hcloud-network.md) |
| Load Balancers | [hcloud-load-balancer.md](references/hcloud-load-balancer.md) |
| Firewalls | [hcloud-firewall.md](references/hcloud-firewall.md) |
| Volumes | [hcloud-volume.md](references/hcloud-volume.md) |
| Floating IPs | [hcloud-floating-ip.md](references/hcloud-floating-ip.md) |
| Primary IPs | [hcloud-primary-ip.md](references/hcloud-primary-ip.md) |
| SSH Keys | [hcloud-ssh-key.md](references/hcloud-ssh-key.md) |
| Images | [hcloud-image.md](references/hcloud-image.md) |
| Certificates | [hcloud-certificate.md](references/hcloud-certificate.md) |
| Placement Groups | [hcloud-placement-group.md](references/hcloud-placement-group.md) |
| DNS Zones | [hcloud-zone.md](references/hcloud-zone.md) |
| Storage Boxes | [hcloud-storage-box.md](references/hcloud-storage-box.md) |
| Datacenters | [hcloud-datacenter.md](references/hcloud-datacenter.md) |
| Context | [hcloud-context.md](references/hcloud-context.md) |

## Provisioning

See [references/provisioning.md](references/provisioning.md) for step-by-step infrastructure setup.

## References

- [provisioning.md](references/provisioning.md) - Step-by-step setup
- [terraform.md](references/terraform.md) - Infrastructure as Code
- [naming.md](references/naming.md) - Naming conventions
- [cost-optimization.md](references/cost-optimization.md) - Cost strategies
- [scripts.md](references/scripts.md) - Automation scripts
- [cert-manager-hetzner.md](references/cert-manager-hetzner.md) - TLS certificates
