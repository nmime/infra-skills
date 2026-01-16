# Deployment Guide

Run all commands from **bastion server**. All deployments are **idempotent** - safe to run multiple times.

## Prerequisites

1. **Bastion tools:**
   ```bash
   # Install required CLI tools on bastion
   apt update && apt install -y kubectl jq

   # Install yq
   wget https://github.com/mikefarah/yq/releases/latest/download/yq_linux_amd64 -O /usr/local/bin/yq
   chmod +x /usr/local/bin/yq

   # Install helm
   curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
   ```

2. **SSH Key:**
   ```bash
   ssh-keygen -t ed25519 -C "platform-admin"
   ```

3. **Cloud Provider Credentials:**

   | Provider | Credential |
   |----------|------------|
   | `hetzner` | `export HCLOUD_TOKEN="..."` |
   | `aws` | `export AWS_ACCESS_KEY_ID="..." AWS_SECRET_ACCESS_KEY="..."` |
   | `gcp` | `export GOOGLE_APPLICATION_CREDENTIALS="/path/to/sa.json"` |
   | `azure` | `export AZURE_SUBSCRIPTION_ID="..." AZURE_TENANT_ID="..."` |
   | `baremetal` | No cloud credentials needed |

## Deployment Steps

### 1. Initialize

```bash
# Create config from template
./platform.sh init

# Edit configuration - SET YOUR CLOUD PROVIDER
vim platform.yaml
```

**Required configuration:**
```yaml
infrastructure:
  cloud_provider: hetzner  # hetzner | aws | gcp | azure | baremetal
  # For baremetal, also set:
  # metallb:
  #   address_pool: 192.168.1.240-192.168.1.250
```

### 2. Deploy Everything

```bash
# Deploy all components in order
./platform.sh deploy all

# This runs:
# 1. Infrastructure (provider-specific or skip for baremetal)
# 2. DNS records (provider-specific or manual)
# 3. Kubernetes cluster (Kubespray)
# 4. LoadBalancer (CCM or MetalLB based on provider)
# 5. TLS (cert-manager)
# 6. MinIO storage
# 7. Vault + ESO
# 8. PostgreSQL
# 9. GitLab + Runner
# 10. ArgoCD
# 11. VictoriaMetrics + Loki + Grafana
# 12. KEDA
```

### 3. Deploy Individual Components

```bash
./platform.sh deploy infra         # Infrastructure (Hetzner only, skip for others)
./platform.sh deploy dns           # DNS records (Hetzner only, manual for others)
./platform.sh deploy cluster       # Kubernetes via Kubespray
./platform.sh deploy loadbalancer  # CCM or MetalLB (based on cloud_provider)
./platform.sh deploy tls           # cert-manager
./platform.sh deploy minio         # S3 storage
./platform.sh deploy secrets       # Vault + ESO
./platform.sh deploy databases     # PostgreSQL
./platform.sh deploy gitlab        # GitLab CE
./platform.sh deploy gitops        # ArgoCD
./platform.sh deploy observability # Monitoring stack
./platform.sh deploy autoscaling   # KEDA
```

### Provider-Specific Notes

**Hetzner:** Full automation - infra, DNS, LoadBalancer all managed.

**AWS/GCP/Azure:** Provision infrastructure using your tools (Terraform, CloudFormation, etc.), then run cluster deployment. LoadBalancer uses cloud CCM.

**Bare Metal:** Provision servers manually, ensure network connectivity, configure MetalLB IP range in `platform.yaml`.

### 4. Verify

```bash
# Check status
./platform.sh status

# Get credentials
./platform.sh credentials

# Health check
./platform.sh health
```

## Post-Deployment

### Initialize Vault

```bash
# Initialize (save keys securely!)
kubectl exec -n vault vault-0 -- vault operator init

# Unseal (run 3 times with different keys)
kubectl exec -n vault vault-0 -- vault operator unseal <key1>
kubectl exec -n vault vault-0 -- vault operator unseal <key2>
kubectl exec -n vault vault-0 -- vault operator unseal <key3>
```

**Note:** For automated environments, configure auto-unseal. See [self-healing.md](self-healing.md#vault-auto-unseal-options) for options including Kubernetes secrets, Transit auto-unseal, and cloud KMS integration.

### Verify DNS

DNS records are created automatically. Verify:

```bash
dig gitlab.example.com +short
dig argocd.example.com +short
dig grafana.example.com +short
```

### Install GitLab Runner

```bash
# Get runner token from GitLab UI
# Admin > CI/CD > Runners > New instance runner

# Install runner
./platform.sh deploy gitlab-runner
```

## Upgrade

```bash
# Update platform.yaml with new versions
vim platform.yaml

# Re-deploy specific component
./platform.sh deploy gitlab

# Or upgrade all
./platform.sh deploy all
```

## Destroy

```bash
# Remove everything (DANGEROUS!)
# DNS: Only platform records removed, zone preserved
./platform.sh destroy
```