# ApplicationSets

Multi-environment and multi-cluster deployments.

## Git Generator (Directory-based)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]
  generators:
    - git:
        repoURL: https://gitlab.example.com/myorg/manifests.git
        revision: main
        directories:
          - path: overlays/*
  template:
    metadata:
      name: 'myapp-{{.path.basename}}'
      namespace: argocd
      labels:
        app.kubernetes.io/name: myapp
        environment: '{{.path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://gitlab.example.com/myorg/manifests.git
        targetRevision: main
        path: '{{.path.path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: 'myapp-{{.path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
          - ServerSideApply=true
```

## List Generator (Explicit Environments)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-list
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: staging
            namespace: myapp-staging
            url: https://staging.example.com
            replicas: "2"
          - env: production
            namespace: myapp-prod
            url: https://app.example.com
            replicas: "5"
  template:
    metadata:
      name: 'myapp-{{.env}}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://gitlab.example.com/myorg/manifests.git
        targetRevision: main
        path: 'overlays/{{.env}}'
        kustomize:
          commonAnnotations:
            app.example.com/environment: '{{.env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

## Matrix Generator (Environments x Services)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices-matrix
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - matrix:
        generators:
          # Environments
          - list:
              elements:
                - env: staging
                  namespace: myapp-staging
                - env: production
                  namespace: myapp-prod
          # Services
          - list:
              elements:
                - service: backend
                  port: "3000"
                - service: frontend
                  port: "80"
                - service: worker
                  port: "8080"
  template:
    metadata:
      name: '{{.service}}-{{.env}}'
      namespace: argocd
      labels:
        service: '{{.service}}'
        environment: '{{.env}}'
    spec:
      project: default
      source:
        repoURL: https://gitlab.example.com/myorg/manifests.git
        targetRevision: main
        path: 'services/{{.service}}/overlays/{{.env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

## Progressive Rollout Strategy

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: progressive-rollout
  namespace: argocd
spec:
  goTemplate: true
  generators:
    - list:
        elements:
          - env: canary
            weight: "10"
            order: "1"
          - env: production
            weight: "90"
            order: "2"
  strategy:
    type: RollingSync
    rollingSync:
      steps:
        - matchExpressions:
            - key: env
              operator: In
              values:
                - canary
        - matchExpressions:
            - key: env
              operator: In
              values:
                - production
  template:
    metadata:
      name: 'myapp-{{.env}}'
      labels:
        env: '{{.env}}'
    spec:
      project: default
      source:
        repoURL: https://gitlab.example.com/myorg/manifests.git
        targetRevision: main
        path: 'overlays/{{.env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: myapp
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```