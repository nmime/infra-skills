---
name: platform-orchestrator
description: Orchestrates all skills for unified platform deployment. All services enabled by default. Run from bastion server.
---

# Platform Orchestrator

Unified deployment orchestration. **All services enabled by default.** All deployments are **idempotent** - safe to run multiple times.

**Run from**: Bastion server or sandboxed environment.

## Cloud Provider Support

| Provider | LoadBalancer | Status |
|----------|--------------|--------|
| `hetzner` | Hetzner CCM | Default |
| `aws` | AWS Cloud Provider | Supported |
| `gcp` | GCP Cloud Provider | Supported |
| `azure` | Azure Cloud Provider | Supported |
| `baremetal` | MetalLB | For bare metal / other clouds |

Set in `platform.yaml`:
```yaml
infrastructure:
  cloud_provider: hetzner  # hetzner | aws | gcp | azure | baremetal
```

## Component Versions (January 2026)

| Component | Version |
|-----------|---------|
| Kubespray | v2.29.1 |
| Kubernetes | v1.34.3 |
| etcd | v3.5.26 |
| containerd | v2.2.1 |
| Cilium | v1.18.6 |
| Gateway API | v1.4.0 |
| cert-manager | v1.19.2 |
| ArgoCD | v3.2.5 |
| Vault | v1.21.2 |
| ESO | v1.2.1 |
| VictoriaMetrics | v1.133.0 |
| Grafana | v12.3.1 |
| Loki | v3.6.3 |
| KEDA | v2.18.x |
| PostgreSQL (Percona) | v2.8.2 |
| MinIO (Bitnami) | v17.0.21 |
| MetalLB | v0.14.9 |

> **Note**: MinIO official images discontinued Oct 2025. Using Bitnami Helm chart.

## Naming Convention

All resources use consistent naming: `{project}-{resource}`

| Resource | Pattern | Example |
|----------|---------|--------|
| Network | `{project}-network` | `myapp-network` |
| Bastion | `{project}-bastion` | `myapp-bastion` |
| Masters | `{project}-master-{n}` | `myapp-master-1` |
| Workers | `{project}-worker-{n}` | `myapp-worker-1` |
| Load Balancer | `{project}-lb` | `myapp-lb` |
| K8s Namespaces | Service name | `gitlab`, `argocd`, `monitoring` |

Default project name: `k8s` (configurable in `platform.yaml`)

## Services (All Enabled by Default)

| Service | Default | DNS Records |
|---------|---------|-------------|
| GitLab | ✅ enabled | `gitlab`, `registry` |
| ArgoCD | ✅ enabled | `argocd` |
| Grafana | ✅ enabled | `grafana` |
| VictoriaMetrics | ✅ enabled | `victoriametrics` |
| Loki | ✅ enabled | `loki` |
| MinIO | ✅ enabled | `minio`, `s3` |
| Vault | ✅ enabled | `vault` |
| PostgreSQL | ✅ enabled | - |
| KEDA | ✅ enabled | - |
| Headscale VPN | ✅ enabled | `vpn` |

Always included: `@`, `*`, `api`, `app`

## Managed Skills

| Skill | Responsibility | Provider |
|-------|---------------|----------|
| **hetzner-infra** | Servers, networks, LB, DNS | Hetzner only |
| **k8s-cluster-management** | Kubernetes, Cilium, TLS, CCM/MetalLB | All providers |
| **network-security** | VPN (Headscale), firewall | All providers |
| **minio-storage** | S3 storage | All providers |
| **k8s-secrets** | Vault + ESO | All providers |
| **k8s-databases** | PostgreSQL | All providers |
| **gitlab-selfhosted** | GitLab CE | All providers |
| **k8s-gitops** | ArgoCD | All providers |
| **k8s-observability** | Monitoring | All providers |
| **k8s-autoscaling** | KEDA | All providers |

> **Note**: For non-Hetzner providers, provision infrastructure manually or with your cloud's tools (Terraform, CloudFormation, etc.), then run k8s-cluster-management.

## Commands

Run from bastion server:

```bash
./platform.sh init              # Create config
./platform.sh deploy all        # Full deployment
./platform.sh deploy <comp>     # Deploy component
./platform.sh status            # Show status
./platform.sh credentials       # Show passwords
./platform.sh health            # Health check
./platform.sh heal              # Auto-fix issues
./platform.sh destroy           # Remove all
```

## Deployment Order

```
1. infra      → hetzner-infra (servers, network, LB)
2. dns        → hetzner-infra (records for enabled services)
3. cluster    → k8s-cluster-management (Kubespray)
4. tls        → k8s-cluster-management (cert-manager)
5. minio      → minio-storage
6. secrets    → k8s-secrets (Vault)
7. databases  → k8s-databases (PostgreSQL)
8. gitlab     → gitlab-selfhosted
9. gitops     → k8s-gitops (ArgoCD)
10. observability → k8s-observability
11. autoscaling   → k8s-autoscaling (KEDA)
```

## Tiers

| Tier | Cost | Nodes | HA |
|------|------|-------|----|
| minimal | ~€12/mo | 2 | ❌ |
| small | ~€21/mo | 3 | ❌ |
| medium | ~€34/mo | 5 | ✅ |
| production | ~€48/mo | 6 | ✅ |

## Service Dependencies

| Service | Required Dependencies | Optional |
|---------|----------------------|----------|
| MinIO | K8s cluster | - |
| Vault | K8s cluster | - |
| PostgreSQL | K8s cluster | - |
| GitLab | K8s, PostgreSQL, MinIO | Vault |
| ArgoCD | K8s cluster | GitLab |
| Loki | K8s cluster, MinIO | - |
| VictoriaMetrics | K8s cluster | - |
| Grafana | K8s, VictoriaMetrics | Loki |
| KEDA | K8s cluster | - |
| Headscale | Bastion server | - |

## Reference Files

- [references/configuration.md](references/configuration.md) - Configuration options
- [references/deployment.md](references/deployment.md) - Deployment guide
- [references/operations.md](references/operations.md) - Day-to-day operations
- [references/scaling.md](references/scaling.md) - Scaling strategies
- [references/self-healing.md](references/self-healing.md) - Self-healing features
- [references/skill-management.md](references/skill-management.md) - Managing skills
- [references/troubleshooting.md](references/troubleshooting.md) - Troubleshooting