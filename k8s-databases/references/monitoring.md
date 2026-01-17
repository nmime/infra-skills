# Database Monitoring with PMM

Percona Monitoring and Management (PMM) for database observability.

## PMM Server (Helm)

### Create Secret

```bash
kubectl create namespace monitoring

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: pmm-secret
  namespace: monitoring
  labels:
    app.kubernetes.io/name: pmm
type: Opaque
data:
  PMM_ADMIN_PASSWORD: $(echo -n "your-secure-password" | base64)
EOF
```

### Install PMM Server

```bash
helm repo add percona https://percona.github.io/percona-helm-charts
helm repo update

# Check available versions
helm search repo percona/pmm --versions

# Install PMM Server
helm upgrade --install pmm percona/pmm \
  --namespace monitoring \
  --set secret.create=false \
  --set secret.name=pmm-secret \
  --set service.type=ClusterIP \
  --set storage.storageClassName=hcloud-volumes \
  --set storage.size=20Gi \
  --version 1.4.8
```

### Verify Installation

```bash
helm list -n monitoring
kubectl get pods -n monitoring -l app.kubernetes.io/name=pmm
```

## Gateway HTTPRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: pmm
  namespace: monitoring
spec:
  parentRefs:
    - name: main-gateway
      namespace: gateway
  hostnames:
    - pmm.example.com
  rules:
    - backendRefs:
        - name: pmm
          port: 443
```

## PostgreSQL with PMM

```yaml
apiVersion: pgv2.percona.com/v2
kind: PerconaPGCluster
metadata:
  name: myapp-pg
  namespace: databases
spec:
  crVersion: "2.8.2"
  postgresVersion: 18

  instances:
    - name: instance1
      replicas: 3
      dataVolumeClaimSpec:
        storageClassName: hcloud-volumes
        resources:
          requests:
            storage: 20Gi

  pmm:
    enabled: true
    image: percona/pmm-client:2.44.0
    serverHost: pmm.monitoring.svc.cluster.local
    secret: pmm-secret
```

## MongoDB with PMM

```yaml
apiVersion: psmdb.percona.com/v1
kind: PerconaServerMongoDB
metadata:
  name: myapp-mongo
  namespace: databases
spec:
  crVersion: "1.21.2"
  image: percona/percona-server-mongodb:8.0.17-6

  replsets:
    - name: rs0
      size: 3
      volumeSpec:
        persistentVolumeClaim:
          storageClassName: hcloud-volumes
          resources:
            requests:
              storage: 20Gi

  pmm:
    enabled: true
    image: percona/pmm-client:2.44.0
    serverHost: pmm.monitoring.svc.cluster.local
```

## PMM Secret for Databases

```bash
kubectl create secret generic pmm-secret -n databases \
  --from-literal=PMM_USER=admin \
  --from-literal=PMM_PASSWORD="your-secure-password"
```

## PMM Features

| Feature | Description |
|---------|-------------|
| Query Analytics (QAN) | Analyze slow queries |
| Metrics Monitor | Historical metrics with Grafana |
| Alerting | Built-in alert rules |
| Advisors | Performance recommendations |

## Access PMM

```bash
# Port forward for local access
kubectl port-forward svc/pmm -n monitoring 8443:443

# Or via Gateway
https://pmm.example.com
```

User: `admin`
Password: Value from `pmm-secret`

## Helm Values Reference

```yaml
# values.yaml
secret:
  create: false
  name: pmm-secret

service:
  type: ClusterIP

storage:
  storageClassName: hcloud-volumes
  size: 20Gi

# Disable telemetry
pmmEnv:
  DISABLE_TELEMETRY: "1"

# Resource limits
resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi
```

## VictoriaMetrics Alternative

If using VictoriaMetrics instead of PMM:

```yaml
# PostgreSQL exporter
spec:
  monitoring:
    pgmonitor:
      exporter:
        image: percona/postgres_exporter:0.15.0
```

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: postgres-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      postgres-operator.crunchydata.com/cluster: myapp-pg
  endpoints:
    - port: exporter
      path: /metrics
```

## Alert Rules

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: database-alerts
  namespace: monitoring
spec:
  groups:
    - name: postgresql
      rules:
        - alert: PostgreSQLDown
          expr: pg_up == 0
          for: 1m
          labels:
            severity: critical
        - alert: PostgreSQLReplicationLag
          expr: pg_replication_lag > 60
          for: 5m
          labels:
            severity: warning

    - name: mongodb
      rules:
        - alert: MongoDBDown
          expr: mongodb_up == 0
          for: 1m
          labels:
            severity: critical
        - alert: MongoDBReplicationLag
          expr: mongodb_replset_member_replication_lag > 60
          for: 5m
          labels:
            severity: warning
```
