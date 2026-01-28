# KEDA Scalers

KEDA supports 70+ scalers. Most useful ones:

## Prometheus/VictoriaMetrics

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-prometheus
  namespace: myapp
spec:
  scaleTargetRef:
    name: backend
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://vmselect-vmcluster.monitoring.svc:8481/select/0/prometheus
        metricName: http_requests_per_second
        threshold: "100"
        query: sum(rate(http_requests_total{namespace="myapp"}[2m]))
```

## Redis Queue

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-redis
  namespace: myapp
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 0
  maxReplicaCount: 20
  triggers:
    - type: redis
      metadata:
        address: redis.myapp.svc.cluster.local:6379
        listName: job_queue
        listLength: "10"
      authenticationRef:
        name: redis-auth
```

## PostgreSQL

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: worker-postgres
  namespace: myapp
spec:
  scaleTargetRef:
    name: worker
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    - type: postgresql
      metadata:
        targetQueryValue: "50"
        query: "SELECT COUNT(*) FROM jobs WHERE status = 'pending'"
        connectionFromEnv: DATABASE_URL
```

## Cron (Scheduled)

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: backend-cron
  namespace: myapp
spec:
  scaleTargetRef:
    name: backend
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    - type: cron
      metadata:
        timezone: Europe/Berlin
        start: 0 8 * * 1-5
        end: 0 18 * * 1-5
        desiredReplicas: "5"
```