# Skill Management

The orchestrator can read, validate, and update all managed skills.

## Managed Skills

```
platform-orchestrator/
├── Manages:
│   ├── network-security       # VPN, firewall, TLS
│   ├── k8s-cluster-management # Kubernetes setup
│   ├── minio-storage          # S3 storage
│   ├── k8s-secrets            # Vault, ESO
│   ├── k8s-databases          # PostgreSQL, MongoDB
│   ├── gitlab-selfhosted      # GitLab CI/CD
│   ├── k8s-gitops             # ArgoCD
│   ├── k8s-observability      # Monitoring, logging
│   └── k8s-autoscaling        # KEDA
```

## Commands

```bash
# List all skills
./platform.sh skill list

# Validate all skills
./platform.sh skill validate

# Read skill file
./platform.sh skill read minio-storage SKILL.md
./platform.sh skill read gitlab-selfhosted references/gitlab-light.md

# Update skill file (auto-backup created)
./platform.sh skill update minio-storage references/config.yaml "$NEW_CONTENT"
```

## Auto-Update on Fix

When the orchestrator fixes an issue, it can update the skill:

```bash
# Example: OOM fix for GitLab
# 1. Detect OOM
# 2. Increase memory limit in cluster
# 3. Update skill config to persist fix

# The fix is saved to:
# - .state/fixes.yaml (for this deployment)
# - gitlab-selfhosted/references/gitlab-light.md (permanent)
```

## Validation

```bash
$ ./platform.sh skill validate

Validating all skills...
========================
Validating: network-security
  ✓ Valid
Validating: minio-storage
  ✓ Valid
Validating: gitlab-selfhosted
  ✓ Valid
...

All skills valid!
```

## Backup & Rollback

```bash
# Backups stored in:
.backups/
├── gitlab-selfhosted-references-gitlab-light.md-20260115-103000
├── minio-storage-SKILL.md-20260115-110000
└── ...

# Rollback
cp .backups/gitlab-selfhosted-...-20260115-103000 ../gitlab-selfhosted/references/gitlab-light.md
```