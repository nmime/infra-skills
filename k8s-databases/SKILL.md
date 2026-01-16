---
name: k8s-databases
description: PostgreSQL via Percona Operator. Single or HA based on tier.
---

# K8s Databases

Percona PostgreSQL Operator v2.8.2. (Updated: January 2026). All scripts are **idempotent** - operators reconcile to desired state.

**Run from**: Bastion server or any machine with kubectl access.

## Supported PostgreSQL Versions

| Version | Status |
|---------|--------|
| PostgreSQL 18.1 | Latest |
| PostgreSQL 17.7 | Supported |
| PostgreSQL 16.11 | Supported |
| PostgreSQL 15.15 | Supported |
| PostgreSQL 14.20 | Supported |
| PostgreSQL 13.23 | Legacy |

## Modes

| Tier | Replicas | HA |
|------|----------|----|
| minimal/small | 1 | ❌ |
| medium/production | 3 | ✅ |

## Features

- Automated major version upgrades (since v2.4.0)
- Asynchronous I/O for better performance
- Huge pages support (if enabled in K8s)

## Scripts

```bash
./scripts/install-postgresql.sh <tier>
```

## Reference Files

- [references/postgresql.md](references/postgresql.md) - PostgreSQL HA
- [references/postgresql-single.md](references/postgresql-single.md) - PostgreSQL single
- [references/mongodb.md](references/mongodb.md) - MongoDB
- [references/backups.md](references/backups.md) - Backup procedures
- [references/monitoring.md](references/monitoring.md) - Database monitoring