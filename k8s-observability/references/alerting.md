# Alerting Configuration

## VMRule for Infrastructure Alerts

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: infrastructure-alerts
  namespace: monitoring
spec:
  groups:
    - name: node
      rules:
        - alert: NodeHighCPU
          expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High CPU on {{ $labels.instance }}"
            description: "CPU usage is {{ $value | printf \"%.1f\" }}%"
        
        - alert: NodeHighMemory
          expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory on {{ $labels.instance }}"
            description: "Memory usage is {{ $value | printf \"%.1f\" }}%"
        
        - alert: NodeDiskFull
          expr: (1 - node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"}) * 100 > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Disk almost full on {{ $labels.instance }}"
        
        - alert: NodeDown
          expr: up{job="node-exporter"} == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "Node {{ $labels.instance }} is down"

    - name: kubernetes
      rules:
        - alert: PodCrashLooping
          expr: rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 0
          for: 15m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} crash looping"
        
        - alert: PodNotReady
          expr: kube_pod_status_ready{condition="true"} == 0
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} not ready"
        
        - alert: DeploymentReplicasMismatch
          expr: kube_deployment_spec_replicas != kube_deployment_status_replicas_available
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} replica mismatch"
        
        - alert: PVCAlmostFull
          expr: (kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes) * 100 > 80
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "PVC {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} almost full"

    - name: application
      rules:
        - alert: HighErrorRate
          expr: |
            sum(rate(http_requests_total{status=~"5.."}[5m])) by (namespace, service)
            / sum(rate(http_requests_total[5m])) by (namespace, service) > 0.05
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "High error rate in {{ $labels.namespace }}/{{ $labels.service }}"
            description: "Error rate is {{ $value | humanizePercentage }}"
        
        - alert: HighLatency
          expr: |
            histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, namespace, service)) > 1
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High latency in {{ $labels.namespace }}/{{ $labels.service }}"
            description: "P95 latency is {{ $value | humanizeDuration }}"
```

## VMAlertmanagerConfig

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMAlertmanagerConfig
metadata:
  name: main-config
  namespace: monitoring
spec:
  route:
    receiver: default
    group_by:
      - alertname
      - namespace
    group_wait: 30s
    group_interval: 5m
    repeat_interval: 4h
    routes:
      - receiver: critical-slack
        match:
          severity: critical
      - receiver: warning-slack
        match:
          severity: warning
  
  receivers:
    - name: default
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-webhook
            key: url
          channel: '#alerts'
          sendResolved: true
          title: '{{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
          text: |
            {{ range .Alerts }}
            *Alert:* {{ .Annotations.summary }}
            *Severity:* {{ .Labels.severity }}
            *Description:* {{ .Annotations.description }}
            {{ end }}
    
    - name: critical-slack
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-webhook
            key: url
          channel: '#alerts-critical'
          sendResolved: true
          color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
          title: '🚨 {{ .Status | toUpper }}: {{ .CommonLabels.alertname }}'
    
    - name: warning-slack
      slackConfigs:
        - apiURL:
            name: alertmanager-slack-webhook
            key: url
          channel: '#alerts-warning'
          sendResolved: true
```

## Slack Webhook Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-slack-webhook
  namespace: monitoring
type: Opaque
stringData:
  url: "<your-slack-webhook-url>"  # Get from: Slack App > Incoming Webhooks
```