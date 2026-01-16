# GitLab Self-Hosted Setup Guide

## Prerequisites

- Access to self-hosted GitLab instance
- GitLab URL (e.g., `https://gitlab.yourcompany.com`)
- Valid user account with project creation permissions
- Git installed locally

## Initial Configuration

### 1. Configure Git for Self-Hosted GitLab

```bash
# Set your GitLab instance URL
export GITLAB_HOST="gitlab.yourcompany.com"

# Configure git to use your identity
git config --global user.name "Your Name"
git config --global user.email "your.email@company.com"
```

### 2. SSL Certificate Handling

**If using self-signed certificates:**

```bash
# Option 1: Disable SSL verification (NOT recommended for production)
git config --global http.sslVerify false

# Option 2: Add certificate to trusted store (Recommended)
# Download the certificate
openssl s_client -connect gitlab.yourcompany.com:443 -showcerts < /dev/null 2>/dev/null | \
  openssl x509 -outform PEM > gitlab-cert.pem

# Add to git config
git config --global http."https://gitlab.yourcompany.com/".sslCAInfo /path/to/gitlab-cert.pem

# Option 3: Add to system trust store
# Linux (Debian/Ubuntu):
sudo cp gitlab-cert.pem /usr/local/share/ca-certificates/gitlab.crt
sudo update-ca-certificates

# macOS:
sudo security add-trusted-cert -d -r trustRoot -k /Library/Keychains/System.keychain gitlab-cert.pem
```

---

## Option A: Monorepo Setup

### 1. Create Project on Self-Hosted GitLab

**Via Web Interface:**
1. Navigate to `https://gitlab.yourcompany.com`
2. Click "New project" → "Create blank project"
3. Project name: `my-fullstack-app`
4. Select namespace (group or personal)
5. Visibility: Internal/Private
6. Uncheck "Initialize repository with a README"
7. Click "Create project"

**Via API:**
```bash
# Create Personal Access Token first:
# User Settings → Access Tokens → Add new token
# Scopes: api, read_repository, write_repository

export GITLAB_TOKEN="your-personal-access-token"
export GITLAB_HOST="gitlab.yourcompany.com"

curl --request POST "https://${GITLAB_HOST}/api/v4/projects" \
  --header "PRIVATE-TOKEN: ${GITLAB_TOKEN}" \
  --header "Content-Type: application/json" \
  --data '{
    "name": "my-fullstack-app",
    "visibility": "private",
    "initialize_with_readme": false
  }'
```

### 2. Initialize Local Repository

```bash
# In project root (containing backend/ and frontend/)
git init

# Create .gitignore
cat > .gitignore << 'EOF'
node_modules/
.env
.env.local
.env.*.local
dist/
build/
.next/
.idea/
.vscode/
*.swp
.DS_Store
*.log
logs/
data/
EOF

# Initial commit
git add .
git commit -m "Initial commit: NestJS + React boilerplate setup"

# Add remote (replace with your actual GitLab URL and path)
git remote add origin https://gitlab.yourcompany.com/YOUR_GROUP/my-fullstack-app.git
# OR with namespace:
# git remote add origin https://gitlab.yourcompany.com/team/projects/my-fullstack-app.git

git branch -M main
git push -u origin main
```

### 3. GitLab CI/CD Pipeline

Create `.gitlab-ci.yml`:

```yaml
stages:
  - test
  - build
  - deploy

variables:
  NODE_VERSION: "20"
  # For self-hosted runners, you might need:
  # DOCKER_HOST: tcp://docker:2375
  # DOCKER_TLS_CERTDIR: ""

# Adjust for your runner configuration
default:
  tags:
    - docker  # or your specific runner tag

.node_cache: &node_cache
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - backend/node_modules/
      - frontend/node_modules/

# ==================== BACKEND ====================
backend:test:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  services:
    - name: postgres:15
      alias: postgres
  variables:
    POSTGRES_DB: test
    POSTGRES_USER: postgres
    POSTGRES_PASSWORD: postgres
    DATABASE_URL: "postgresql://postgres:postgres@postgres:5432/test"
  script:
    - cd backend
    - npm ci
    - npm run lint
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
    expire_in: 1 day

# ==================== FRONTEND ====================
frontend:test:
  stage: test
  image: node:${NODE_VERSION}
  <<: *node_cache
  script:
    - cd frontend
    - npm ci
    - npm run lint
    - npm run test --passWithNoTests

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
    expire_in: 1 day

# ==================== DOCKER BUILD ====================
build:docker:
  stage: build
  image: docker:24
  services:
    - name: docker:24-dind
      alias: docker
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    # For self-hosted registry:
    REGISTRY: "registry.yourcompany.com"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - docker build -t $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA ./backend
    - docker build -t $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA ./frontend
    - docker push $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA
    # Tag as latest for main branch
    - |
      if [ "$CI_COMMIT_BRANCH" == "main" ]; then
        docker tag $CI_REGISTRY_IMAGE/backend:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE/backend:latest
        docker tag $CI_REGISTRY_IMAGE/frontend:$CI_COMMIT_SHA $CI_REGISTRY_IMAGE/frontend:latest
        docker push $CI_REGISTRY_IMAGE/backend:latest
        docker push $CI_REGISTRY_IMAGE/frontend:latest
      fi
  only:
    - main
    - develop

# ==================== DEPLOY ====================
deploy:staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  script:
    - ssh $DEPLOY_USER@$STAGING_SERVER "cd /app && docker-compose pull && docker-compose up -d"
  environment:
    name: staging
    url: https://staging.yourcompany.com
  only:
    - develop
  when: manual

deploy:production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
    - echo "$SSH_KNOWN_HOSTS" >> ~/.ssh/known_hosts
  script:
    - ssh $DEPLOY_USER@$PRODUCTION_SERVER "cd /app && docker-compose pull && docker-compose up -d"
  environment:
    name: production
    url: https://app.yourcompany.com
  only:
    - main
  when: manual
```

