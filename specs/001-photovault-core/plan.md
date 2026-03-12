# Implementation Plan: PhotoVault Core

**Branch**: `001-photovault-core` | **Date**: 2026-03-12 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `/specs/001-photovault-core/spec.md`

## Summary

Implementación del core de PhotoVault: plataforma de fotos con restricciones para fotógrafos. Arquitectura de microservicios con NestJS monorepo (4 servicios + gateway), Angular 21 frontend, PostgreSQL + Redis, almacenamiento en S3 (MinIO para desarrollo). Protección multicapa anti-descarga y anti-captura como diferenciador principal.

## Technical Context

**Language/Version**: TypeScript 5.9, Node.js 22 LTS
**Primary Dependencies**: NestJS 11, Angular 21, Sharp (image processing), @aws-sdk/client-s3, Passport.js, cockatiel (resilience), nestjs-otel (tracing)
**Storage**: PostgreSQL 16 (4 schemas) + Redis 7 + AWS S3 (MinIO local)
**Testing**: Vitest (unit), Supertest (integration), Playwright (E2E)
**Observability**: OpenTelemetry + nestjs-otel + Jaeger (dev)
**Upload Strategy**: Presigned PUT URL directo a S3/MinIO (ver architecture-analysis.md D-11)
**Target Platform**: Web (SPA + SSR para portafolios públicos)
**Project Type**: Web application (microservices backend + SPA frontend)
**Performance Goals**: Portfolio pages < 3s, upload 10MB < 10s, 500 concurrent uploads
**Constraints**: Imágenes originales nunca expuestas, presigned URLs max 15 min, cobertura > 80%
**Scale/Scope**: MVP para ~1000 fotógrafos iniciales, escalable horizontalmente

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. SDD | PASS | Spec aprobada antes de plan. Plan antes de código. |
| II. TDD | PASS | Vitest + Supertest + Playwright definidos. Cobertura 80%/70% configurada. |
| III. OWASP Top 10 | PASS | Cada endpoint tiene validación de permisos (A01). bcrypt para passwords (A02). ORM parametrizado (A03). Threat model en research (A04). CSP/HSTS headers (A05). npm audit en CI (A06). Rate limiting + JWT expiry (A07). Magic bytes validation (A08). Winston audit logs (A09). URL validation (A10). |
| IV. ISO 27001 | PASS | Data classification definida. Audit logging. Cifrado en reposo/tránsito. Desarrollo seguro. |
| V. Simplicidad | PASS | 4 servicios justificados por dominio. Sin over-engineering. |
| VI. Protección de Contenido | PASS | Server-side validation, presigned URLs, nunca originales expuestos, watermark server-side. |
| Git Flow | PASS | Feature branches desde development, PRs obligatorios, commits convencionales. |

## Project Structure

### Documentation (this feature)

```text
specs/001-photovault-core/
├── spec.md
├── plan.md                    # This file
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── api-gateway.md
│   └── microservices.md
├── checklists/
│   └── requirements.md
└── tasks.md                   # Next: /speckit.tasks
```

### Source Code (repository root)

