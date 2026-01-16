# GitLab CI Integration

## Complete CI/CD Pipeline

```yaml
# .gitlab-ci.yml
variables:
  DOCKER_TLS_CERTDIR: "/certs"
  IMAGE_NAME: $CI_REGISTRY_IMAGE
  MANIFESTS_REPO: "myorg/myapp-manifests"
  ARGOCD_SERVER: "argocd.example.com"

stages:
  - test
  - build
  - publish
  - deploy

# ==================== TEST ====================
.test_template:
  stage: test
  image: node:20-alpine
  cache:
    key: ${CI_JOB_NAME}-${CI_COMMIT_REF_SLUG}
    paths:
      - node_modules/
  before_script:
    - npm ci --prefer-offline

test:backend:
  extends: .test_template
  script:
    - cd backend
    - npm ci
    - npm run lint
    - npm run test:cov
  coverage: '/All files[^|]*\|[^|]*\s+([\d\.]+)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: backend/coverage/cobertura-coverage.xml
    expire_in: 1 week
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - backend/**/*
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
      changes:
        - backend/**/*

test:frontend:
  extends: .test_template
  script:
    - cd frontend
    - npm ci
    - npm run lint
    - npm run test -- --passWithNoTests
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - frontend/**/*
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
      changes:
        - frontend/**/*

# ==================== BUILD ====================
.build_template:
  stage: build
  image: docker:26
  services:
    - docker:26-dind
  variables:
    DOCKER_HOST: tcp://docker:2376
    DOCKER_DRIVER: overlay2
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY

build:backend:
  extends: .build_template
  script:
    - docker build 
        --build-arg NODE_ENV=production
        --cache-from $IMAGE_NAME/backend:latest
        -t $IMAGE_NAME/backend:$CI_COMMIT_SHA
        -t $IMAGE_NAME/backend:$CI_COMMIT_REF_SLUG
        ./backend
    - docker push $IMAGE_NAME/backend:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME/backend:$CI_COMMIT_REF_SLUG
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE_NAME/backend:$CI_COMMIT_SHA $IMAGE_NAME/backend:latest
        docker push $IMAGE_NAME/backend:latest
      fi
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
      changes:
        - backend/**/*
        - .gitlab-ci.yml

build:frontend:
  extends: .build_template
  script:
    - docker build
        --build-arg NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL
        --cache-from $IMAGE_NAME/frontend:latest
        -t $IMAGE_NAME/frontend:$CI_COMMIT_SHA
        -t $IMAGE_NAME/frontend:$CI_COMMIT_REF_SLUG
        ./frontend
    - docker push $IMAGE_NAME/frontend:$CI_COMMIT_SHA
    - docker push $IMAGE_NAME/frontend:$CI_COMMIT_REF_SLUG
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $IMAGE_NAME/frontend:$CI_COMMIT_SHA $IMAGE_NAME/frontend:latest
        docker push $IMAGE_NAME/frontend:latest
      fi
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'
      changes:
        - frontend/**/*
        - .gitlab-ci.yml

# ==================== UPDATE MANIFESTS ====================
update:manifests:
  stage: publish
  image: alpine:3.20
  before_script:
    - apk add --no-cache git curl yq
  script:
    - |
      # Clone manifests repo
      git clone https://oauth2:${GITLAB_TOKEN}@gitlab.example.com/${MANIFESTS_REPO}.git manifests
      cd manifests
      
      # Determine environment
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        ENV="production"
      else
        ENV="staging"
      fi
      
      # Update image tags using yq
      yq -i ".images[0].newTag = \"${CI_COMMIT_SHA}\"" overlays/${ENV}/kustomization.yaml
      yq -i ".images[1].newTag = \"${CI_COMMIT_SHA}\"" overlays/${ENV}/kustomization.yaml
      
      # Commit and push
      git config user.email "gitlab-ci@example.com"
      git config user.name "GitLab CI"
      git add .
      git diff --staged --quiet || git commit -m "chore: update ${ENV} images to ${CI_COMMIT_SHA}
      
      Source commit: ${CI_PROJECT_URL}/-/commit/${CI_COMMIT_SHA}
      Pipeline: ${CI_PIPELINE_URL}"
      git push origin main
  rules:
    - if: '$CI_COMMIT_BRANCH == "main" || $CI_COMMIT_BRANCH == "develop"'

# ==================== DEPLOY ====================
.deploy_template:
  stage: deploy
  image: argoproj/argocd:v2.11.3
  script:
    - argocd login $ARGOCD_SERVER --username admin --password $ARGOCD_PASSWORD --insecure --grpc-web
    - argocd app sync $APP_NAME --prune --force
    - argocd app wait $APP_NAME --timeout 300

deploy:staging:
  extends: .deploy_template
  variables:
    APP_NAME: myapp-staging
  environment:
    name: staging
    url: https://staging.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'

deploy:production:
  extends: .deploy_template
  variables:
    APP_NAME: myapp-production
  environment:
    name: production
    url: https://app.example.com
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
  when: manual
  allow_failure: false
```

## Required GitLab CI Variables

Settings → CI/CD → Variables:

| Variable | Type | Description |
|----------|------|-------------|
| `GITLAB_TOKEN` | Variable (masked) | PAT with write access to manifests repo |
| `ARGOCD_PASSWORD` | Variable (masked) | ArgoCD admin password |
| `NEXT_PUBLIC_API_URL` | Variable | Frontend API URL |

## ArgoCD Image Updater (Alternative to CI)

```bash
# Install Image Updater
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/v0.14.0/manifests/install.yaml
```

```yaml
# Application with Image Updater annotations
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd
  annotations:
    argocd-image-updater.argoproj.io/image-list: |
      backend=registry.gitlab.example.com/myorg/myapp/backend
      frontend=registry.gitlab.example.com/myorg/myapp/frontend
    argocd-image-updater.argoproj.io/backend.update-strategy: latest
    argocd-image-updater.argoproj.io/frontend.update-strategy: latest
    argocd-image-updater.argoproj.io/write-back-method: git
    argocd-image-updater.argoproj.io/git-branch: main
spec:
  # ... rest of spec
```