# Helm Charts

Production-ready Helm charts for NestJS + React deployment.

## Chart Structure

```
helm/
└── myapp/
    ├── Chart.yaml
    ├── values.yaml
    ├── values-staging.yaml
    ├── values-production.yaml
    ├── templates/
    │   ├── _helpers.tpl
    │   ├── namespace.yaml
    │   ├── backend/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   ├── hpa.yaml
    │   │   ├── configmap.yaml
    │   │   ├── secret.yaml
    │   │   └── serviceaccount.yaml
    │   ├── frontend/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── hpa.yaml
    │   ├── database/
    │   │   ├── statefulset.yaml
    │   │   ├── service.yaml
    │   │   └── pvc.yaml
    │   ├── redis/
    │   │   ├── statefulset.yaml
    │   │   └── service.yaml
    │   ├── ingress.yaml
    │   └── cluster-issuer.yaml
    └── charts/             # Subcharts (optional)
```

---

## Chart.yaml

```yaml
# helm/myapp/Chart.yaml
apiVersion: v2
name: myapp
description: Full-stack NestJS + React application
type: application
version: 1.0.0
appVersion: "1.0.0"

keywords:
  - nestjs
  - react
  - fullstack
  - typescript

maintainers:
  - name: Your Name
    email: your@email.com

dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: "17.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

---

## values.yaml (Default)

```yaml
# helm/myapp/values.yaml

# Global settings
global:
  imageRegistry: ""
  imagePullSecrets: []
  storageClass: "hcloud-volumes"

# Namespace
namespace:
  create: true
  name: myapp

# ==================== BACKEND ====================
backend:
  enabled: true
  name: backend
  
  image:
    repository: registry.example.com/myapp/backend
    tag: latest
    pullPolicy: Always
  
  replicaCount: 2
  
  service:
    type: ClusterIP
    port: 80
    targetPort: 3000
  
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
  
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  
  # Environment configuration
  config:
    NODE_ENV: production
    APP_PORT: "3000"
    API_PREFIX: api
    DATABASE_TYPE: postgres
    DATABASE_PORT: "5432"
    DATABASE_SSL_ENABLED: "true"
    DATABASE_SYNCHRONIZE: "false"
  
  # Secrets (use external secrets in production!)
  secrets:
    DATABASE_HOST: postgres
    DATABASE_NAME: myapp
    DATABASE_USERNAME: myapp
    DATABASE_PASSWORD: ""  # Set via --set or external secrets
    AUTH_JWT_SECRET: ""
    AUTH_REFRESH_SECRET: ""
  
  # Health checks
  livenessProbe:
    enabled: true
    path: /api/health
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  
  readinessProbe:
    enabled: true
    path: /api/health
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3
  
  # Persistence for uploads
  persistence:
    enabled: true
    size: 10Gi
    accessMode: ReadWriteOnce
    storageClass: ""
  
  # Pod security
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    fsGroup: 1000
  
  containerSecurityContext:
    allowPrivilegeEscalation: false
    readOnlyRootFilesystem: true
    capabilities:
      drop:
        - ALL

# ==================== FRONTEND ====================
frontend:
  enabled: true
  name: frontend
  
  image:
    repository: registry.example.com/myapp/frontend
    tag: latest
    pullPolicy: Always
  
  replicaCount: 2
  
  service:
    type: ClusterIP
    port: 80
    targetPort: 3000
  
  resources:
    requests:
      cpu: 50m
      memory: 128Mi
    limits:
      cpu: 200m
      memory: 256Mi
  
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 70
  
  env:
    NEXT_PUBLIC_API_URL: https://api.example.com
  
  livenessProbe:
    enabled: true
    path: /
    initialDelaySeconds: 15
    periodSeconds: 10
  
  readinessProbe:
    enabled: true
    path: /
    initialDelaySeconds: 5
    periodSeconds: 5

