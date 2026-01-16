# Database Monitoring

## PostgreSQL Metrics

```yaml
# Enable monitoring in PerconaPGCluster
spec:
  monitoring:
    pgmonitor:
      exporter:
        image: percona/postgres_exporter:0.15.0
```

## VMServiceScrape for PostgreSQL

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

## MongoDB ServiceMonitor

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: mongodb-metrics
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app.kubernetes.io/component: mongod
  endpoints:
    - port: metrics
      path: /metrics
```

## Grafana Dashboards

| Database | Dashboard ID |
|----------|-------------|
| PostgreSQL | 9628, 12485 |
| MongoDB | 2583, 7353 |

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
          annotations:
            summary: "PostgreSQL instance down"
        
        - alert: PostgreSQLReplicationLag
          expr: pg_replication_lag > 60
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL replication lag > 60s"
        
        - alert: PostgreSQLConnectionsHigh
          expr: pg_stat_activity_count / pg_settings_max_connections > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PostgreSQL connections > 80%"
    
    - name: mongodb
      rules:
        - alert: MongoDBDown
          expr: mongodb_up == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "MongoDB instance down"
        
        - alert: MongoDBReplicationLag
          expr: mongodb_replset_member_replication_lag > 60
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MongoDB replication lag > 60s"
```