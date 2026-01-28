# Repository Structure

## Recommended Structure

```
# Application Repository (myapp)
myapp/
├── .gitlab-ci.yml
├── backend/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
├── frontend/
│   ├── Dockerfile
│   ├── src/
│   └── package.json
└── README.md

# Manifests Repository (myapp-manifests)
myapp-manifests/
├── base/
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── hpa.yaml
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── routes/
│       ├── gateway.yaml
│       └── httproutes.yaml
├── overlays/
│   ├── staging/
│   │   ├── kustomization.yaml
│   │   └── patches/
│   └── production/
│       ├── kustomization.yaml
│       └── patches/
└── applicationsets/
    ├── myapp.yaml
    └── project.yaml
```

## Kustomization Examples

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp

resources:
  - namespace.yaml
  - backend/deployment.yaml
  - backend/service.yaml
  - backend/hpa.yaml
  - frontend/deployment.yaml
  - frontend/service.yaml
  - routes/httproutes.yaml

images:
  - name: backend
    newName: registry.example.com/myapp/backend
    newTag: latest
  - name: frontend
    newName: registry.example.com/myapp/frontend
    newTag: latest

---
# overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: myapp-prod

resources:
  - ../../base

images:
  - name: backend
    newName: registry.example.com/myapp/backend
    newTag: v1.2.3  # Updated by CI
  - name: frontend
    newName: registry.example.com/myapp/frontend
    newTag: v1.2.3

patches:
  - path: patches/replicas.yaml
  - path: patches/resources.yaml

configMapGenerator:
  - name: app-config
    behavior: merge
    literals:
      - NODE_ENV=production
```