# ==================== DATABASE ====================
postgresql:
  enabled: true
  auth:
    username: myapp
    password: ""  # Set via --set
    database: myapp
  primary:
    persistence:
      enabled: true
      size: 20Gi
      storageClass: "hcloud-volumes"
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi

# Use external database instead
externalDatabase:
  enabled: false
  host: ""
  port: 5432
  database: myapp
  username: myapp
  password: ""
  existingSecret: ""

# ==================== REDIS ====================
redis:
  enabled: true
  auth:
    enabled: true
    password: ""  # Set via --set
  master:
    persistence:
      enabled: true
      size: 5Gi
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
  replica:
    replicaCount: 0  # Disable replicas for cost savings

# ==================== INGRESS ====================
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
  
  hosts:
    - host: app.example.com
      paths:
        - path: /
          pathType: Prefix
          service: frontend
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
          service: backend
  
  tls:
    - secretName: myapp-tls
      hosts:
        - app.example.com
        - api.example.com

# ==================== CERT-MANAGER ====================
certManager:
  enabled: true
  email: admin@example.com
  staging: false  # Set to true for testing

# ==================== SERVICE ACCOUNT ====================
serviceAccount:
  create: true
  name: ""
  annotations: {}

# ==================== POD DISRUPTION BUDGET ====================
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# ==================== NETWORK POLICIES ====================
networkPolicies:
  enabled: false  # Enable for stricter security
```

---

## values-production.yaml

```yaml
# helm/myapp/values-production.yaml

backend:
  replicaCount: 3
  
  resources:
    requests:
      cpu: 200m
      memory: 512Mi
    limits:
      cpu: 1000m
      memory: 1Gi
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 20

frontend:
  replicaCount: 3
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10

postgresql:
  primary:
    persistence:
      size: 50Gi
    resources:
      requests:
        cpu: 500m
        memory: 1Gi
      limits:
        cpu: 2000m
        memory: 2Gi

redis:
  master:
    persistence:
      size: 10Gi

podDisruptionBudget:
  enabled: true
  minAvailable: 2

networkPolicies:
  enabled: true
```

---

## Template Helpers

```yaml
# helm/myapp/templates/_helpers.tpl
{{/*
Expand the name of the chart.
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "myapp.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Create chart name and version as used by the chart label.
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Backend labels
*/}}
{{- define "myapp.backend.labels" -}}
{{ include "myapp.labels" . }}
app.kubernetes.io/component: backend
{{- end }}

{{/*
Frontend labels
*/}}
{{- define "myapp.frontend.labels" -}}
{{ include "myapp.labels" . }}
app.kubernetes.io/component: frontend
{{- end }}

{{/*
Database host
*/}}
{{- define "myapp.databaseHost" -}}
{{- if .Values.postgresql.enabled }}
{{- printf "%s-postgresql" .Release.Name }}
{{- else }}
{{- .Values.externalDatabase.host }}
{{- end }}
{{- end }}

