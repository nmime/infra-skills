# PostgreSQL Single Instance

For minimal/small tiers. Percona PostgreSQL Operator v2.8.2.

## Single Instance Cluster

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: postgresql
  namespace: databases
spec:
  crVersion: "2.8.2"
  postgresVersion: 18

  instances:
    - name: instance1
      replicas: 1
      resources:
        requests:
          cpu: 100m
          memory: 256Mi
        limits:
          cpu: 500m
          memory: 512Mi
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 10Gi

  users:
    - name: app
      databases:
        - app
    - name: gitlab
      databases:
        - gitlabhq_production

  proxy:
    pgBouncer:
      replicas: 1
      resources:
        requests:
          cpu: 25m
          memory: 32Mi
      config:
        global:
          pool_mode: session
          max_client_conn: "100"

  backups:
    pgbackrest:
      repos:
        - name: repo1
          volume:
            volumeClaimSpec:
              storageClassName: hcloud-volumes
              accessModes:
                - ReadWriteOnce
              resources:
                requests:
                  storage: 5Gi
```

## Get Connection

```bash
kubectl get secret postgresql-pguser-app -n databases \
  -o jsonpath='{.data.uri}' | base64 -d
```

## Resource Summary

| Component | CPU | Memory |
|-----------|-----|--------|
| Operator | 50m | 64Mi |
| PostgreSQL | 100-500m | 256-512Mi |
| PgBouncer | 25m | 32Mi |
| **Total** | **~175-575m** | **~350-600Mi** |
