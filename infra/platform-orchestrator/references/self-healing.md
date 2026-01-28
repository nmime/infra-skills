# Self-Healing System

The platform automatically detects and fixes common issues. All fixes are **idempotent** - safe to apply multiple times.

## How It Works

```
┌─────────────────────────────────────────────────────────────┐
│                    SELF-HEALING LOOP                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   1. DETECT                                                 │
│   ────────                                                  │
│   • Monitor pod status (CrashLoop, Pending, OOM)           │
│   • Check PVC status (Pending, Lost)                       │
│   • Verify certificates (expired, invalid)                 │
│   • Test endpoints (connection refused)                    │
│                                                             │
│   2. DIAGNOSE                                               │
│   ──────────                                                │
│   • Parse logs for error patterns                          │
│   • Check events for root cause                            │
│   • Identify affected component                            │
│                                                             │
│   3. FIX                                                    │
│   ───                                                       │
│   • Apply known fix for error pattern                      │
│   • Update resource limits if OOM                          │
│   • Restart pods if stuck                                  │
│   • Fix configurations                                     │
│                                                             │
│   4. PERSIST                                                │
│   ───────                                                   │
│   • Save fix to state file                                 │
│   • Update skill configuration                             │
│   • Apply fix on future deployments                        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Common Auto-Fixes

| Issue | Detection | Auto-Fix |
|-------|-----------|----------|
| CrashLoopBackOff | Pod status | Restart, check logs |
| OOMKilled | Pod events | Increase memory limit |
| ImagePullBackOff | Pod status | Delete pod, retry |
| PVC Pending | PVC status | Check storage class |
| Connection Refused | Logs | Restart dependencies |
| Certificate Expired | Cert status | Trigger renewal |
| Vault Sealed | Vault status | Alert or auto-unseal |

## Vault Auto-Unseal (Kubernetes Secrets)

Vault requires unsealing after restarts. This setup stores unseal keys in Kubernetes secrets for automatic unsealing.

### Step 1: Initialize Vault and Save Keys

```bash
# Initialize Vault (first time only)
kubectl exec -n vault vault-0 -- vault operator init -key-shares=3 -key-threshold=3 -format=json > vault-init.json

# Extract keys
UNSEAL_KEY_1=$(jq -r '.unseal_keys_b64[0]' vault-init.json)
UNSEAL_KEY_2=$(jq -r '.unseal_keys_b64[1]' vault-init.json)
UNSEAL_KEY_3=$(jq -r '.unseal_keys_b64[2]' vault-init.json)
ROOT_TOKEN=$(jq -r '.root_token' vault-init.json)

# Create Kubernetes secret with unseal keys
kubectl create secret generic vault-unseal-keys -n vault \
  --from-literal=key1=$UNSEAL_KEY_1 \
  --from-literal=key2=$UNSEAL_KEY_2 \
  --from-literal=key3=$UNSEAL_KEY_3

# Save root token separately (for admin access)
kubectl create secret generic vault-root-token -n vault \
  --from-literal=token=$ROOT_TOKEN

# IMPORTANT: Backup vault-init.json securely, then delete it
# cp vault-init.json /secure/backup/location/
# rm vault-init.json
```

### Step 2: Create Auto-Unseal CronJob

```yaml
# vault-auto-unseal.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: vault-unseal
  namespace: vault
spec:
  schedule: "*/5 * * * *"  # Check every 5 minutes
  jobTemplate:
    spec:
      template:
        spec:
          serviceAccountName: vault
          containers:
          - name: unseal
            image: hashicorp/vault:1.21.2
            command:
            - /bin/sh
            - -c
            - |
              export VAULT_ADDR=http://vault:8200
              if vault status 2>&1 | grep -q "Sealed.*true"; then
                echo "Vault is sealed, unsealing..."
                vault operator unseal $UNSEAL_KEY_1
                vault operator unseal $UNSEAL_KEY_2
                vault operator unseal $UNSEAL_KEY_3
                echo "Vault unsealed."
              else
                echo "Vault is already unsealed."
              fi
            env:
            - name: UNSEAL_KEY_1
              valueFrom:
                secretKeyRef:
                  name: vault-unseal-keys
                  key: key1
            - name: UNSEAL_KEY_2
              valueFrom:
                secretKeyRef:
                  name: vault-unseal-keys
                  key: key2
            - name: UNSEAL_KEY_3
              valueFrom:
                secretKeyRef:
                  name: vault-unseal-keys
                  key: key3
          restartPolicy: OnFailure
```

```bash
# Apply the CronJob
kubectl apply -f vault-auto-unseal.yaml

# Manually trigger unseal now (optional)
kubectl create job --from=cronjob/vault-unseal vault-unseal-now -n vault
```

### Step 3: Verify

```bash
# Check Vault status
kubectl exec -n vault vault-0 -- vault status

# Should show: Sealed = false
```

**Security Note:** Unseal keys are stored in Kubernetes secrets. Anyone with access to the `vault` namespace can read them. For higher security, use cloud KMS or manual unsealing.

## Commands

```bash
# Run health check
./platform.sh health

# Auto-heal all issues
./platform.sh heal

# Fix specific component
./platform.sh heal gitlab
./platform.sh heal minio

# View saved fixes
cat .state/fixes.yaml

# Apply saved fixes on deploy
./platform.sh deploy all  # Automatically applies
```

## Persisted Fixes

Fixes are saved to `.state/fixes.yaml` and reapplied:

```yaml
fixes:
  - namespace: gitlab
    resource: gitlab-webservice
    type: memory
    value: 4Gi
    timestamp: 2026-01-15T10:30:00Z
  
  - namespace: monitoring
    resource: loki
    type: storage
    value: 20Gi
    timestamp: 2026-01-15T11:00:00Z
```