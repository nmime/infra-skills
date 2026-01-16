# Secrets Best Practices

## Vault Policies

```hcl
# Least privilege policy
path "secret/data/myapp/*" {
  capabilities = ["read"]
}

# Admin policy
path "secret/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

## Secret Rotation

```yaml
# Auto-refresh secrets every hour
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: rotating-secret
spec:
  refreshInterval: 1h  # Refresh every hour
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: rotating-secret
  data:
    - secretKey: api-key
      remoteRef:
        key: secret/data/myapp/api
        property: key
```

## GitOps Safe Secrets

```yaml
# ExternalSecret is safe to commit to Git
# Only references, no actual secrets
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: myapp
spec:
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: myapp-secrets
    template:
      type: Opaque
      data:
        DATABASE_URL: "postgresql://{{ .db_user }}:{{ .db_pass }}@postgres:5432/myapp"
  data:
    - secretKey: db_user
      remoteRef:
        key: secret/data/myapp/database
        property: username
    - secretKey: db_pass
      remoteRef:
        key: secret/data/myapp/database
        property: password
```

## Monitoring

```yaml
# Alert on secret sync failures
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: external-secrets-alerts
spec:
  groups:
    - name: external-secrets
      rules:
        - alert: ExternalSecretSyncFailed
          expr: external_secrets_sync_calls_error > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "External Secret sync failed"
```