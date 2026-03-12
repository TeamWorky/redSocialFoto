# Tasks: PhotoVault Core

**Input**: Design documents from `/specs/001-photovault-core/`
**Prerequisites**: plan.md (required), spec.md (required for user stories), research.md, data-model.md, contracts/

**Tests**: TDD is NON-NEGOTIABLE per constitution. Test tasks are included for each user story and MUST be written and FAIL before implementation.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3, US4, US5)
- Include exact file paths in descriptions

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization, monorepo structure, Docker infrastructure, environment config

- [ ] T001 Initialize NestJS monorepo with `nest-cli.json` configuring 5 apps: api-gateway, auth-service, photos-service, social-service, billing-service
- [ ] T002 Create root `package.json` with workspace dependencies (NestJS 11, TypeScript 5.9, Vitest, Sharp, @aws-sdk/client-s3, Passport.js, TypeORM, ioredis, BullMQ)
- [ ] T003 [P] Create root `tsconfig.json` and `tsconfig.build.json` with strict mode and path aliases for `@libs/common`
- [ ] T004 [P] Create `eslint.config.mjs` and `.prettierrc` with project linting/formatting rules
- [ ] T005 [P] Create `docker-compose.yml` with PostgreSQL 16 (port 5432, 4 schemas), Redis 7 (port 6379), MinIO (ports 9000/9001) in docker-compose.yml
- [ ] T006 [P] Create `.env.example` with all environment variables (DB, Redis, MinIO/S3, JWT secrets, OAuth, SMTP) in .env.example
- [ ] T007 [P] Create `Dockerfile` with multi-stage build pattern (build → production) in Dockerfile
- [ ] T008 [P] Create `.dockerignore` with node_modules, dist, .env, specs, .specify exclusions in .dockerignore
- [ ] T009 Create skeleton `main.ts` for each app with microservice transport config (Redis) in apps/*/src/main.ts

**Checkpoint**: Monorepo compiles, Docker services start, all 5 apps boot (empty)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Shared libraries, database framework, API Gateway skeleton — MUST complete before ANY user story

**CRITICAL**: No user story work can begin until this phase is complete

- [ ] T010 Create `BaseEntity` with UUID PK, createdAt, updatedAt, soft-delete in libs/common/entities/base.entity.ts
- [ ] T011 [P] Create shared decorators: @Public, @Roles, @MinRole, @CurrentUser in libs/common/decorators/
- [ ] T012 [P] Create shared DTOs: PaginationDto, PaginatedResponseDto in libs/common/dto/
- [ ] T013 [P] Create shared enums: Role (ADMIN, PHOTOGRAPHER), Visibility (PUBLIC, PRIVATE, LINK_ONLY), ProcessingStatus in libs/common/enums/
- [ ] T014 [P] Create shared interfaces: ApiResponse, JwtPayload, MicroservicePayload in libs/common/interfaces/
- [ ] T015 [P] Create shared exceptions: BusinessException and domain variants in libs/common/exceptions/
- [ ] T016 [P] Create PasswordStrengthValidator (min 8, 1 upper, 1 number, 1 special) in libs/common/validators/password-strength.validator.ts
- [ ] T017 [P] Create BaseRepository with soft-delete aware CRUD in libs/common/repositories/base.repository.ts
- [ ] T018 [P] Create AppConstants with Redis key patterns, S3 paths, pagination defaults in libs/common/constants/app.constants.ts
- [ ] T019 Create JwtAuthGuard with Redis blacklist check in apps/api-gateway/src/guards/jwt-auth.guard.ts
- [ ] T020 [P] Create RolesGuard and MinRoleGuard in apps/api-gateway/src/guards/roles.guard.ts
- [ ] T021 [P] Create TransformInterceptor (ApiResponse wrapping) in apps/api-gateway/src/interceptors/transform.interceptor.ts
- [ ] T022 [P] Create LoggingInterceptor (Winston, request timing) in apps/api-gateway/src/interceptors/logging.interceptor.ts
- [ ] T023 [P] Create TimeoutInterceptor (30s default) in apps/api-gateway/src/interceptors/timeout.interceptor.ts
- [ ] T024 [P] Create HttpExceptionFilter with structured error responses in apps/api-gateway/src/filters/http-exception.filter.ts
- [ ] T025 [P] Create RequestIdMiddleware (X-Request-Id header) in apps/api-gateway/src/middlewares/request-id.middleware.ts
- [ ] T026 Create API Gateway main.ts with Helmet, CORS, Swagger/Scalar, ValidationPipe, rate limiting in apps/api-gateway/src/main.ts
- [ ] T027 [P] Create env.validation.ts with Joi schema for all environment variables in apps/api-gateway/src/config/env.validation.ts
- [ ] T028 [P] Create HealthModule with DB, Redis, MinIO health checks in apps/api-gateway/src/health/health.module.ts
- [ ] T029 Create RedisModule (shared, singleton) with ioredis client factory in libs/common/redis/redis.module.ts
- [ ] T030 [P] Create LoggerModule with Winston (console + file transports) in libs/common/logger/logger.module.ts
- [ ] T031 [P] Create QueueModule with BullMQ config (Redis connection) in libs/common/queue/queue.module.ts
- [ ] T032 Create TypeORM configuration with 4 schemas (auth, photos, social, billing) and migration support in database/typeorm.config.ts
- [ ] T033 Create initial migration creating all 4 schemas (auth, photos, social, billing) in database/migrations/001-create-schemas.ts
- [ ] T034 Create ResilienceService singleton with cockatiel (SamplingBreaker, ConsecutiveBreaker, RetryPolicy, TimeoutPolicy per service) in libs/common/resilience/resilience.service.ts
- [ ] T035 [P] Create shared contract types for inter-service communication (auth, photos, billing, events) in libs/common/contracts/
- [ ] T036 [P] Configure OpenTelemetry SDK with auto-instrumentation (HTTP, pg, ioredis, TypeORM, BullMQ) in libs/common/tracing/tracing.ts
- [ ] T037 [P] Configure nestjs-otel module with @Span() decorator support in libs/common/tracing/otel.module.ts
- [ ] T038 [P] Add Jaeger container to docker-compose.yml (ports 16686 UI, 4318 OTLP/HTTP) in docker-compose.yml
- [ ] T039 [P] Configure @nestjs/throttler with ThrottlerStorageRedisService for distributed rate limiting in apps/api-gateway/src/config/throttler.config.ts

**Checkpoint**: Foundation ready — Gateway serves health endpoint, Redis/DB connected, shared libs importable, tracing active, resilience configured

---

## Phase 3: US1 — Photographer Registration & Login (Priority: P1)

**Goal**: Photographers can register (email/OAuth), login, manage sessions, and reset passwords with full security (rate limiting, lockout, audit)

**Independent Test**: Register → Login → Get access token → Access protected endpoint → Logout → Token rejected

### Tests for US1 (TDD — Write FIRST, ensure FAIL)

- [ ] T040 [P] [US1] Unit test for AuthService (register, login, refresh, logout, validateToken) in apps/auth-service/test/auth.service.spec.ts
- [ ] T041 [P] [US1] Unit test for PhotographerService (CRUD, profile update, username validation) in apps/auth-service/test/photographers.service.spec.ts
- [ ] T042 [P] [US1] Unit test for SessionService (create, revoke, cleanup expired) in apps/auth-service/test/sessions.service.spec.ts
- [ ] T043 [P] [US1] Unit test for JwtStrategy and OAuthStrategy in apps/auth-service/test/strategies.spec.ts
- [ ] T044 [P] [US1] Unit test for LoginAttemptService (rate limiting logic, lockout) in apps/auth-service/test/login-attempt.service.spec.ts
- [ ] T045 [P] [US1] Integration test for auth endpoints (register → login → refresh → logout flow) in apps/api-gateway/test/auth.e2e-spec.ts
- [ ] T046 [P] [US1] Integration test for profile endpoints (get/update profile, upload picture) in apps/api-gateway/test/profile.e2e-spec.ts

### Implementation for US1

- [ ] T047 [US1] Create Photographer entity with all fields (email, passwordHash, displayName, username, bio, profilePictureUrl, socialLinks, OAuth, storagePlan, role) in apps/auth-service/src/photographers/entities/photographer.entity.ts
- [ ] T048 [P] [US1] Create Session entity (photographerId, refreshToken hash, ipAddress, userAgent, expiresAt, isRevoked) in apps/auth-service/src/sessions/entities/session.entity.ts
- [ ] T049 [P] [US1] Create LoginAttempt entity (email, ipAddress, success, failureReason) in apps/auth-service/src/auth/entities/login-attempt.entity.ts
- [ ] T050 [P] [US1] Create auth DTOs: RegisterDto, LoginDto, RefreshTokenDto, ForgotPasswordDto, ResetPasswordDto, OAuthLoginDto in apps/auth-service/src/auth/dto/
- [ ] T051 [P] [US1] Create photographer DTOs: UpdateProfileDto, UpdatePictureDto in apps/auth-service/src/photographers/dto/
- [ ] T052 [US1] Create migration for auth schema (photographers, sessions, login_attempts tables with indices) in database/migrations/
- [ ] T053 [US1] Implement PhotographerService (create, findByEmail, findByUsername, updateProfile, uploadPicture) in apps/auth-service/src/photographers/photographers.service.ts
- [ ] T054 [US1] Implement SessionService (createSession, revokeSession, revokeAllSessions, cleanupExpired) in apps/auth-service/src/sessions/sessions.service.ts
- [ ] T055 [US1] Implement LoginAttemptService (recordAttempt, checkRateLimit, checkLockout — 5 attempts/min, 15-min lockout) in apps/auth-service/src/auth/login-attempt.service.ts
- [ ] T056 [US1] Implement JwtStrategy (validate token, check Redis blacklist) in apps/auth-service/src/auth/strategies/jwt.strategy.ts
- [ ] T057 [P] [US1] Implement GoogleOAuthStrategy (validate idToken, create/link photographer) in apps/auth-service/src/auth/strategies/google.strategy.ts
- [ ] T058 [P] [US1] Implement AppleOAuthStrategy (validate idToken, create/link photographer) in apps/auth-service/src/auth/strategies/apple.strategy.ts
- [ ] T059 [US1] Implement AuthService (register, login with rate limiting, OAuth, refresh, logout with Redis blacklist, forgotPassword, resetPassword) in apps/auth-service/src/auth/auth.service.ts
- [ ] T060 [US1] Implement AuthController with @MessagePattern handlers (auth.validate_token, auth.get_photographer, auth.get_photographers_bulk) in apps/auth-service/src/auth/auth.controller.ts
- [ ] T061 [US1] Implement PhotographerController with @MessagePattern handlers for profile operations in apps/auth-service/src/photographers/photographers.controller.ts
- [ ] T062 [US1] Emit @EventPattern('photographer.registered') event on successful registration in apps/auth-service/src/auth/auth.service.ts
- [ ] T063 [US1] Create EmailModule with BullMQ processor for verification/password-reset emails in apps/auth-service/src/email/
- [ ] T064 [US1] Create API Gateway auth routes: POST /auth/register, /auth/login, /auth/oauth/:provider, /auth/refresh, /auth/logout, /auth/forgot-password, /auth/reset-password in apps/api-gateway/src/routes/auth.controller.ts
- [ ] T065 [US1] Create API Gateway profile routes: GET /profile/me, PATCH /profile/me, PUT /profile/me/picture in apps/api-gateway/src/routes/profile.controller.ts
- [ ] T066 [US1] Create Angular auth feature: login page, register page, forgot-password page in frontend/src/app/features/auth/
- [ ] T067 [US1] Create Angular AuthStateService with signals (currentUser, isAuthenticated) and HTTP interceptor chain in frontend/src/app/core/services/auth-state.service.ts
- [ ] T068 [US1] Create Angular auth guard for protected routes in frontend/src/app/core/guards/auth.guard.ts
- [ ] T069 [US1] Create admin seeder (admin@photovault.com / Admin123!) in database/seeders/admin.seeder.ts

**Checkpoint**: Full auth flow works end-to-end. Register, login (email + OAuth), refresh tokens, logout, password reset. Rate limiting blocks brute force. Login attempts audited.

---

## Phase 4: US2 — Photo Upload with Protection (Priority: P1)

**Goal**: Photographers upload photos (JPEG/PNG/WebP, max 50MB, batch 20), auto-processing (thumbnails/optimized), visibility controls, anti-download/anti-screenshot protections

**Independent Test**: Upload photo → Processing completes → View via presigned URL → Anti-download active → Visibility change → Storage updated

### Tests for US2 (TDD — Write FIRST, ensure FAIL)

- [ ] T070 [P] [US2] Unit test for PhotosService (upload-url, confirm-upload, get, update, delete, presigned URL generation) in apps/photos-service/test/photos.service.spec.ts
- [ ] T071 [P] [US2] Unit test for StorageClientService (S3/MinIO presigned PUT URL, presigned GET URL, delete, verify existence) in apps/photos-service/test/storage-client.service.spec.ts
- [ ] T072 [P] [US2] Unit test for ImageProcessorService (thumbnail, optimized, watermark generation) in apps/photos-service/test/image-processor.service.spec.ts
- [ ] T073 [P] [US2] Unit test for file validation (magic bytes, size, mime type — post-upload async) in apps/photos-service/test/file-validation.spec.ts
- [ ] T074 [P] [US2] Integration test for presigned URL upload flow (get-url → PUT to S3 → confirm → process → view) in apps/api-gateway/test/photos.e2e-spec.ts

### Implementation for US2

- [ ] T075 [US2] Create Photo entity with all fields (photographerId, title, description, tags, visibility, S3 keys, fileSizeBytes, dimensions, shareToken, processingStatus) in apps/photos-service/src/photos/entities/photo.entity.ts
- [ ] T076 [US2] Create photo DTOs: UploadUrlRequestDto, UploadUrlResponseDto, ConfirmUploadDto, UpdatePhotoDto, PhotoResponseDto, PhotoListQueryDto in apps/photos-service/src/photos/dto/
- [ ] T077 [US2] Create migration for photos schema (photos table with GIN index on tags, indices on photographerId, visibility, shareToken) in database/migrations/
- [ ] T078 [US2] Implement StorageClientService (S3/MinIO: generatePresignedPutUrl with locked Content-Type, generatePresignedGetUrl, verifyObjectExists, deleteObject — endpoint configurable via env, forcePathStyle for MinIO) in apps/photos-service/src/storage/storage-client.service.ts
- [ ] T079 [US2] Implement async file validation in processing worker (magic bytes check after upload, reject non-images) in apps/photos-service/src/processing/validators/file-validator.ts
- [ ] T080 [US2] Implement ImageProcessorService with Sharp (thumbnail 300x300, optimized 60% quality WebP, watermark overlay) in apps/photos-service/src/processing/image-processor.service.ts
- [ ] T081 [US2] Implement PhotoProcessingProcessor (BullMQ worker: validate file → generate thumbnail + optimized → update status) with DLQ on failure in apps/photos-service/src/processing/processors/photo-processing.processor.ts
- [ ] T082 [US2] Implement PhotosService with presigned URL flow (generateUploadUrls with billing check, confirmUpload with S3 verification, get with presigned GET URLs, update metadata, delete with S3 cleanup) in apps/photos-service/src/photos/photos.service.ts
- [ ] T083 [US2] Implement PhotosController with @MessagePattern handlers (photos.get_photo, photos.get_storage_usage, photos.get_public_photos) in apps/photos-service/src/photos/photos.controller.ts
- [ ] T084 [US2] Emit @EventPattern events: photo.uploaded, photo.deleted, photo.processed in apps/photos-service/src/photos/photos.service.ts
- [ ] T085 [US2] Implement orphan cleanup cron (delete S3 objects without DB record, runs every 1h) in apps/photos-service/src/storage/orphan-cleanup.service.ts
- [ ] T086 [US2] Create API Gateway photo routes: POST /photos/upload-url, POST /photos/confirm-upload, GET /photos, GET /photos/:id, PATCH /photos/:id, DELETE /photos/:id, GET /photos/shared/:shareToken in apps/api-gateway/src/routes/photos.controller.ts
- [ ] T087 [US2] Configure MinIO CORS for PUT from frontend origin in docker-compose.yml (entrypoint script)
- [ ] T088 [US2] Create Angular UploadStateService with signals (uploadQueue, currentUpload, uploadProgress) for presigned URL flow in frontend/src/app/core/services/upload-state.service.ts
- [ ] T089 [US2] Create Angular upload feature: drag-drop component, presigned URL upload with progress (XHR for progress events), batch upload queue in frontend/src/app/features/upload/
- [ ] T090 [US2] Create Angular photo-viewer component with canvas overlay, dynamic watermark, anti-download protections (context menu suppression, drag prevention, CSS user-select:none) in frontend/src/app/features/photo-viewer/
- [ ] T091 [US2] Create Angular photo detail page with protected image display, metadata, visibility badge in frontend/src/app/features/photo-viewer/photo-detail.component.ts

**Checkpoint**: Upload photo → auto-processed (thumbnail + optimized) → view via presigned URL → anti-download/screenshot active → storage usage tracked → visibility controls work

---

## Phase 5: US3 — Photographer Portfolio (Priority: P2)

**Goal**: Public portfolio pages accessible via username slug, featuring bio, profile picture, social links, featured photos/albums with all protections active

**Independent Test**: Set username → Configure portfolio → Access /photographer/:username → Featured photos visible with protections → Profile picture displayed

### Tests for US3 (TDD — Write FIRST, ensure FAIL)

- [ ] T092 [P] [US3] Unit test for PortfolioService (getByUsername, featured photos, cache management) in apps/api-gateway/test/portfolio.service.spec.ts
- [ ] T093 [P] [US3] Integration test for portfolio endpoint (GET /photographer/:username) in apps/api-gateway/test/portfolio.e2e-spec.ts

### Implementation for US3

- [ ] T094 [US3] Add username validation logic (regex: ^[a-z0-9][a-z0-9-]{1,28}[a-z0-9]$, reserved words check) in apps/auth-service/src/photographers/validators/username.validator.ts
- [ ] T095 [US3] Implement PortfolioService (aggregate photographer profile + featured photos + public albums, Redis cache 5 min, circuit breaker for cross-service calls) in apps/api-gateway/src/portfolio/portfolio.service.ts
- [ ] T096 [US3] Create API Gateway portfolio route: GET /photographer/:username (PUBLIC) in apps/api-gateway/src/routes/portfolio.controller.ts
- [ ] T097 [US3] Create Angular portfolio feature: public portfolio page (bio, social links, featured photos grid, album list) with SSR support in frontend/src/app/features/portfolio/
- [ ] T098 [US3] Implement portfolio image display using photo-viewer component with all protections in frontend/src/app/features/portfolio/portfolio-gallery.component.ts

**Checkpoint**: Username set → Portfolio page loads < 3s → Featured photos displayed with protections → Social links visible → Cached in Redis

---

## Phase 6: US4 — Storage Dashboard & Usage Tracking (Priority: P2)

**Goal**: Real-time storage usage display, visual progress bar, per-album breakdown, threshold warnings (80%/95%), upload blocking at 100%

**Independent Test**: Upload photos → Dashboard shows updated usage → Delete photo → Usage decreases → Threshold warning appears at 80%

### Tests for US4 (TDD — Write FIRST, ensure FAIL)

- [ ] T099 [P] [US4] Unit test for StoragePlanService (CRUD plans, assign to photographer) in apps/billing-service/test/storage-plan.service.spec.ts
- [ ] T100 [P] [US4] Unit test for UsageTrackingService (calculate usage, breakdown, thresholds, DLQ handling) in apps/billing-service/test/usage-tracking.service.spec.ts
- [ ] T101 [P] [US4] Integration test for storage endpoints (GET /storage, usage tracking, threshold warnings) in apps/api-gateway/test/storage.e2e-spec.ts

### Implementation for US4

- [ ] T102 [US4] Create StoragePlan entity (name, storageLimitBytes, priceMonthly, priceYearly, maxPhotoSizeMb, maxBatchUpload, features) in apps/billing-service/src/plans/entities/storage-plan.entity.ts
- [ ] T103 [US4] Create storage DTOs: StoragePlanDto, StorageUsageDto, StorageBreakdownDto in apps/billing-service/src/plans/dto/
- [ ] T104 [US4] Create migration for billing schema (storage_plans table) and seed data (Free 1GB, Basic 10GB, Pro 50GB, Enterprise 500GB) in database/migrations/
- [ ] T105 [US4] Implement StoragePlanService (CRUD plans, getByPhotographer) in apps/billing-service/src/plans/storage-plan.service.ts
- [ ] T106 [US4] Implement UsageTrackingService (calculateUsage from photos-service, breakdown by album, check thresholds 80/95/100%) in apps/billing-service/src/usage/usage-tracking.service.ts
- [ ] T107 [US4] Implement BillingController with @MessagePattern handlers (billing.check_storage, billing.get_plan) with circuit breaker wrapping in apps/billing-service/src/billing.controller.ts
- [ ] T108 [US4] Consume @EventPattern('photographer.registered') to assign free plan (with DLQ on failure) in apps/billing-service/src/billing.controller.ts
- [ ] T109 [US4] Consume @EventPattern('photo.uploaded') and @EventPattern('photo.deleted') to update storage usage (with DLQ on failure) in apps/billing-service/src/usage/usage-tracking.service.ts
- [ ] T110 [US4] Emit @EventPattern('storage.threshold_reached') at 80% and 95% usage in apps/billing-service/src/usage/usage-tracking.service.ts
- [ ] T111 [US4] Implement storage reconciliation cron (verify storageUsedBytes vs SUM(fileSizeBytes), every 6h) in apps/billing-service/src/usage/reconciliation.service.ts
- [ ] T112 [US4] Create API Gateway storage route: GET /storage in apps/api-gateway/src/routes/storage.controller.ts
- [ ] T113 [US4] Create Angular StorageStateService with signals (storageUsage, currentPlan, warnings) in frontend/src/app/core/services/storage-state.service.ts
- [ ] T114 [US4] Create Angular storage dashboard component (usage bar, percentage, breakdown table, threshold warnings) in frontend/src/app/features/dashboard/storage-dashboard.component.ts

**Checkpoint**: Free plan auto-assigned on register → Upload tracked → Dashboard shows real-time usage (within 5 min) → Threshold warnings at 80%/95% → Uploads blocked at 100%

---

## Phase 7: US5 — Photo Management (Priority: P2)

**Goal**: Photo library with paginated grid, metadata editing (title/description/tags), visibility changes, album CRUD, search/filter, deletion with storage recalculation

**Independent Test**: View photo library → Create album → Add photos → Edit metadata → Change visibility → Search by tags → Delete photo → Storage updated

### Tests for US5 (TDD — Write FIRST, ensure FAIL)

- [ ] T115 [P] [US5] Unit test for AlbumService (CRUD, add/remove photos, reorder) in apps/photos-service/test/albums.service.spec.ts
- [ ] T116 [P] [US5] Unit test for photo search/filter (visibility, tags, date range, album) in apps/photos-service/test/photos-search.spec.ts
- [ ] T117 [P] [US5] Integration test for album endpoints and photo management in apps/api-gateway/test/albums.e2e-spec.ts

### Implementation for US5

- [ ] T118 [US5] Create Album entity (photographerId, name, description, visibility, coverPhotoId, isFeatured, position) in apps/photos-service/src/albums/entities/album.entity.ts
- [ ] T119 [P] [US5] Create AlbumPhoto pivot entity (albumId, photoId, position, addedAt) in apps/photos-service/src/albums/entities/album-photo.entity.ts
- [ ] T120 [US5] Create album DTOs: CreateAlbumDto, UpdateAlbumDto, AddPhotosDto, AlbumResponseDto in apps/photos-service/src/albums/dto/
- [ ] T121 [US5] Create migration for albums and album_photos tables in database/migrations/
- [ ] T122 [US5] Implement AlbumService (create, update, delete, addPhotos, removePhoto, reorder, getCoverPhoto) in apps/photos-service/src/albums/albums.service.ts
- [ ] T123 [US5] Implement photo search/filter in PhotosService (filter by visibility, tags via GIN index, date range, album, full-text search on title/description) in apps/photos-service/src/photos/photos.service.ts
- [ ] T124 [US5] Implement AlbumController with REST handlers in apps/photos-service/src/albums/albums.controller.ts
- [ ] T125 [US5] Create API Gateway album routes: POST /albums, GET /albums, GET /albums/:id, PATCH /albums/:id, DELETE /albums/:id, POST /albums/:id/photos, DELETE /albums/:id/photos/:photoId in apps/api-gateway/src/routes/albums.controller.ts
- [ ] T126 [US5] Create Angular photo library component (paginated grid, filter panel, search bar, sort options) in frontend/src/app/features/dashboard/photo-library.component.ts
- [ ] T127 [US5] Create Angular album management component (create, edit, drag-reorder photos, set cover) in frontend/src/app/features/dashboard/album-manager.component.ts
- [ ] T128 [US5] Create Angular photo edit dialog (title, description, tags, visibility, featured toggle) in frontend/src/app/features/dashboard/photo-edit-dialog.component.ts
- [ ] T129 [US5] Create Angular delete confirmation dialog with storage impact display in frontend/src/app/shared/components/confirm-delete-dialog.component.ts

**Checkpoint**: Photo library displays all photos → Albums created/managed → Metadata editable → Visibility changeable → Search by tags works → Deletion frees storage

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: Security hardening, social service scaffold, performance, documentation, E2E tests

- [ ] T130 Create social-service skeleton with Like, Comment, Follow entities and empty handlers in apps/social-service/src/
- [ ] T131 [P] Implement CSP, HSTS, X-Frame-Options security headers in API Gateway in apps/api-gateway/src/main.ts
- [ ] T132 [P] Implement CORS configuration (whitelist frontend origin) in apps/api-gateway/src/main.ts
- [ ] T133 [P] Create CI workflow (build, lint, test, npm audit) in .github/workflows/ci.yml
- [ ] T134 [P] Create CD workflow (build Docker images, deploy) in .github/workflows/cd.yml
- [ ] T135 Create Playwright E2E test: registration → login → presigned upload → view portfolio flow in frontend/test/e2e/full-flow.spec.ts
- [ ] T136 [P] Create Playwright E2E test: anti-download protections verification in frontend/test/e2e/protections.spec.ts
- [ ] T137 [P] Add Swagger/Scalar API documentation decorators to all gateway controllers in apps/api-gateway/src/routes/
- [ ] T138 Run quickstart.md validation (full setup from scratch, verify all access points, Jaeger traces visible)
- [ ] T139 [P] Verify test coverage meets thresholds (>80% lines, >70% branches) across all services

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion — BLOCKS all user stories
- **US1 (Phase 3)**: Depends on Foundational (Phase 2) — auth is prerequisite for US2-US5
- **US2 (Phase 4)**: Depends on US1 (needs auth + photographer entity) + Foundational
- **US3 (Phase 5)**: Depends on US1 (photographer profiles) + US2 (photos to display)
- **US4 (Phase 6)**: Depends on US1 (photographer assignment) + US2 (photo upload events)
- **US5 (Phase 7)**: Depends on US2 (photo entity exists)
- **Polish (Phase 8)**: Depends on all desired user stories being complete

### User Story Dependencies

- **US1 (P1)**: Start after Phase 2 — No dependencies on other stories
- **US2 (P1)**: Start after US1 — Needs auth for upload authorization and billing.check_storage
- **US3 (P2)**: Start after US2 — Needs photos to display in portfolio (can start UI in parallel)
- **US4 (P2)**: Start after US1 — Needs photographer entity, integrates with photo events from US2
- **US5 (P2)**: Start after US2 — Needs photo entity, extends with albums

### Within Each User Story

1. Tests MUST be written and FAIL before implementation (TDD)
2. Entities before services
3. Services before controllers
4. Backend before frontend components
5. Core implementation before cross-service integration
6. Story complete before moving to next priority

### Parallel Opportunities

- Phase 1: T003, T004, T005, T006, T007, T008 can all run in parallel
- Phase 2: T011-T018 (shared libs), T019-T025 (gateway), T034-T039 (resilience/tracing/throttler) in parallel
- US1: All test tasks (T040-T046) in parallel, then entities T048-T051 in parallel
- US2: All test tasks (T070-T074) in parallel, then T075+T076 in parallel
- US4 and US5 can potentially run in parallel (different services, minimal overlap)
- Phase 8: T131-T134, T136-T137, T139 can all run in parallel

---

## Parallel Example: US2 — Photo Upload (Presigned URL Flow)

```bash
# TDD: Launch all US2 tests together (they MUST fail):
Task: T070 "Unit test for PhotosService (upload-url, confirm-upload)"
Task: T071 "Unit test for StorageClientService (presigned PUT/GET URLs)"
Task: T072 "Unit test for ImageProcessorService"
Task: T073 "Unit test for file validation (post-upload async)"
Task: T074 "Integration test for presigned URL upload flow"

# Then launch parallel entity/DTO creation:
Task: T075 "Create Photo entity"
Task: T076 "Create photo DTOs (UploadUrlRequest/Response, ConfirmUpload)"

# Then migration (depends on entity):
Task: T077 "Create migration for photos schema"

# Then sequential implementation:
Task: T078 "Implement StorageClientService" (presigned PUT/GET, forcePathStyle)
Task: T079 "Async file validation in worker"
Task: T080 "ImageProcessorService" (Sharp)
Task: T081 "PhotoProcessingProcessor" (BullMQ worker + DLQ)
Task: T082 "PhotosService" (presigned URL flow, depends on above)
```

---

## Implementation Strategy

### MVP First (US1 + US2)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (CRITICAL — blocks all stories)
3. Complete Phase 3: US1 — Registration & Login
4. **VALIDATE**: Auth flow works end-to-end
5. Complete Phase 4: US2 — Photo Upload with Protection
6. **STOP AND VALIDATE**: Upload → Process → View with protections → Storage tracked
7. **Deploy/Demo MVP**: Auth + Photo upload + Protections = core value proposition

### Incremental Delivery

1. Setup + Foundational → Foundation ready
2. US1 → Auth works → Internal demo
3. US2 → Upload + Protections → **MVP Release**
4. US3 → Portfolios → Enhanced product
5. US4 → Storage Dashboard → Billing ready
6. US5 → Photo Management → Full Phase 1 complete
7. Polish → Production ready

### Parallel Team Strategy

With multiple developers after Phase 2:

- **Developer A**: US1 (auth) → US3 (portfolio)
- **Developer B**: US2 (photos) → US5 (photo management)
- **Developer C**: US4 (billing/storage) → Phase 8 (polish)

---

## Notes

- [P] tasks = different files, no dependencies
- [Story] label maps task to specific user story for traceability
- Each user story should be independently completable and testable
- TDD: Verify tests FAIL before implementing (Red → Green → Refactor)
- Commit after each task or logical group following conventional commits
- Stop at any checkpoint to validate story independently
- OWASP: Every endpoint validates auth (A01), inputs sanitized (A03), rate limited (A07), audit logged (A09)
- ISO 27001: All login attempts logged (A.8.15), data classified (CONFIDENCIAL/RESTRINGIDO/INTERNO/PUBLICO)
