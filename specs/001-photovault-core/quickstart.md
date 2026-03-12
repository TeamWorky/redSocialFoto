# Quickstart: PhotoVault Core

**Branch**: `001-photovault-core` | **Date**: 2026-03-12

## Prerequisites

- Node.js 22 LTS
- Docker & Docker Compose
- Git

## Setup

### 1. Clone and install

```bash
git clone https://github.com/TeamWorky/redSocialFoto.git
cd redSocialFoto
npm install
```

### 2. Start infrastructure

```bash
docker compose up -d
# Starts: PostgreSQL (5432), Redis (6379), MinIO (9000/9001)
```

### 3. Configure environment

```bash
cp .env.example .env
# Edit .env with your values (defaults work for local development)
```

### 4. Run migrations

```bash
npm run migration:run
```

### 5. Start services

```bash
# All services at once
npm run start:dev

# Or individually
npm run start:dev api-gateway
npm run start:dev auth-service
npm run start:dev photos-service
npm run start:dev social-service
npm run start:dev billing-service
```

### 6. Start frontend

```bash
cd frontend
npm install
npm run start
# Open http://localhost:4200
```

## Access Points

| Service | URL | Notes |
|---------|-----|-------|
| Frontend | http://localhost:4200 | Angular SPA |
| API Gateway | http://localhost:3000/api/v1 | Main API |
| Swagger | http://localhost:3000/api-docs | API documentation |
| MinIO Console | http://localhost:9001 | S3-compatible storage (minioadmin/minioadmin) |

## Default Data

After running migrations and seeds:
- **Admin**: admin@photovault.com / Admin123!
- **Storage Plans**: Free (1GB), Basic (10GB), Pro (50GB), Enterprise (500GB)
- **MinIO Bucket**: `photovault` (auto-created)

## Testing

```bash
# Unit tests
npm run test

# Integration tests
npm run test:integration

# E2E tests
npm run test:e2e

# Coverage
npm run test:cov
```

## Key Commands

```bash
npm run lint              # Lint all projects
npm run format            # Format code
npm run migration:generate -- -n MigrationName  # Generate migration
npm run migration:run     # Run pending migrations
npm run migration:revert  # Revert last migration
```
