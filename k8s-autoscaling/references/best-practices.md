# Autoscaling Best Practices

## 1. Set Proper Resource Requests

```yaml
# HPA calculates utilization based on REQUESTS!
# If requests are too low, pods will scale too early
# If requests are too high, pods won't scale when needed

resources:
  requests:
    cpu: 100m      # Set based on actual steady-state usage
    memory: 256Mi
  limits:
    cpu: 500m      # 5x headroom for spikes
    memory: 512Mi  # 2x headroom
```

## 2. Use Stabilization Windows

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300  # Prevent flapping
    policies:
      - type: Percent
        value: 10            # Max 10% scale down
        periodSeconds: 60    # Per minute
  scaleUp:
    stabilizationWindowSeconds: 0  # Scale up immediately
    policies:
      - type: Percent
        value: 100           # Can double
        periodSeconds: 15    # Every 15s
```

## 3. Use Multiple Metrics

```yaml
# Don't rely on just CPU
metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Add memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Add application metric
  - type: External
    external:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

## 4. Set Appropriate Min/Max

```yaml
spec:
  minReplicas: 2   # Always have 2 for HA
  maxReplicas: 20  # Cap based on budget/capacity
```

## 5. Test Your Scaling

```bash
# Load test to verify scaling
kubectl run -it --rm load-generator --image=busybox -- /bin/sh -c \
  "while true; do wget -q -O- http://backend.myapp.svc/api/health; done"

# Watch HPA
kubectl get hpa -n myapp -w

# Watch pods
kubectl get pods -n myapp -w
```

## 6. Monitor Scaling Events

```yaml
# VMRule for autoscaling alerts
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMRule
metadata:
  name: autoscaling-alerts
  namespace: monitoring
spec:
  groups:
    - name: autoscaling
      rules:
        - alert: HPAMaxedOut
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas 
            == kube_horizontalpodautoscaler_spec_max_replicas
          for: 15m
          labels:
            severity: warning
          annotations:
            summary: "HPA {{ $labels.horizontalpodautoscaler }} at max"
        
        - alert: ScaledObjectError
          expr: keda_scaledobject_errors > 0
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "KEDA ScaledObject error"
```

## 7. Don't Autoscale Databases

```bash
# For databases, use operator-managed scaling instead
kubectl patch perconapgcluster myapp-pg -n databases --type=merge \
  -p '{"spec":{"instances":[{"name":"instance1","replicas":5}]}}'
```