# NestJS Boilerplate Reference

## Overview

Production-ready NestJS boilerplate with:
- TypeORM with PostgreSQL
- JWT Authentication with refresh tokens
- Social authentication (Google, Facebook, Apple, Twitter)
- Email verification and password reset
- File uploads (local/S3)
- Swagger API documentation
- Docker support
- Database seeding
- i18n support

## Project Structure

```
backend/
├── src/
│   ├── auth/              # Authentication module
│   ├── config/            # Configuration modules
│   ├── database/          # Database config, migrations, seeds
│   ├── files/             # File upload module
│   ├── forgot/            # Password reset
│   ├── mail/              # Email module
│   ├── roles/             # Role-based access control
│   ├── session/           # Session management
│   ├── social/            # Social auth providers
│   ├── statuses/          # User status management
│   ├── users/             # User module
│   └── utils/             # Utilities and helpers
├── test/                  # E2E tests
├── docker-compose.yml
└── Dockerfile
```

## Key Commands

```bash
# Development
npm run start:dev        # Start with hot-reload
npm run start:debug      # Start with debugger

# Database
npm run migration:generate -- src/database/migrations/MigrationName
npm run migration:run    # Run pending migrations
npm run migration:revert # Revert last migration
npm run seed:run         # Run database seeds
npm run schema:drop      # Drop all tables

# Testing
npm run test             # Unit tests
npm run test:e2e         # E2E tests
npm run test:cov         # Test coverage

# Code Quality
npm run lint             # ESLint
npm run format           # Prettier

# Build
npm run build            # Production build
npm run start:prod       # Start production
```

## API Endpoints

### Authentication
```
POST   /api/v1/auth/email/login          # Login with email
POST   /api/v1/auth/email/register       # Register new user
POST   /api/v1/auth/email/confirm        # Confirm email
POST   /api/v1/auth/forgot/password      # Request password reset
POST   /api/v1/auth/reset/password       # Reset password
POST   /api/v1/auth/refresh              # Refresh access token
POST   /api/v1/auth/logout               # Logout
GET    /api/v1/auth/me                   # Get current user
```

### Users
```
GET    /api/v1/users                     # List users (admin)
GET    /api/v1/users/:id                 # Get user by ID
PATCH  /api/v1/users/:id                 # Update user
DELETE /api/v1/users/:id                 # Delete user
```

### Files
```
POST   /api/v1/files/upload              # Upload file
```

## Adding New Module

```bash
# Generate module
nest g module modules/posts
nest g controller modules/posts
nest g service modules/posts

# Generate entity
nest g class modules/posts/entities/post.entity --no-spec
```

## Common Customizations

### Add New Role

```typescript
// src/roles/roles.enum.ts
export enum RoleEnum {
  admin = 1,
  user = 2,
  moderator = 3,  // Add new role
}
```

### Add New User Status

```typescript
// src/statuses/statuses.enum.ts
export enum StatusEnum {
  active = 1,
  inactive = 2,
  suspended = 3,  // Add new status
}
```

### Custom Validation

```typescript
import { IsNotEmpty, IsString, MinLength } from 'class-validator';

export class CreatePostDto {
  @IsString()
  @IsNotEmpty()
  @MinLength(3)
  title: string;

  @IsString()
  content: string;
}
```