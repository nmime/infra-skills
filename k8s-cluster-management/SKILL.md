---
name: k8s-cluster-management
description: Kubernetes installation (Kubespray) + core addons (Cilium, Gateway API, cert-manager, LoadBalancer). Multi-cloud support.
---

# K8s Cluster Management

Kubernetes via **Kubespray** + core addons. (Updated: January 2026). All scripts are **idempotent** - Kubespray playbooks converge to desired state.

**Run from**: Bastion server (Ansible runs from bastion to cluster nodes).

## Cloud Provider Support

| Provider | LoadBalancer | Script Flag |
|----------|--------------|-------------|
| `hetzner` | Hetzner CCM v1.22.0 | `--provider hetzner` |
| `aws` | AWS Cloud Provider | `--provider aws` |
| `gcp` | GCP Cloud Provider | `--provider gcp` |
| `azure` | Azure Cloud Provider | `--provider azure` |
| `baremetal` | MetalLB v0.14.9 | `--provider baremetal` |

## Components (January 2026)

| Component | Version | Purpose |
|-----------|---------|--------|
| Kubernetes | v1.34.3 | Cluster |
| Kubespray | v2.29.1 | Installer |
| etcd | v3.5.26 | Key-value store |
| containerd | v2.2.1 | Container runtime |
| Cilium | v1.18.6 | CNI + Gateway |
| Gateway API | v1.4.0 | Ingress |
| cert-manager | v1.19.2 | TLS automation |
| MetalLB | v0.14.9 | Bare metal LB |

> **Note**: For K8s v1.35.0, wait for Kubespray v2.30+.

## Tiers

| Tier | Masters | Workers | HA |
|------|---------|---------|----|
| minimal | 1* | 1 | ❌ |
| small | 1 | 2 | ❌ |
| medium | 3 | 2 | ✅ |
| production | 3 | 3+ | ✅ |

*schedulable

## Installation

Run from bastion server. See reference files for detailed commands:
- Kubernetes cluster: [references/kubespray.md](references/kubespray.md)
- CNI: [references/cilium.md](references/cilium.md)
- Ingress: [references/gateway-api.md](references/gateway-api.md)
- TLS: [references/cert-manager.md](references/cert-manager.md)

## kubectl Access

After installation, kubectl works directly from bastion:

```bash
# On bastion
kubectl get nodes
kubectl get pods -A
```

Or via VPN from any connected server:

```bash
# Connect to VPN first
tailscale up --login-server https://vpn.example.com --authkey <KEY>

# Then kubectl works
kubectl get nodes
```

## Reference Files

- [references/kubespray.md](references/kubespray.md) - Installation
- [references/cilium.md](references/cilium.md) - CNI
- [references/gateway-api.md](references/gateway-api.md) - Ingress
- [references/cert-manager.md](references/cert-manager.md) - TLS
- [references/upgrades.md](references/upgrades.md) - Cluster upgrades
- [references/essential-components.md](references/essential-components.md) - Essential components
- [references/troubleshooting.md](references/troubleshooting.md) - Troubleshooting