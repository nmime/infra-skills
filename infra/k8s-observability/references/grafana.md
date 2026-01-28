# Grafana Configuration

## Version Information (Latest - January 2026)

| Component | Version |
|-----------|---------|
| Grafana | 12.4.0 |
| Grafana Helm Chart | 8.10.0 |

## Gateway API Route

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: grafana
  namespace: monitoring
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "grafana.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: grafana
          port: 80
```

## Recommended Dashboards

| Dashboard | ID | Description |
|-----------|-----|-------------|
| Node Exporter Full | 1860 | Node metrics |
| Kubernetes Cluster | 7249 | Cluster overview |
| VictoriaMetrics Cluster | 11176 | VM metrics |
| Loki Dashboard | 13639 | Loki stats |
| Cilium | 16611 | Cilium/Hubble |
| PostgreSQL | 9628 | PostgreSQL |
| MongoDB | 2583 | MongoDB |
| ArgoCD | 14584 | ArgoCD |