> **Base**: Estructura derivada de [TeamWorky/nest-proptech-backend](https://github.com/TeamWorky/nest-proptech-backend) (plantilla NestJS).
> Se reutilizan: BaseEntity (UUID+timestamps+soft-delete), Guards (JWT, Roles, MinRole, CanModify), Interceptors (Transform, Logging, Timeout), Filters (HttpException), Middlewares (RequestId), decorators (@Public, @Roles, @CurrentUser), redis module, queue module, email module, logger module, health module, env validation, Docker multi-stage, y patrones de testing.

```text
redSocialFoto/
├── apps/
│   ├── api-gateway/              # NestJS - Routing, rate limiting, auth guard
│   │   ├── src/
│   │   │   ├── guards/           # Reutilizar: JwtAuthGuard, RolesGuard, MinRoleGuard
│   │   │   ├── interceptors/     # Reutilizar: TransformInterceptor, LoggingInterceptor, TimeoutInterceptor
│   │   │   ├── filters/          # Reutilizar: HttpExceptionFilter
│   │   │   ├── middlewares/      # Reutilizar: RequestIdMiddleware
│   │   │   ├── config/           # Reutilizar: env.validation.ts (Joi)
│   │   │   ├── health/           # Reutilizar: HealthModule
│   │   │   └── main.ts           # Reutilizar: Helmet, CORS, Swagger/Scalar, ValidationPipe global
│   │   └── test/
│   ├── auth-service/             # NestJS - Auth, photographers, sessions
│   │   ├── src/
│   │   │   ├── auth/             # Reutilizar: auth module (login, register, JWT, refresh)
│   │   │   │   ├── strategies/   # Reutilizar: jwt.strategy.ts + agregar OAuth strategies
│   │   │   │   ├── dto/          # Reutilizar: login.dto, register.dto, refresh-token.dto
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── auth.module.ts
│   │   │   │   └── auth.service.ts
│   │   │   ├── photographers/    # Adaptar de: users module → photographers
│   │   │   │   ├── entities/     # Adaptar: user.entity → photographer.entity
│   │   │   │   ├── dto/
│   │   │   │   ├── photographers.controller.ts
│   │   │   │   ├── photographers.module.ts
│   │   │   │   └── photographers.service.ts
│   │   │   ├── sessions/         # NUEVO: Session management (refresh tokens en DB)
│   │   │   ├── email/            # Reutilizar: email module (BullMQ + Nodemailer)
│   │   │   └── main.ts
│   │   └── test/
│   ├── photos-service/           # NestJS - Photos, albums, S3, processing
│   │   ├── src/
│   │   │   ├── photos/           # NUEVO: Photo CRUD, visibility
│   │   │   │   ├── entities/
│   │   │   │   ├── dto/
│   │   │   │   ├── photos.controller.ts
│   │   │   │   └── photos.service.ts
│   │   │   ├── albums/           # NUEVO: Album CRUD
│   │   │   ├── storage/          # NUEVO: S3/MinIO client, presigned URLs
│   │   │   ├── processing/       # NUEVO: Sharp workers (thumbnails, watermarks)
│   │   │   │   └── processors/   # Patrón de: email.processor.ts
│   │   │   └── main.ts
│   │   └── test/
│   ├── social-service/           # NestJS - Likes, comments, follows, feed
│   │   ├── src/
│   │   │   ├── likes/            # NUEVO
│   │   │   ├── comments/         # NUEVO
│   │   │   ├── follows/          # NUEVO
│   │   │   ├── feed/             # NUEVO
│   │   │   └── main.ts
│   │   └── test/
│   └── billing-service/          # NestJS - Plans, storage tracking
│       ├── src/
│       │   ├── plans/            # NUEVO: StoragePlan CRUD
│       │   ├── usage/            # NUEVO: Storage tracking
│       │   └── main.ts
│       └── test/
├── libs/
│   └── common/                   # Reutilizar completamente de nest-proptech-backend
│       ├── decorators/           # @Public, @Roles, @MinRole, @CurrentUser
│       ├── dto/                  # PaginationDto
│       ├── entities/             # BaseEntity (UUID, timestamps, soft-delete)
│       ├── enums/                # Adaptar: Role enum (ADMIN, PHOTOGRAPHER, VISITOR)
│       ├── exceptions/           # BusinessException + variantes
│       ├── interfaces/           # ApiResponse interface
│       ├── repositories/         # BaseRepository
│       ├── validators/           # PasswordStrengthValidator
│       ├── utils/                # SoftDeleteUtil
│       └── constants/            # AppConstants
├── frontend/                     # Angular 21 SPA
│   ├── src/
│   │   ├── app/
│   │   │   ├── core/             # Guards, interceptors, services singleton
│   │   │   ├── shared/           # Shared components, pipes, directives
│   │   │   ├── features/
│   │   │   │   ├── auth/         # Login, register, forgot password
│   │   │   │   ├── dashboard/    # Photo library, storage
│   │   │   │   ├── portfolio/    # Public portfolio view
│   │   │   │   ├── upload/       # Photo upload with progress
│   │   │   │   ├── photo-viewer/ # Protected image viewer (canvas overlay)
│   │   │   │   └── profile/      # Profile editor
│   │   │   └── app.routes.ts
│   │   ├── environments/
│   │   └── styles/
│   └── test/
├── docker-compose.yml            # PostgreSQL + Redis + MinIO (dev)
├── docker-compose.prod.yml       # Producción con resource limits
├── Dockerfile                    # Multi-stage (reutilizar patrón de nest-proptech)
├── .specify/                     # Spec Kit
├── specs/                        # Specifications
├── .github/
│   └── workflows/
│       ├── ci.yml                # Build + test + audit
│       └── cd.yml                # Deploy
├── PROJECT.md
├── REQUIREMENTS.md
├── FEATURES.md
├── nest-cli.json                 # NestJS monorepo config
├── package.json                  # Root workspace
├── tsconfig.json
├── tsconfig.build.json
├── eslint.config.mjs
├── .prettierrc
├── .env.example
└── .dockerignore
```

**Structure Decision**: NestJS monorepo basado en la plantilla `nest-proptech-backend`. Cada microservicio es una app en `apps/`. Se reutilizan guards, interceptors, filters, decorators, base entity, email module, redis module, queue module, y patrones de testing de la plantilla. Frontend Angular en `frontend/`. Docker Compose con PostgreSQL, Redis y MinIO.

**Reutilización de nest-proptech-backend**:
| Componente | Acción | Fuente |
|------------|--------|--------|
| BaseEntity | Reutilizar tal cual | common/entities/base.entity.ts |
| Guards (JWT, Roles, MinRole) | Reutilizar, adaptar roles | guards/*.ts |
| Interceptors (Transform, Logging, Timeout) | Reutilizar tal cual | interceptors/*.ts |
| HttpExceptionFilter | Reutilizar tal cual | filters/http-exception.filter.ts |
| RequestIdMiddleware | Reutilizar tal cual | middlewares/request-id.middleware.ts |
| Decorators (@Public, @Roles, @CurrentUser) | Reutilizar tal cual | common/decorators/*.ts |
| Auth module (JWT + Refresh) | Reutilizar, agregar OAuth | auth/*.ts |
| Email module (BullMQ + Nodemailer) | Reutilizar tal cual | email/*.ts |
| Redis module | Reutilizar tal cual | redis/*.ts |
| Queue module (BullMQ) | Reutilizar tal cual | queue/*.ts |
| Logger module (Winston) | Reutilizar tal cual | logger/*.ts |
| Health module | Reutilizar tal cual | health/*.ts |
| Env validation (Joi) | Adaptar variables | config/env.validation.ts |
| Docker multi-stage | Reutilizar patrón | Dockerfile |
| ApiResponse interface | Reutilizar tal cual | common/interfaces/*.ts |
| PasswordStrengthValidator | Reutilizar tal cual | common/validators/*.ts |
| User entity → Photographer | Adaptar campos | users/entities/user.entity.ts |
| Admin seeder | Adaptar a PhotoVault admin | database/seeders/*.ts |
| main.ts setup | Reutilizar patrón | main.ts |

## Complexity Tracking

| Decision | Why Needed | Simpler Alternative Rejected Because |
|----------|------------|-------------------------------------|
| 4 servicios + gateway | Separación por dominio permite escalar photos-service independientemente (es el más intensivo en I/O) | Monolito no permite escalar procesamiento de imágenes separado del auth |
| Redis como transport | Ya requerido para cache/sesiones, reutilizar evita agregar RabbitMQ | TCP no soporta pub/sub necesario para eventos async |
| MinIO para desarrollo | Permite testing real de presigned URLs y S3 API sin costos AWS | Mocks de S3 no validan comportamiento real de URLs firmadas |
| Presigned URL (upload directo) | Gateway no procesa archivos (0 RAM vs 3GB/batch) | Proxy por gateway inviable a 500 uploads concurrentes |
| Cockatiel (circuit breaker) | Maduro (VS Code), tipado excelente, componible | nestjs-resilience poco documentado; custom propenso a errores |
| BullMQ DLQ + reconciliación | Garantiza entrega de eventos críticos | Solo retry sin DLQ pierde eventos permanentemente |
| OpenTelemetry distribuido | Tracing entre 5 procesos, auto-instrumentación | Solo Winston+RequestId insuficiente para debugging distribuido |
| Angular Signals (sin NgRx) | Suficiente para MVP, menos boilerplate | NgRx agrega complejidad innecesaria |
