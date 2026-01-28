# MinIO Monitoring

## VMServiceScrape for VictoriaMetrics

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: minio
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: minio
  namespaceSelector:
    matchNames:
      - minio
  endpoints:
    - port: http
      path: /minio/v2/metrics/cluster
      interval: 30s
```

## Key Metrics

| Metric | Description |
|--------|-------------|
| `minio_cluster_capacity_usable_total_bytes` | Total usable capacity |
| `minio_cluster_capacity_usable_free_bytes` | Free capacity |
| `minio_bucket_usage_total_bytes` | Bucket size |
| `minio_bucket_objects_count` | Object count |
| `minio_s3_requests_total` | S3 requests |
| `minio_s3_requests_errors_total` | S3 errors |
| `minio_s3_traffic_received_bytes` | Ingress traffic |
| `minio_s3_traffic_sent_bytes` | Egress traffic |

## Grafana Dashboard

Import dashboard ID: **13502** (MinIO Dashboard)

## VMRule Alerts

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: minio-alerts
  namespace: monitoring
spec:
  groups:
    - name: minio
      rules:
        - alert: MinIOClusterDiskUsageHigh
          expr: |
            (1 - minio_cluster_capacity_usable_free_bytes / minio_cluster_capacity_usable_total_bytes) > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MinIO disk usage above 80%"
        
        - alert: MinIONodeDown
          expr: minio_cluster_nodes_offline_total > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "MinIO node offline"
        
        - alert: MinIOHighErrorRate
          expr: |
            rate(minio_s3_requests_errors_total[5m]) 
            / rate(minio_s3_requests_total[5m]) > 0.05
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "MinIO error rate above 5%"
```