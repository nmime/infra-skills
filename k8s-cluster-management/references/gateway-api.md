# Cilium Gateway API

Modern Kubernetes ingress using Gateway API with Cilium.

## Version Information (January 2026)

| Component | Version |
|-----------|---------|
| Gateway API | v1.4.0 |
| Cilium | v1.18.6 |

## Install Gateway API CRDs

```bash
# Standard channel (stable)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/standard-install.yaml

# Experimental channel (TCPRoute, TLSRoute, GRPCRoute)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.4.0/experimental-install.yaml

# Verify
kubectl get crd | grep gateway
```

## Production Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: cilium-system
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      protocol: HTTP
      port: 80
      allowedRoutes:
        namespaces:
          from: All
    - name: https-wildcard
      protocol: HTTPS
      port: 443
      hostname: "*.example.com"
      tls:
        mode: Terminate
        certificateRefs:
          - kind: Secret
            name: wildcard-tls
            namespace: cilium-secrets
      allowedRoutes:
        namespaces:
          from: All
```

## HTTPRoute Example

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-route
  namespace: myapp
spec:
  parentRefs:
    - name: main-gateway
      namespace: cilium-system
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: frontend
          port: 80
```