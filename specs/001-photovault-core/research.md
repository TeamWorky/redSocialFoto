# Research: PhotoVault Core

**Branch**: `001-photovault-core` | **Date**: 2026-03-12

## Decision Log

### D-01: Frontend Framework

- **Decision**: Angular 21 con Standalone Components
- **Rationale**: Consistencia con el equipo (experiencia previa en Angular). Standalone components simplifican la arquitectura. Angular Material 21 para UI consistente. SSR con Angular Universal para portafolios públicos (SEO).
- **Alternatives considered**:
  - Next.js: Mejor SSR nativo, pero el equipo tiene más experiencia en Angular.
  - Nuxt: Buen framework pero menor ecosistema para este caso de uso.

### D-02: Backend Architecture

- **Decision**: NestJS monorepo con microservicios (Nx o NestJS workspaces)
- **Rationale**: Permite escalar servicios independientemente. El monorepo facilita compartir código (DTOs, interfaces, entidades). NestJS Microservices soporta múltiples transportes (TCP, Redis, RabbitMQ). Se empieza con comunicación TCP/Redis y se puede migrar a message broker si escala.
- **Alternatives considered**:
  - NestJS monolítico + BullMQ: Más simple pero limita escalabilidad futura.
  - Mix NestJS + Python: Complejidad operacional innecesaria; sharp (Node.js) cubre procesamiento de imágenes.

### D-03: Microservices Split

- **Decision**: 4 servicios + 1 gateway
- **Rationale**: Separación por dominio de negocio. Cada servicio es independiente y deployable.

| Servicio | Responsabilidad | Puerto |
|----------|----------------|--------|
| `api-gateway` | Routing, rate limiting, auth validation, CORS | 3000 |
| `auth-service` | Registro, login, OAuth, JWT, sesiones | 3001 |
| `photos-service` | Upload S3, procesamiento imágenes, restricciones, álbumes | 3002 |
| `social-service` | Likes, comentarios, follows, feed | 3003 |
| `billing-service` | Planes, almacenamiento, pagos | 3004 |

- **Alternatives considered**:
  - 3 servicios (auth+billing, photos, social): Menos granular pero acopla auth con billing.
  - 6+ servicios: Over-engineering para el MVP.

### D-04: Database Strategy

- **Decision**: PostgreSQL (una DB por servicio lógico, schemas separados) + Redis (cache, sesiones, colas)
- **Rationale**: PostgreSQL para ACID, full-text search (fotógrafos), y JSONB (metadata flexible). Schemas separados mantienen aislamiento sin múltiples instancias DB en desarrollo. Redis para cache de portafolios, sesiones JWT blacklist, y BullMQ queues.
- **Alternatives considered**:
  - DB por servicio (instancias separadas): Demasiada complejidad operacional para MVP.
  - MongoDB: Flexible pero pierde integridad referencial necesaria para pagos y auth.

### D-05: Image Processing

- **Decision**: Sharp (Node.js) para procesamiento de imágenes
- **Rationale**: Sharp es la librería más performante de Node.js para imágenes. Soporta resize, format conversion, watermark overlay, y metadata extraction. Se ejecuta en workers BullMQ dentro del photos-service para no bloquear requests.
- **Alternatives considered**:
  - Pillow (Python): Requiere servicio separado en otro lenguaje.
  - ImageMagick: Más complejo de configurar, mayor superficie de ataque (OWASP).

### D-06: Storage & CDN

- **Decision**: AWS S3 (producción) + MinIO (desarrollo/testing local) + CloudFront (CDN para thumbnails públicos)
- **Rationale**: S3 presigned URLs para acceso controlado en producción. MinIO como alternativa standalone S3-compatible para desarrollo local y tests (Docker container, misma API S3). CloudFront para servir thumbnails públicos con baja latencia. Estructura: `s3://photovault/{userId}/originals/`, `s3://photovault/{userId}/optimized/`, `s3://photovault/{userId}/thumbnails/`.
- **Entornos**:
  - **Local/Test**: MinIO en Docker (`minio/minio`), endpoint configurable via env vars. Mismo SDK `@aws-sdk/client-s3` con endpoint override.
  - **Staging/Prod**: AWS S3 real con IAM roles y policies.
  - La abstracción es transparente: el código usa el SDK de S3, solo cambia el endpoint.
