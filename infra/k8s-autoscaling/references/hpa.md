# Horizontal Pod Autoscaler (HPA)

## Basic CPU/Memory HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
    # CPU utilization
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Memory utilization
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
  
  # Scaling behavior (prevents flapping)
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scale down
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60  # Scale down max 10% per minute
    scaleUp:
      stabilizationWindowSeconds: 0    # Scale up immediately
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15  # Can double every 15s
        - type: Pods
          value: 4
          periodSeconds: 15  # Or add 4 pods every 15s
      selectPolicy: Max  # Use whichever adds more pods
```

## HPA with Custom Metrics (VictoriaMetrics)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-custom-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
    # CPU as baseline
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Custom metric: requests per second per pod
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"  # Scale when > 100 RPS per pod
    
    # External metric: queue length
    - type: External
      external:
        metric:
          name: redis_queue_length
          selector:
            matchLabels:
              queue: tasks
        target:
          type: AverageValue
          averageValue: "30"  # Scale when > 30 items per pod
```

## Check HPA Status

```bash
# View HPA
kubectl get hpa -n myapp

# Detailed status
kubectl describe hpa backend-hpa -n myapp

# Watch scaling events
kubectl get events -n myapp --field-selector reason=SuccessfulRescale -w

# Debug metrics
kubectl get --raw "/apis/metrics.k8s.io/v1beta1/namespaces/myapp/pods" | jq
```

## Resource Requests are Critical!

```yaml
# HPA calculates utilization based on REQUESTS, not limits!
resources:
  requests:
    cpu: 100m      # HPA 70% = 70m actual usage triggers scaling
    memory: 256Mi  # HPA 80% = 200Mi actual usage triggers scaling
  limits:
    cpu: 500m
    memory: 512Mi
```