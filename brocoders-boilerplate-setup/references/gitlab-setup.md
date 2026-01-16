# GitLab Cloud Setup Guide

## Prerequisites

- GitLab.com account
- Git installed locally
- GitLab CLI (optional): `brew install glab` or [download](https://gitlab.com/gitlab-org/cli)

## Option A: Monorepo Setup

### 1. Create GitLab Project

**Using GitLab CLI:**
```bash
glab repo create my-fullstack-app --private
```

**Using Web Interface:**
1. Go to https://gitlab.com/projects/new
2. Click "Create blank project"
3. Project name: `my-fullstack-app`
4. Visibility: Private/Public
5. Uncheck "Initialize repository with a README"
6. Click "Create project"

### 2. Initialize Local Repository

```bash
# In project root (containing backend/ and frontend/)
git init

# Create root .gitignore
cat > .gitignore << 'EOF'
# Dependencies
node_modules/

# Environment files
.env
.env.local
.env.*.local

# Build outputs
dist/
build/
.next/

# IDE
.idea/
.vscode/
*.swp

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Docker
data/
EOF

# Create root package.json (optional workspace setup)
cat > package.json << 'EOF'
{
  "name": "my-fullstack-app",
  "private": true,
  "workspaces": ["backend", "frontend"],
  "scripts": {
    "backend:dev": "npm run start:dev --workspace=backend",
    "frontend:dev": "npm run dev --workspace=frontend"
  }
}
EOF

# Initial commit
git add .
git commit -m "Initial commit: NestJS + React boilerplate setup"

# Add remote and push
git remote add origin https://gitlab.com/YOUR_USERNAME/my-fullstack-app.git
git branch -M main
git push -u origin main
```

### 3. GitLab CI/CD Pipeline

Create `.gitlab-ci.yml` in project root:

```yaml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"

# Cache configuration
.node_cache: &node_cache
  cache:
    key:
      files:
        - backend/package-lock.json
        - frontend/package-lock.json
    paths:
      - backend/node_modules/
      - frontend/node_modules/
    policy: pull-push

# ==================== BACKEND ====================
backend:lint:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd backend
    - npm ci
    - npm run lint
  only:
    changes:
      - backend/**/*

backend:test:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  services:
    - postgres:15
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
    DATABASE_URL: "postgresql://postgres:postgres@postgres:5432/test"
  script:
    - cd backend
    - npm ci
    - npm run test
  only:
    changes:
      - backend/**/*

backend:build:
  stage: build
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd backend
    - npm ci
    - npm run build
  artifacts:
    paths:
      - backend/dist/
    expire_in: 1 hour
  only:
    changes:
      - backend/**/*

# ==================== FRONTEND ====================
frontend:lint:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd frontend
    - npm ci
    - npm run lint
  only:
    changes:
      - frontend/**/*

frontend:test:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd frontend
    - npm ci
    - npm run test --passWithNoTests
  only:
    changes:
      - frontend/**/*

frontend:build:
  stage: build
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd frontend
    - npm ci
    - npm run build
  artifacts:
    paths:
      - frontend/dist/
      - frontend/.next/
    expire_in: 1 hour
  only:
    changes:
      - frontend/**/*

# ==================== DEPLOY ====================
deploy:staging:
  stage: deploy
  image: alpine:latest
  script:
    - echo "Deploy to staging environment"
    # Add your deployment commands here
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop
  when: manual

deploy:production:
  stage: deploy
  image: alpine:latest
  script:
    - echo "Deploy to production environment"
    # Add your deployment commands here
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual
```

### 4. Protected Branches

Settings → Repository → Protected branches:
- Branch: `main`
- Allowed to merge: Maintainers
- Allowed to push: No one
- Require approval: Yes

---

## Option B: Polyrepo Setup

### Backend Repository

```bash
cd backend
git init
git add .
git commit -m "Initial commit: NestJS boilerplate"

# Create project via CLI or web
glab repo create my-app-backend --private
git remote add origin https://gitlab.com/YOUR_USERNAME/my-app-backend.git
git branch -M main
git push -u origin main
```

### Frontend Repository

```bash
cd frontend
git init
git add .
git commit -m "Initial commit: React boilerplate"

glab repo create my-app-frontend --private
git remote add origin https://gitlab.com/YOUR_USERNAME/my-app-frontend.git
git branch -M main
git push -u origin main
```

---

## SSH vs HTTPS Authentication

### Using SSH (Recommended)

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your_email@example.com"

# Add to GitLab
# Copy content of ~/.ssh/id_ed25519.pub
# Go to: User Settings → SSH Keys → Add key

# Use SSH remote
git remote set-url origin git@gitlab.com:YOUR_USERNAME/my-fullstack-app.git
```

### Using HTTPS with Token

```bash
# Create Personal Access Token at:
# User Settings → Access Tokens
# Select scopes: read_repository, write_repository

# Clone with token
git clone https://oauth2:YOUR_TOKEN@gitlab.com/YOUR_USERNAME/my-fullstack-app.git
```

---

## GitLab CI/CD Variables

Settings → CI/CD → Variables:

```
DATABASE_URL (masked, protected)
JWT_SECRET (masked, protected)
DOCKER_REGISTRY_USER
DOCKER_REGISTRY_PASSWORD (masked)
```

---

## Container Registry (Docker)

```yaml
# Add to .gitlab-ci.yml for Docker builds
build:docker:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA ./backend
    - docker push $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA
    - docker build -t $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA ./frontend
    - docker push $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
  only:
    - main
```

---

## GitLab Pages (Frontend Static Hosting)

```yaml
pages:
  stage: deploy
  script:
    - cd frontend
    - npm ci
    - npm run build
    - mv dist ../public
  artifacts:
    paths:
      - public
  only:
    - main
```