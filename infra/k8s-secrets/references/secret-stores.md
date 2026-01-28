# Secret Stores Configuration

## Vault SecretStore (Namespace-scoped)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
  namespace: myapp
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "myapp-role"
          serviceAccountRef:
            name: "myapp-sa"
```

## ClusterSecretStore (Cluster-wide)

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault.vault.svc.cluster.local:8200"
      path: "secret"
      version: "v2"
      namespace: "admin"  # Vault namespace (Enterprise)
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: "external-secrets"
            namespace: "external-secrets"
```

## Multiple Secret Stores

```yaml
# Production secrets from Vault
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-production
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "production"
      version: "v2"
      caBundle: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
      auth:
        tokenSecretRef:
          name: vault-token
          namespace: external-secrets
          key: token
---
# Development secrets from a different path
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-development
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "development"
      version: "v2"
      auth:
        tokenSecretRef:
          name: vault-token
          namespace: external-secrets
          key: token
```