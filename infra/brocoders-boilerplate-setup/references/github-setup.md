# GitHub Setup Guide

## Prerequisites

- GitHub account
- Git installed locally
- GitHub CLI (optional but recommended): `brew install gh` or [download](https://cli.github.com/)

## Option A: Monorepo Setup

### 1. Create GitHub Repository

**Using GitHub CLI:**
```bash
gh repo create my-fullstack-app --private --source=. --remote=origin
```

**Using Web Interface:**
1. Go to https://github.com/new
2. Repository name: `my-fullstack-app`
3. Choose Private/Public
4. Do NOT initialize with README
5. Click "Create repository"

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
*.swo

# OS
.DS_Store
Thumbs.db

# Logs
*.log
logs/

# Docker volumes
data/
EOF

# Create root package.json for workspace (optional)
cat > package.json << 'EOF'
{
  "name": "my-fullstack-app",
  "private": true,
  "workspaces": ["backend", "frontend"],
  "scripts": {
    "backend:dev": "npm run start:dev --workspace=backend",
    "frontend:dev": "npm run dev --workspace=frontend",
    "install:all": "npm install --workspaces"
  }
}
EOF

# Initial commit
git add .
git commit -m "Initial commit: NestJS + React boilerplate setup"

# Add remote and push
git remote add origin https://github.com/YOUR_USERNAME/my-fullstack-app.git
git branch -M main
git push -u origin main
```

### 3. GitHub Actions CI/CD

Create `.github/workflows/ci.yml`:

```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  backend-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./backend
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: backend/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run tests
        run: npm run test
        env:
          DATABASE_URL: postgres://postgres:postgres@localhost:5432/test
      
      - name: Build
        run: npm run build

  frontend-test:
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ./frontend
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend/package-lock.json
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run linting
        run: npm run lint
      
      - name: Run tests
        run: npm run test --passWithNoTests
      
      - name: Build
        run: npm run build
```

### 4. Branch Protection (Recommended)

```bash
# Using GitHub CLI
gh api repos/{owner}/{repo}/branches/main/protection -X PUT \
  -F required_status_checks='{"strict":true,"contexts":["backend-test","frontend-test"]}' \
  -F enforce_admins=false \
  -F required_pull_request_reviews='{"required_approving_review_count":1}'
```

Or via Settings → Branches → Add rule.

---

## Option B: Polyrepo Setup (Separate Repositories)

### Backend Repository

```bash
cd backend

# Initialize git
git init
git add .
git commit -m "Initial commit: NestJS boilerplate"

# Create and push to GitHub
gh repo create my-app-backend --private --source=. --remote=origin --push
# OR manually:
# git remote add origin https://github.com/YOUR_USERNAME/my-app-backend.git
# git branch -M main
# git push -u origin main
```

### Frontend Repository

```bash
cd frontend

# Initialize git
git init
git add .
git commit -m "Initial commit: React boilerplate"

# Create and push to GitHub
gh repo create my-app-frontend --private --source=. --remote=origin --push
```

---

## SSH vs HTTPS Authentication

### Using SSH (Recommended)

```bash
# Generate SSH key if needed
ssh-keygen -t ed25519 -C "your_email@example.com"

# Add to ssh-agent
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519

# Add public key to GitHub
gh ssh-key add ~/.ssh/id_ed25519.pub --title "My Machine"
# OR copy ~/.ssh/id_ed25519.pub to GitHub Settings → SSH Keys

# Use SSH remote
git remote set-url origin git@github.com:YOUR_USERNAME/my-fullstack-app.git
```

### Using HTTPS with Token

```bash
# Create Personal Access Token at:
# https://github.com/settings/tokens/new
# Select scopes: repo, workflow

# Configure credential caching
git config --global credential.helper cache
# OR for permanent storage:
git config --global credential.helper store
```

---

## GitHub Secrets for CI/CD

Add secrets at: Repository → Settings → Secrets and variables → Actions

**Required secrets for deployment:**
```
DATABASE_URL=postgresql://user:pass@host:5432/db
JWT_SECRET=your-jwt-secret
AWS_ACCESS_KEY_ID=xxx (if using AWS)
AWS_SECRET_ACCESS_KEY=xxx
```

---

## Deployment Options

### Vercel (Frontend)

```bash
# Install Vercel CLI
npm i -g vercel

cd frontend
vercel link
vercel --prod
```

### Railway/Render (Backend)

Add to workflow for auto-deploy:

```yaml
deploy-backend:
  needs: backend-test
  if: github.ref == 'refs/heads/main'
  runs-on: ubuntu-latest
  steps:
    - uses: actions/checkout@v4
    - name: Deploy to Railway
      uses: bervProject/railway-deploy@main
      with:
        railway_token: ${{ secrets.RAILWAY_TOKEN }}
        service: backend
```