- **Alternatives considered**:
  - LocalStack: Más completo pero más pesado para solo S3.
  - Cloudflare R2: Más barato pero menor ecosistema SDK.
  - Fake S3 (mock en tests): No valida comportamiento real de presigned URLs.

### D-07: Authentication

- **Decision**: JWT (access + refresh tokens) + Passport.js strategies
- **Rationale**: Access token (15 min) + refresh token (7 días). Redis blacklist para logout inmediato. Passport.js para OAuth (Google, Apple). Consistente con la constitución (OWASP A07).
- **Alternatives considered**:
  - Session-based: No escala bien con microservicios.
  - Auth0/Firebase Auth: Dependencia externa, costos a escala.

### D-08: Anti-Download & Anti-Screenshot

- **Decision**: Enfoque multicapa (server + client)
- **Rationale**:
  - **Server-side**: Presigned URLs (15 min), servir solo versión optimizada (60%+ compresión), watermark server-side con Sharp, validación de referer/origin.
  - **Client-side**: Canvas rendering con overlay dinámico, CSS protections (user-select: none, pointer-events), context menu suppression, drag prevention.
  - Ninguna protección client-side es infalible; la defensa real es nunca exponer el original.
- **Alternatives considered**:
  - DRM (Widevine): Excesivamente complejo para imágenes, orientado a video.
  - Invisible watermarking (steganography): Fase futura, no MVP.

### D-09: Testing Strategy

- **Decision**: Vitest (unit) + Supertest (integration) + Playwright (E2E)
- **Rationale**: Vitest para tests unitarios rápidos (compatible con Jest API). Supertest para tests de endpoints API. Playwright para E2E (flujos de auth, upload, portafolio). Cobertura mínima: 80% líneas, 70% branches (constitución).
- **Alternatives considered**:
  - Jest: Más lento que Vitest, mayor consumo de memoria.
  - Cypress: Solo frontend, Playwright cubre más escenarios.

### D-10: Communication Between Services

- **Decision**: Redis como transport para NestJS Microservices (message patterns)
- **Rationale**: Redis ya está en el stack para cache/sesiones. NestJS @MessagePattern y @EventPattern para comunicación sync/async. Bajo overhead, fácil de configurar. Migrable a RabbitMQ/NATS si se necesita en el futuro.
- **Alternatives considered**:
  - TCP: Más simple pero no soporta pub/sub.
  - RabbitMQ: Over-engineering para MVP.
  - gRPC: Mayor performance pero más complejo de mantener.

### D-11: Upload Strategy

- **Decision**: Presigned PUT URL (upload directo a S3/MinIO desde frontend)
- **Rationale**: Con archivos de 50MB y lotes de 20 (= 1GB por batch), el proxy por gateway consume ~3GB RAM por batch (triple buffering). Presigned URL elimina el gateway del path de datos. Flujo de 3 pasos: (1) solicitar URL, (2) PUT directo a S3, (3) confirmar upload. Post-upload validation async (magic bytes en worker). Requiere CORS en MinIO y `forcePathStyle: true`.
- **Alternatives considered**:
  - Proxy por gateway: Consumo de RAM inviable a escala. Descartado.
  - Streaming por gateway (custom Multer): Reduce RAM pero gateway sigue en el path. Complejidad sin beneficio claro vs presigned.

### D-12: Resilience Pattern

- **Decision**: Cockatiel para circuit breaker + BullMQ retry con backoff exponencial
- **Rationale**: Cockatiel es maduro (usado en VS Code), excelente tipado TypeScript, políticas componibles. Se wrappea en `ResilienceService` singleton inyectable. SamplingBreaker(20%, 30s) para servicios no-críticos, ConsecutiveBreaker(5) para auth. Fallback por servicio (cache Redis o permitir con validación posterior).
- **Alternatives considered**:
  - nestjs-resilience: NestJS nativo pero solo 311 stars, documentación pobre del circuit breaker.
  - Custom: Reimplementar state machine es propenso a errores.