---

## Option B: Polyrepo Setup

### Backend Repository

```bash
cd backend
git init
git add .
git commit -m "Initial commit: NestJS boilerplate"
git remote add origin https://gitlab.yourcompany.com/YOUR_GROUP/my-app-backend.git
git branch -M main
git push -u origin main
```

### Frontend Repository

```bash
cd frontend
git init
git add .
git commit -m "Initial commit: React boilerplate"
git remote add origin https://gitlab.yourcompany.com/YOUR_GROUP/my-app-frontend.git
git branch -M main
git push -u origin main
```

---

## SSH Authentication for Self-Hosted GitLab

### Generate and Add SSH Key

```bash
# Generate SSH key
ssh-keygen -t ed25519 -C "your.email@company.com" -f ~/.ssh/gitlab_selfhosted

# Add to SSH config
cat >> ~/.ssh/config << EOF

Host gitlab.yourcompany.com
  HostName gitlab.yourcompany.com
  User git
  IdentityFile ~/.ssh/gitlab_selfhosted
  IdentitiesOnly yes
EOF

# Copy public key
cat ~/.ssh/gitlab_selfhosted.pub
# Add this to: User Settings → SSH Keys

# Test connection
ssh -T git@gitlab.yourcompany.com

# Use SSH remote
git remote set-url origin git@gitlab.yourcompany.com:YOUR_GROUP/my-fullstack-app.git
```

### For Non-Standard SSH Port

```bash
# If GitLab uses a different SSH port (e.g., 2222)
cat >> ~/.ssh/config << EOF

Host gitlab.yourcompany.com
  HostName gitlab.yourcompany.com
  Port 2222
  User git
  IdentityFile ~/.ssh/gitlab_selfhosted
EOF
```

---

## GitLab Runner Setup (Self-Hosted)

### Install GitLab Runner on Server

```bash
# Download and install
curl -L "https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh" | sudo bash
sudo apt-get install gitlab-runner

# Register runner
sudo gitlab-runner register \
  --url "https://gitlab.yourcompany.com" \
  --registration-token "YOUR_REGISTRATION_TOKEN" \
  --description "docker-runner" \
  --executor "docker" \
  --docker-image "node:20" \
  --docker-privileged \
  --docker-volumes "/certs/client"
```

### Runner with Docker-in-Docker

```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  name = "docker-runner"
  url = "https://gitlab.yourcompany.com"
  token = "RUNNER_TOKEN"
  executor = "docker"
  [runners.docker]
    tls_verify = false
    image = "docker:24"
    privileged = true
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/certs/client", "/cache"]
    shm_size = 0
```

---

## CI/CD Variables

Settings → CI/CD → Variables:

```
# Database
DATABASE_URL (masked, protected)

# Application secrets
JWT_SECRET (masked, protected)
APP_SECRET (masked, protected)

# Docker registry (if using external registry)
DOCKER_REGISTRY_URL
DOCKER_REGISTRY_USER
DOCKER_REGISTRY_PASSWORD (masked)

# Deployment
SSH_PRIVATE_KEY (masked, file type)
SSH_KNOWN_HOSTS
STAGING_SERVER
PRODUCTION_SERVER
DEPLOY_USER
```

---

## Private Container Registry

### Using GitLab's Built-in Registry

```bash
# Login to GitLab registry
docker login registry.yourcompany.com
# Username: your-gitlab-username
# Password: your-personal-access-token (with read_registry, write_registry scopes)

# Build and push
docker build -t registry.yourcompany.com/group/project/backend:latest ./backend
docker push registry.yourcompany.com/group/project/backend:latest
```

### Using External Registry (Harbor, Nexus, etc.)

```yaml
# In .gitlab-ci.yml
variables:
  REGISTRY: "harbor.yourcompany.com"
  IMAGE_PATH: "${REGISTRY}/my-project"

build:docker:
  before_script:
    - docker login -u $HARBOR_USER -p $HARBOR_PASSWORD $REGISTRY
  script:
    - docker build -t ${IMAGE_PATH}/backend:${CI_COMMIT_SHA} ./backend
    - docker push ${IMAGE_PATH}/backend:${CI_COMMIT_SHA}
```

---

## Troubleshooting Self-Hosted Issues

### SSL/TLS Issues

```bash
# Test SSL connection
openssl s_client -connect gitlab.yourcompany.com:443

# Git SSL debugging
GIT_SSL_NO_VERIFY=1 git clone https://gitlab.yourcompany.com/group/repo.git
```

### Network/Firewall Issues

```bash
# Test connectivity
curl -v https://gitlab.yourcompany.com/api/v4/projects

# Check if ports are open
nc -zv gitlab.yourcompany.com 443
nc -zv gitlab.yourcompany.com 22
```

### Runner Connection Issues

```bash
# Check runner status
sudo gitlab-runner verify

# View runner logs
sudo journalctl -u gitlab-runner -f
```