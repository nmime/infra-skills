# Environment Setup Guide

## Backend (NestJS) Environment

### 1. Create Environment File

```bash
cd backend
cp env-example .env
```

### 2. Configure `.env` Variables

```env
# Application
NODE_ENV=development
APP_PORT=3000
APP_NAME="My App API"
API_PREFIX=api
APP_FALLBACK_LANGUAGE=en
APP_HEADER_LANGUAGE=x-custom-lang

# Frontend URL (for CORS)
FRONTEND_DOMAIN=http://localhost:3001

# Database
DATABASE_TYPE=postgres
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=postgres
DATABASE_NAME=my_app
DATABASE_SYNCHRONIZE=false
DATABASE_MAX_CONNECTIONS=100
DATABASE_SSL_ENABLED=false
DATABASE_REJECT_UNAUTHORIZED=false

# Authentication
AUTH_JWT_SECRET=your-super-secret-jwt-key-change-in-production
AUTH_JWT_TOKEN_EXPIRES_IN=15m
AUTH_REFRESH_SECRET=your-refresh-secret-key
AUTH_REFRESH_TOKEN_EXPIRES_IN=7d
AUTH_FORGOT_SECRET=your-forgot-password-secret
AUTH_FORGOT_TOKEN_EXPIRES_IN=30m
AUTH_CONFIRM_EMAIL_SECRET=your-confirm-email-secret
AUTH_CONFIRM_EMAIL_TOKEN_EXPIRES_IN=1d

# Mail
MAIL_HOST=maildev
MAIL_PORT=1025
MAIL_USER=
MAIL_PASSWORD=
MAIL_IGNORE_TLS=true
MAIL_SECURE=false
MAIL_REQUIRE_TLS=false
MAIL_DEFAULT_EMAIL=noreply@example.com
MAIL_DEFAULT_NAME="My App"
MAIL_CLIENT_PORT=1080

# File Storage
FILE_DRIVER=local
# For S3:
# FILE_DRIVER=s3
# ACCESS_KEY_ID=your-access-key
# SECRET_ACCESS_KEY=your-secret-key
# AWS_S3_REGION=us-east-1
# AWS_DEFAULT_S3_BUCKET=my-bucket

# Social Auth (Optional)
# GOOGLE_CLIENT_ID=
# GOOGLE_CLIENT_SECRET=
# FACEBOOK_APP_ID=
# FACEBOOK_APP_SECRET=
# APPLE_APP_AUDIENCE=
# TWITTER_CONSUMER_KEY=
# TWITTER_CONSUMER_SECRET=

# Workers
WORKER_HOST=redis://localhost:6379/1
```

### 3. Database Setup

```bash
# Option 1: Using Docker (Recommended)
cd backend
docker-compose up -d postgres

# Option 2: Local PostgreSQL
createdb my_app

# Run migrations
npm run migration:run

# Seed initial data
npm run seed:run
```

### 4. Start Backend

```bash
# Install dependencies
npm install

# Development mode
npm run start:dev

# Or using Docker
docker-compose up -d
```

---

## Frontend (React) Environment

### 1. Create Environment File

```bash
cd frontend
cp .env.example .env.local
```

### 2. Configure `.env.local` Variables

```env
# API Configuration
NEXT_PUBLIC_API_URL=http://localhost:3000/api

# App Configuration
NEXT_PUBLIC_APP_NAME="My App"
NEXT_PUBLIC_APP_URL=http://localhost:3001

# Authentication
NEXT_PUBLIC_AUTH_GOOGLE_ID=your-google-client-id
NEXT_PUBLIC_AUTH_FACEBOOK_ID=your-facebook-app-id

# Feature Flags (Optional)
NEXT_PUBLIC_ENABLE_SOCIAL_AUTH=true
NEXT_PUBLIC_ENABLE_REGISTRATION=true
```

### 3. Start Frontend

```bash
# Install dependencies
npm install

# Development mode
npm run dev

# Build for production
npm run build
npm run start
```

---

## Full Docker Setup (Both Services)

### 1. Create Root `docker-compose.yml`

```yaml
version: '3.8'

services:
  # PostgreSQL Database
  postgres:
    image: postgres:15-alpine
    container_name: app-postgres
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: my_app
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis (for queues/caching)
  redis:
    image: redis:7-alpine
    container_name: app-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data

  # Mail Development Server
  maildev:
    image: maildev/maildev
    container_name: app-maildev
    ports:
      - "1080:1080"  # Web UI
      - "1025:1025"  # SMTP

  # Backend API
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: app-backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    environment:
      - NODE_ENV=development
      - DATABASE_HOST=postgres
      - WORKER_HOST=redis://redis:6379/1
      - MAIL_HOST=maildev
    ports:
      - "3000:3000"
    volumes:
      - ./backend:/app
      - /app/node_modules
    command: npm run start:dev

  # Frontend
  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    container_name: app-frontend
    depends_on:
      - backend
    environment:
      - NEXT_PUBLIC_API_URL=http://localhost:3000/api
    ports:
      - "3001:3000"
    volumes:
      - ./frontend:/app
      - /app/node_modules
      - /app/.next
    command: npm run dev

volumes:
  postgres_data:
  redis_data:
```

### 2. Run Everything

```bash
# Start all services
docker-compose up -d

# View logs
docker-compose logs -f

# Stop all services
docker-compose down

# Reset database
docker-compose down -v
docker-compose up -d
```

---

## Development URLs

| Service | URL |
|---------|-----|
| Backend API | http://localhost:3000/api |
| API Documentation | http://localhost:3000/docs |
| Frontend | http://localhost:3001 |
| Mail UI (MailDev) | http://localhost:1080 |
| Database | localhost:5432 |
| Redis | localhost:6379 |

---

## Production Environment Checklist

- [ ] Change all default secrets and passwords
- [ ] Set `NODE_ENV=production`
- [ ] Enable SSL/TLS for database connections
- [ ] Configure proper CORS origins
- [ ] Set up proper mail service (SendGrid, SES, etc.)
- [ ] Configure file storage (S3, CloudFront, etc.)
- [ ] Enable rate limiting
- [ ] Set up monitoring and logging
- [ ] Configure proper JWT expiration times
- [ ] Enable database connection pooling