### D-13: Eventual Consistency

- **Decision**: BullMQ DLQ pattern + reconciliación periódica (cron cada 6h)
- **Rationale**: No existe DLQ nativo en BullMQ. Se implementa con dos colas: main + DLQ. Después de 4 intentos (exp. backoff), job va a DLQ para log + alerta + retry manual. Cron de reconciliación verifica: storageUsedBytes vs SUM real, fotógrafos sin plan, archivos S3 huérfanos. Transactional Outbox pattern considerado para fase futura.
- **Alternatives considered**:
  - Solo retry sin DLQ: Se pierden eventos después de fallos exhaustos.
  - Transactional Outbox: Garantía transaccional pero over-engineering para MVP.

### D-14: Database Migration Strategy

- **Decision**: Migraciones centralizadas con prefijo numérico en `database/migrations/`
- **Rationale**: Una sola fuente de migraciones para los 4 schemas. Cada migración declara su schema con `SET search_path`. Prefijo numérico garantiza orden. TypeORM advisory lock previene ejecución concurrente. Sin foreign keys cross-schema (aislamiento de dominio).
- **Alternatives considered**:
  - Migraciones por servicio: Complejidad de coordinación en CI/CD y rollback.

### D-15: OWASP Threat Model

- **Decision**: Modelo de amenazas con 21 vectores de ataque identificados y mitigaciones específicas
- **Rationale**: Identificar y mitigar amenazas antes de la implementación. Cada vector mapeado a componente y control específico. Cubre A01 a A10 de OWASP Top 10. Clasificación de datos según ISO 27001 (CONFIDENCIAL, RESTRINGIDO, INTERNO, PÚBLICO).
- **Alternatives considered**:
  - Threat model post-implementación: Demasiado tarde para cambios arquitectónicos.

### D-16: Observability

- **Decision**: OpenTelemetry SDK + nestjs-otel + Jaeger (dev)
- **Rationale**: Auto-instrumentación para HTTP, pg, ioredis, TypeORM, BullMQ. nestjs-otel para decoradores @Span(). W3C traceparent propaga traceId entre servicios. SDK DEBE inicializarse ANTES de NestFactory.create(). ParentBasedSampler 10% en producción.
- **Alternatives considered**:
  - Solo Winston + RequestId: Insuficiente para tracing distribuido entre 5 procesos.
  - DataDog agent: Vendor lock-in, costo a escala.

### D-17: Frontend State Management

- **Decision**: Angular Signals + Services singleton (sin NgRx para MVP)
- **Rationale**: Angular 21 Signals proveen reactividad nativa suficiente. NgRx agrega complejidad significativa para MVP. Services singleton con signal() cubren: auth, upload progress, photo cache, storage. Migrable a NgRx si la complejidad crece.
- **Alternatives considered**:
  - NgRx: Over-engineering para MVP. Actions/reducers/effects/selectors innecesarios.
  - Akita: Menor adopción que NgRx, mismo problema de complejidad.

### D-18: Contract Testing

- **Decision**: Contract types compartidos en libs/common/contracts/ + integration tests
- **Rationale**: Monorepo permite compartir tipos TypeScript entre servicios. Los contract types validan en compilación. Integration tests validan runtime. Sin Pact broker (complejidad de infraestructura).
- **Alternatives considered**:
  - Pact: Broker + CI flow complejo. Innecesario con monorepo y tipos compartidos.

### D-19: Rate Limiting

- **Decision**: @nestjs/throttler con Redis storage (ThrottlerStorageRedisService)
- **Rationale**: Rate limiting distribuido por IP + usuario autenticado. Redis comparte contadores entre instancias del gateway. Login: 5/min por email+IP. API: 100/min por usuario, 30/min anónimo. Upload URL: 20/5min.
- **Alternatives considered**:
  - express-rate-limit: No distribuido sin adapter Redis. Menos integrado con NestJS.
