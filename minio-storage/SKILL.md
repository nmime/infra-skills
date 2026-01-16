---
name: minio-storage
description: S3-compatible storage. Standalone or distributed based on tier.
---

# MinIO Storage

S3-compatible storage for platform services. (Updated: January 2026). All scripts are **idempotent** - uses `helm upgrade --install`.

**Run from**: Bastion server or any machine with kubectl access.

## ⚠️ Important: MinIO Image Source Change

As of October 2025, MinIO no longer provides official Docker images. Use one of these alternatives:

| Option | Image | Notes |
|--------|-------|-------|
| **Chainguard (Recommended)** | `cgr.dev/chainguard/minio` | Free tier, vulnerability-free |
| **Bitnami** | `bitnami/minio` | Community maintained |
| **Build from source** | Self-built | Requires Go 1.24+ |

## Modes

| Tier | Mode | Replicas |
|------|------|----------|
| minimal/small | standalone | 1 |
| medium/production | distributed | 4 |

## Scripts

```bash
./scripts/install-minio.sh <mode> <size>
./scripts/install-minio.sh standalone 50Gi
./scripts/install-minio.sh distributed 100Gi
```

## Integrations

- GitLab (artifacts, uploads, LFS)
- Loki (log storage)
- Backups

## Reference Files

- [references/installation.md](references/installation.md) - Installation overview
- [references/standalone.md](references/standalone.md) - Standalone mode
- [references/distributed.md](references/distributed.md) - Distributed mode
- [references/buckets.md](references/buckets.md) - Bucket management
- [references/gitlab-integration.md](references/gitlab-integration.md) - GitLab integration
- [references/loki-integration.md](references/loki-integration.md) - Loki integration
- [references/backup-integration.md](references/backup-integration.md) - Backup integration
- [references/monitoring.md](references/monitoring.md) - Monitoring
- [references/operations.md](references/operations.md) - Day-to-day operations