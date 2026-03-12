# redSocialFoto Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-12

## Active Technologies

- TypeScript 5.9, Node.js 22 LTS + NestJS 11, Angular 21, Sharp (image processing), @aws-sdk/client-s3, Passport.js, cockatiel (resilience), nestjs-otel (tracing)

## Project Structure

```text
redSocialFoto/
├── apps/
│   ├── api-gateway/          # NestJS - Routing, rate limiting, auth guard (port 3000)
│   ├── auth-service/         # NestJS - Auth, photographers, sessions (port 3001)
│   ├── photos-service/       # NestJS - Photos, albums, S3, processing (port 3002)
│   ├── social-service/       # NestJS - Likes, comments, follows, feed (port 3003)
│   └── billing-service/      # NestJS - Plans, storage tracking (port 3004)
├── libs/
│   └── common/               # Shared: BaseEntity, Guards, Interceptors, Decorators, DTOs
├── frontend/                 # Angular 21 SPA
├── database/
│   └── migrations/           # Centralized TypeORM migrations (4 schemas)
├── docker-compose.yml        # PostgreSQL + Redis + MinIO (dev)
├── specs/                    # Spec Kit specifications
└── .specify/                 # Spec Kit config
```

## Commands

```bash
# Dev
npm run start:dev api-gateway    # Start gateway
npm run start:dev auth-service   # Start auth service
docker compose up -d             # Start PostgreSQL + Redis + MinIO

# Test
npm run test                     # Unit tests (Vitest)
npm run test:integration         # Integration tests (Supertest)
npm run test:e2e                 # E2E tests (Playwright)
npm run lint                     # ESLint

# Database
npm run migration:run            # Run all migrations
npm run migration:revert         # Revert last migration
```

## Code Style

- TypeScript strict mode, ESLint + Prettier
- NestJS conventions: modules, controllers, services, DTOs with class-validator
- Conventional Commits: feat/fix/docs/style/refactor/test/chore/hotfix/security

## Recent Changes

- 001-photovault-core: Full specification and architecture analysis (21 decisions, 139 tasks)

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