{{/*
Redis host
*/}}
{{- define "myapp.redisHost" -}}
{{- if .Values.redis.enabled }}
{{- printf "redis://%s-redis-master:6379" .Release.Name }}
{{- else }}
{{- .Values.externalRedis.host }}
{{- end }}
{{- end }}
```

---

## Backend Deployment Template

```yaml
# helm/myapp/templates/backend/deployment.yaml
{{- if .Values.backend.enabled -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}-backend
  labels:
    {{- include "myapp.backend.labels" . | nindent 4 }}
spec:
  {{- if not .Values.backend.autoscaling.enabled }}
  replicas: {{ .Values.backend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      app: backend
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
        {{- include "myapp.selectorLabels" . | nindent 8 }}
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/backend/configmap.yaml") . | sha256sum }}
        checksum/secret: {{ include (print $.Template.BasePath "/backend/secret.yaml") . | sha256sum }}
    spec:
      {{- with .Values.backend.securityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: backend
          image: "{{ .Values.backend.image.repository }}:{{ .Values.backend.image.tag }}"
          imagePullPolicy: {{ .Values.backend.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.backend.service.targetPort }}
              protocol: TCP
          envFrom:
            - configMapRef:
                name: {{ include "myapp.fullname" . }}-backend-config
            - secretRef:
                name: {{ include "myapp.fullname" . }}-backend-secrets
          env:
            - name: DATABASE_HOST
              value: {{ include "myapp.databaseHost" . }}
            - name: WORKER_HOST
              value: {{ include "myapp.redisHost" . }}
          {{- with .Values.backend.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- if .Values.backend.livenessProbe.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.backend.livenessProbe.path }}
              port: http
            initialDelaySeconds: {{ .Values.backend.livenessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.backend.livenessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.backend.livenessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.backend.livenessProbe.failureThreshold }}
          {{- end }}
          {{- if .Values.backend.readinessProbe.enabled }}
          readinessProbe:
            httpGet:
              path: {{ .Values.backend.readinessProbe.path }}
              port: http
            initialDelaySeconds: {{ .Values.backend.readinessProbe.initialDelaySeconds }}
            periodSeconds: {{ .Values.backend.readinessProbe.periodSeconds }}
            timeoutSeconds: {{ .Values.backend.readinessProbe.timeoutSeconds }}
            failureThreshold: {{ .Values.backend.readinessProbe.failureThreshold }}
          {{- end }}
          {{- with .Values.backend.containerSecurityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            {{- if .Values.backend.persistence.enabled }}
            - name: uploads
              mountPath: /app/uploads
            {{- end }}
      volumes:
        - name: tmp
          emptyDir: {}
        {{- if .Values.backend.persistence.enabled }}
        - name: uploads
          persistentVolumeClaim:
            claimName: {{ include "myapp.fullname" . }}-backend-uploads
        {{- end }}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: backend
                topologyKey: kubernetes.io/hostname
{{- end }}
```

---

## Ingress Template

```yaml
# helm/myapp/templates/ingress.yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  ingressClassName: {{ .Values.ingress.className }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "myapp.fullname" $ }}-{{ .service }}
                port:
                  number: 80
          {{- end }}
    {{- end }}
{{- end }}
```

---

## Helm Commands

```bash
# Add Bitnami repo for dependencies
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Update dependencies
cd helm/myapp
helm dependency update

# Install (development)
helm install myapp ./helm/myapp \
  --namespace myapp \
  --create-namespace \
  --set postgresql.auth.password=secretpassword \
  --set redis.auth.password=secretpassword \
  --set backend.secrets.AUTH_JWT_SECRET=jwtsecret \
  --set backend.secrets.DATABASE_PASSWORD=secretpassword

# Install (production)
helm install myapp ./helm/myapp \
  --namespace myapp-prod \
  --create-namespace \
  -f helm/myapp/values-production.yaml \
  --set-file backend.secrets=secrets/production.yaml

# Upgrade
helm upgrade myapp ./helm/myapp \
  --namespace myapp \
  --reuse-values \
  --set backend.image.tag=v1.1.0 \
  --set frontend.image.tag=v1.1.0

# Rollback
helm rollback myapp 1 --namespace myapp

# Uninstall
helm uninstall myapp --namespace myapp

# Template (dry-run)
helm template myapp ./helm/myapp --debug

# Lint
helm lint ./helm/myapp
```

---

## Cost Estimation (Hetzner)

| Component | Specs | Monthly Cost |
|-----------|-------|-------------|
| 3x CX21 Workers | 2 vCPU, 4GB RAM | €15.48 (€5.16 each) |
| 1x CX11 Master | 1 vCPU, 2GB RAM | €3.29 |
| Load Balancer | LB11 | €5.39 |
| 20GB Volume (DB) | Block storage | €0.96 |
| 10GB Volume (Redis) | Block storage | €0.48 |
| **Total** | | **~€25-30/month** |