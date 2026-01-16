# Platform Configuration

Configuration is **declarative** - changes trigger idempotent deployments that converge to the desired state.

## Configuration File

All platform settings are in `platform.yaml`. Copy the example and customize:

```bash
cp platform.example.yaml platform.yaml
vim platform.yaml
```

## Required Settings

```yaml
# MUST change these:
global:
  domain: your-domain.com      # Your domain
  email: admin@your-domain.com # Let's Encrypt email

infrastructure:
  cloud_provider: hetzner      # hetzner | aws | gcp | azure | baremetal
  ssh_key_path: ~/.ssh/id_ed25519  # Your SSH key
```

## Cloud Provider Configuration

The `cloud_provider` setting determines LoadBalancer implementation and cloud integrations.

| Provider | LoadBalancer | Cloud Controller | Notes |
|----------|--------------|------------------|-------|
| `hetzner` | Hetzner LB | Hetzner CCM v1.22.0 | Default. Native Hetzner integration |
| `aws` | AWS NLB/ALB | AWS Cloud Provider | Requires IAM permissions |
| `gcp` | GCP LB | GCP Cloud Provider | Requires service account |
| `azure` | Azure LB | Azure Cloud Provider | Requires service principal |
| `baremetal` | MetalLB v0.14.9 | None | For bare metal / on-prem / other clouds |

### Hetzner (Default)

```yaml
infrastructure:
  cloud_provider: hetzner
  hetzner:
    token: ${HCLOUD_TOKEN}     # Or set env var
    network: k8s-network       # Private network name
    location: fsn1             # fsn1 | nbg1 | hel1
```

### AWS

```yaml
infrastructure:
  cloud_provider: aws
  aws:
    region: eu-central-1
    vpc_id: vpc-xxxxx          # Optional: auto-detect if not set
    # IAM: Nodes need ec2:*, elasticloadbalancing:*, ecr:* permissions
```

### GCP

```yaml
infrastructure:
  cloud_provider: gcp
  gcp:
    project: my-project
    region: europe-west1
    network: default
    # Service account with Compute Admin, Network Admin roles
```

### Azure

```yaml
infrastructure:
  cloud_provider: azure
  azure:
    subscription_id: xxxxx
    resource_group: k8s-rg
    location: westeurope
    vnet_name: k8s-vnet
    # Service principal with Contributor role on resource group
```

### Bare Metal / Other Clouds

```yaml
infrastructure:
  cloud_provider: baremetal
  metallb:
    address_pool: 192.168.1.240-192.168.1.250  # IP range for LoadBalancers
    # Or use CIDR notation:
    # address_pool: 192.168.1.240/28
```

> **Note**: For clouds without native K8s integration (DigitalOcean, Vultr, OVH, etc.), use `baremetal` with MetalLB.

## Environment Profiles

### Production (~€48/mo)
```yaml
global:
  environment: production

kubernetes:
  version: v1.34.3  # Kubespray v2.29.1

infrastructure:
  bastion:
    type: cx23      # 2 vCPU, 4GB - €2.99/mo
  control_plane:
    count: 3
    type: cx33      # 4 vCPU, 8GB - €4.99/mo
  workers:
    count: 3
    type: cx33      # 4 vCPU, 8GB - €4.99/mo

storage:
  replicas: 4
  size_per_replica: 100Gi

databases:
  postgresql:
    replicas: 3
```

### Medium (~€34/mo)
```yaml
global:
  environment: staging

infrastructure:
  bastion:
    type: cx23      # 2 vCPU, 4GB - €2.99/mo
  control_plane:
    count: 3
    type: cx23      # 2 vCPU, 4GB - €2.99/mo
  workers:
    count: 2
    type: cx33      # 4 vCPU, 8GB - €4.99/mo

storage:
  replicas: 4
  size_per_replica: 50Gi

databases:
  postgresql:
    replicas: 2
```

### Small (~€21/mo)
```yaml
global:
  environment: development

infrastructure:
  bastion:
    type: cx23      # 2 vCPU, 4GB - €2.99/mo
  control_plane:
    count: 1
    type: cx23      # 2 vCPU, 4GB - €2.99/mo
  workers:
    count: 2
    type: cx23      # 2 vCPU, 4GB - €2.99/mo

storage:
  replicas: 1
  size_per_replica: 20Gi

databases:
  postgresql:
    replicas: 1
```

## Component Toggle

Enable/disable any component:

```yaml
# Minimal setup (just GitLab)
storage:
  enabled: true      # Required for GitLab

secrets:
  enabled: false     # Skip Vault

databases:
  postgresql:
    enabled: true    # Required for GitLab
  mongodb:
    enabled: false   # Skip MongoDB

gitlab:
  enabled: true

gitops:
  enabled: false     # Skip ArgoCD

observability:
  metrics:
    enabled: false
  logging:
    enabled: false
  grafana:
    enabled: false

autoscaling:
  enabled: false
```