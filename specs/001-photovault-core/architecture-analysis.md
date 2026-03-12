# Análisis Arquitectónico: PhotoVault Core

**Branch**: `001-photovault-core` | **Fecha**: 2026-03-12
**Propósito**: Análisis detallado de decisiones D-11 a D-21 (diagramas, configuraciones, flujos)
**Log maestro**: Ver [research.md](research.md) para el índice completo de decisiones D-01 a D-21

---

## D-11: Estrategia de Upload — Presigned URL Directo a S3

### Problema
No estaba definido si los archivos pasan por el API Gateway (proxy) o se suben directamente a S3/MinIO. Con archivos de hasta 50MB y lotes de 20 (= 1GB por batch), esta decisión impacta la escalabilidad fundamentalmente.

### Decisión: **Presigned PUT URL (upload directo a S3/MinIO)**

### Análisis

| Aspecto | Proxy por Gateway | Presigned URL Directo |
|---------|-------------------|-----------------------|
| RAM del gateway por batch | ~3GB (triple buffering) | ~0 (solo genera URLs) |
| Ancho de banda | Doble (client→gateway→S3) | Simple (client→S3) |
| Validación de archivo | Síncrona (antes de guardar) | Asíncrona (post-upload) |
| Complejidad frontend | Simple (1 POST) | Media (2 pasos: get URL + PUT) |
| CORS en S3/MinIO | No necesario | Requerido |
| Escalabilidad | Limitada por gateway | Limitada solo por S3 |

### Flujo Definido

```
┌──────────┐     1. POST /photos/upload-url      ┌──────────────┐
│ Frontend │ ──────────────────────────────────▶  │ API Gateway  │
│ (Angular)│                                      │              │
│          │  ◀── { uploads: [{ presignedUrl,     │  → photos-svc│
│          │        photoId, objectKey, expiresIn }] }            │
│          │                                      └──────────────┘
│          │     2. PUT file directamente
│          │ ──────────────────────────────────▶  ┌──────────────┐
│          │                                      │ MinIO / S3   │
│          │  ◀── 200 OK + ETag                   │              │
│          │                                      └──────────────┘
│          │     3. POST /photos/confirm-upload
│          │ ──────────────────────────────────▶  ┌──────────────┐
│          │                                      │ API Gateway  │
│          │  ◀── { photoId, status: processing } │  → photos-svc│
└──────────┘                                      └──────────────┘
                                                         │
                                                    4. BullMQ job
                                                         ▼
                                                  ┌──────────────┐
                                                  │ Sharp Worker  │
                                                  │ (thumbnail +  │
                                                  │  optimized)   │
                                                  └──────────────┘
```

### Endpoints Nuevos

| Endpoint | Método | Descripción |
|----------|--------|-------------|
| `/photos/upload-url` | POST | Genera presigned PUT URL(s). Payload: `{ files: [{ filename, contentType, sizeBytes }], visibility?, albumId? }`. Valida: billing.check_storage, formato, tamaño. Retorna: array de `{ presignedUrl, photoId, objectKey, expiresIn }` |
| `/photos/confirm-upload` | POST | Confirma que el archivo existe en S3. Payload: `{ photoId, etag }`. Valida existencia real en S3, registra en DB, lanza BullMQ job de procesamiento |

### Seguridad (Presigned URLs)

- **Object key generado server-side**: UUID-based, nunca aceptar key del cliente
- **Content-Type bloqueado en la firma**: Solo `image/jpeg`, `image/png`, `image/webp`
- **Expiración corta**: 5 minutos para upload (no 15 como las de lectura)
- **Confirmación obligatoria**: Backend verifica que el objeto realmente existe en S3 antes de registrar
- **Validación async post-upload**: Magic bytes check en el worker de procesamiento
- **Limpieza de huérfanos**: Cron job que elimina archivos en S3 sin registro en DB (cada 1h)

### MinIO en Desarrollo

- Endpoint configurable via env: `S3_ENDPOINT=http://localhost:9000`
- `forcePathStyle: true` obligatorio (MinIO no soporta virtual-hosted)
- CORS configurado en MinIO para permitir PUT desde `http://localhost:4200`
- Mismo SDK `@aws-sdk/client-s3` funciona para ambos entornos

### Alternativa Rechazada
Proxy por gateway: Descartado por consumo de 3GB RAM por batch y doble ancho de banda. Inviable para la meta de 500 uploads concurrentes.

---

## D-12: Patrón de Resiliencia Inter-servicios

### Problema
Con 4 microservicios comunicándose via Redis, no hay manejo de fallos: si billing-service cae, ¿el upload se bloquea indefinidamente?

### Decisión: **Cockatiel (circuit breaker) + BullMQ retry con backoff exponencial + fallback por servicio**

### Análisis de Opciones

| Librería | Pros | Contras |
|----------|------|---------|
| nestjs-resilience | NestJS nativo, decoradores | 311 stars, documentación pobre |
| cockatiel | Maduro (usado en VS Code), tipado excelente | Requiere wrapper manual en NestJS |
| Custom | Sin dependencias | Propenso a errores, reimplementar state machine |

**Cockatiel**: Madurez y confiabilidad > conveniencia de decoradores. Se wrappea en un `ResilienceService` singleton inyectable.

### Estrategia por Servicio

| Servicio destino | Tipo de llamada | Circuit Breaker | Fallback |
|------------------|-----------------|-----------------|----------|
| billing-service (`billing.check_storage`) | Sync (pre-upload) | SamplingBreaker(20%, 30s) | Permitir upload + validar async |
| auth-service (`auth.validate_token`) | Sync (cada request) | ConsecutiveBreaker(5) | Rechazar request (no fallback seguro) |
| auth-service (`auth.get_photographer`) | Sync (info) | SamplingBreaker(20%, 30s) | Cache Redis (TTL 5 min) |
| photos-service (`photos.get_photo`) | Sync (social) | SamplingBreaker(20%, 30s) | Retornar datos parciales |

### Configuración del Circuit Breaker

```
Estado: CLOSED → OPEN → HALF-OPEN → CLOSED/OPEN
- OPEN después de: 20% fallos en ventana de 30s (min 5 requests)
- HALF-OPEN después de: 10s en OPEN
- Test: 1 request en HALF-OPEN, si pasa → CLOSED, si falla → OPEN
```

### Retry Policy (BullMQ Jobs)

```
- Intentos: 4 (1 original + 3 retries)
- Backoff: exponencial con jitter
- Delays: ~1s, ~2s, ~4s (base 1000ms)
- Después de 4 fallos: DLQ (ver D-13)
```

### Timeout por Servicio

| Servicio | Timeout |
|----------|---------|
| auth.validate_token | 3s |
| auth.get_photographer | 5s |
| billing.check_storage | 5s |
| photos.get_photo | 5s |
| Image processing (Sharp) | 60s |

---

## D-13: Consistencia Eventual y Dead Letter Queue

### Problema
Con eventos async (`photo.uploaded`, `photo.deleted`, `photographer.registered`), ¿qué pasa si un consumer no procesa un evento? Puede haber inconsistencia entre servicios.

### Decisión: **BullMQ retry + DLQ pattern + reconciliación periódica**

### Estrategia

#### Eventos Críticos (requieren garantía de entrega)

| Evento | Productor | Consumidor | Impacto si se pierde |
|--------|-----------|------------|----------------------|
| `photographer.registered` | auth-service | billing-service | Fotógrafo sin plan asignado |
| `photo.uploaded` | photos-service | billing-service | Uso de storage no actualizado |
| `photo.deleted` | photos-service | billing-service, social-service | Storage no liberado, likes/comments huérfanos |

#### Implementación: Patrón de Dos Colas

```
Main Queue                          DLQ (Dead Letter Queue)
┌─────────────┐   4 intentos       ┌─────────────────────┐
│ photo.upload│──────────────────▶  │ photo.upload.dlq    │
│ .event      │   (exp. backoff)   │                     │
└─────────────┘                    │ → Log + Alerta      │
                                   │ → Persistir en DB   │
                                   │ → Retry manual/auto │
                                   └─────────────────────┘
```

#### Configuración BullMQ

```
defaultJobOptions:
  attempts: 4
  backoff: { type: 'exponential', delay: 1000 }
  removeOnFail: { age: 86400, count: 10000 }  // Retener 24h o 10K jobs
```

#### Reconciliación Periódica (Safety Net)

Cron job cada 6 horas que verifica consistencia:
1. **Storage**: Comparar `storageUsedBytes` en Photographer con `SUM(fileSizeBytes)` real en Photos
2. **Planes**: Verificar que todo Photographer tiene un StoragePlan asignado
3. **Huérfanos S3**: Archivos en S3 sin registro en DB → marcar para limpieza

### Patrón de Outbox (Fase futura)
Para garantía de entrega transaccional, migrar a Transactional Outbox pattern:
- Guardar evento en tabla `outbox` dentro de la misma transacción que el cambio
- Worker poll/CDC publica eventos desde la tabla
- **No MVP**: El patrón DLQ + reconciliación es suficiente para ~1000 fotógrafos

---

## D-14: Estrategia de Migración Multi-Schema

### Problema
4 schemas en 1 PostgreSQL. ¿Cada servicio ejecuta sus propias migraciones? ¿Hay orden? ¿Locks?

### Decisión: **Migraciones centralizadas con prefijo de schema + orden explícito**

### Estrategia

```
database/
├── migrations/
│   ├── 001-create-schemas.ts           # Crea los 4 schemas
│   ├── 002-auth-photographers.ts       # auth.photographers
│   ├── 003-auth-sessions.ts            # auth.sessions
│   ├── 004-auth-login-attempts.ts      # auth.login_attempts
│   ├── 005-photos-photos.ts            # photos.photos
│   ├── 006-photos-albums.ts            # photos.albums
│   ├── 007-billing-storage-plans.ts    # billing.storage_plans
│   ├── 008-social-likes.ts             # social.likes
│   ├── 009-social-comments.ts          # social.comments
│   ├── 010-social-follows.ts           # social.follows
│   └── 011-billing-seed-plans.ts       # Seed: Free, Basic, Pro, Enterprise
├── seeders/
│   └── admin.seeder.ts                 # admin@photovault.com
└── typeorm.config.ts                   # Config compartida
```

### Reglas

1. **Una sola fuente de migraciones**: `database/migrations/` en la raíz del monorepo
2. **Prefijo numérico**: Garantiza orden de ejecución
3. **Cada migración declara schema**: `SET search_path TO 'auth'` al inicio
4. **Lock advisory**: TypeORM usa advisory lock por defecto (`migrationsTransactionMode: 'all'`)
5. **CI/CD**: `npm run migration:run` se ejecuta una vez antes de iniciar cualquier servicio
6. **Rollback**: `npm run migration:revert` revierte la última migración (una a la vez)
7. **Sin FKs cross-schema**: Los servicios NO crean foreign keys entre schemas (aislamiento de dominio)

### Entornos

| Entorno | Estrategia |
|---------|-----------|
| Local | `npm run migration:run` manual o en `docker-compose up` |
| CI | Step del pipeline antes de tests de integración |
| Staging/Prod | Step del CD pipeline con rollback automático si falla |

---

## D-15: Modelo de Amenazas OWASP

### Threat Model: PhotoVault Core

| ID | Categoría OWASP | Vector de Ataque | Componente | Mitigación | Prioridad |
|----|------------------|-------------------|------------|------------|-----------|
| TM-01 | A01 Broken Access Control | Acceso a foto privada de otro usuario | photos-service | Verificar `photographerId` en cada query. Guard a nivel de servicio. Presigned URLs scoped al usuario | CRÍTICA |
| TM-02 | A01 Broken Access Control | Manipulación de presigned URL para acceder a otros archivos | photos-service/S3 | Object key generado server-side (UUID). URL firmada incluye key exacto | CRÍTICA |
| TM-03 | A02 Cryptographic Failures | Passwords almacenados en texto plano o hash débil | auth-service | bcrypt con salt rounds ≥ 12 (o argon2id) | CRÍTICA |
| TM-04 | A02 Cryptographic Failures | JWT secret débil o hardcodeado | auth-service | Secret de env variable, min 256 bits, rotación planificada | CRÍTICA |
| TM-05 | A03 Injection | SQL injection via búsqueda de fotos/tags | photos-service | TypeORM parametrizado. NUNCA concatenar queries. GIN index con parámetros | ALTA |
| TM-06 | A03 Injection | XSS via título/descripción/comentarios | social-service, photos-service | Validación de estructura con class-validator. Sanitización HTML con sanitize-html (server-side). CSP headers. No renderizar HTML raw en frontend | ALTA |
| TM-07 | A04 Insecure Design | SSRF via URL de imagen o social link | auth-service (socialLinks) | Validar URLs con allowlist de protocolos (https only). No hacer fetch server-side de URLs de usuario | ALTA |
| TM-08 | A04 Insecure Design | Path traversal en S3 object keys | photos-service | Keys generados server-side como UUID. Validar que key no contiene `..` o `/` | ALTA |
| TM-09 | A05 Security Misconfiguration | Headers de seguridad faltantes | api-gateway | Helmet (CSP, HSTS, X-Frame-Options, X-Content-Type-Options). CORS whitelist | MEDIA |
| TM-10 | A05 Security Misconfiguration | MinIO/S3 bucket público | Infraestructura | Bucket policy: private por defecto. Solo presigned URLs para acceso | CRÍTICA |
| TM-11 | A06 Vulnerable Components | Dependencias con CVEs conocidos | Todas | `npm audit` en CI. Renovate/Dependabot. Lock file | MEDIA |
| TM-12 | A07 Auth Failures | Brute force de login | auth-service | Rate limiting: 5 intentos/min + lockout 15 min. Redis sorted set para tracking | ALTA |
| TM-13 | A07 Auth Failures | Refresh token robado | auth-service | Refresh token hashed en DB. Rotación en cada uso. Revocación por sesión | ALTA |
| TM-14 | A07 Auth Failures | JWT usado después de logout | auth-service/gateway | Redis blacklist con TTL = tiempo restante del token | ALTA |
| TM-15 | A08 Software/Data Integrity | Archivo malicioso disfrazado como imagen | photos-service | Validación de magic bytes (no solo MIME type). Sharp rechaza archivos no-imagen | ALTA |
| TM-16 | A08 Software/Data Integrity | Presigned URL re-usada | photos-service | Expiración 5 min (upload) / 15 min (lectura). Idempotency en confirm-upload | MEDIA |
| TM-17 | A09 Security Logging | Login attempts no auditados | auth-service | LoginAttempt entity con email, IP, timestamp, resultado. Retención 90 días | ALTA |
| TM-18 | A09 Security Logging | Acciones admin no rastreables | api-gateway | Winston audit log con RequestId, userId, action, timestamp en cada operación | MEDIA |
| TM-19 | A10 SSRF | Redirect en presigned URL | photos-service | Presigned URLs no aceptan redirects. Validar host del endpoint S3 | BAJA |
| TM-20 | Content Protection | Descarga directa de imagen original | photos-service | NUNCA exponer originalKey. Solo servir optimizedKey/thumbnailKey via presigned URL | CRÍTICA |
| TM-21 | Content Protection | Captura de pantalla | frontend | Canvas overlay con watermark dinámico. CSS user-select:none. No es infalible (documentado) | MEDIA |

### Datos Clasificados (ISO 27001)

| Clasificación | Datos | Tratamiento |
|---------------|-------|-------------|
| CONFIDENCIAL | Fotos originales (S3 keys) | Nunca expuestos. Solo acceso interno del worker |
| RESTRINGIDO | Passwords (hash), emails, tokens, IPs | Cifrado en reposo. Acceso auditado. Retención limitada |
| INTERNO | Metadata de fotos, configuración | Acceso autenticado. Logs de acceso |
| PÚBLICO | Portafolios, fotos públicas (optimizadas) | Presigned URLs. Watermark. Sin originales |

---

## D-16: Estrategia de Observabilidad Distribuida

### Problema
Con 5 procesos (gateway + 4 servicios), debuggear sin correlation IDs ni tracing distribuido será muy difícil.

### Decisión: **OpenTelemetry (SDK) + nestjs-otel (decoradores) + Jaeger (dev) / OTLP collector (prod)**

### Stack de Observabilidad

| Capa | Herramienta | Propósito |
|------|-------------|-----------|
| Traces | OpenTelemetry SDK + nestjs-otel | Tracing distribuido entre servicios |
| Logs | Winston + correlation ID | Logs estructurados con traceId |
| Métricas | OpenTelemetry Metrics | Request count, latency, error rate |
| Visualización (dev) | Jaeger (Docker) | UI de traces en desarrollo |
| Exportador | OTLP/HTTP | Compatible con Jaeger, Grafana Tempo, DataDog |

### Instrumentación Automática

```
@opentelemetry/auto-instrumentations-node  → HTTP, Express, ioredis, pg
opentelemetry-instrumentation-typeorm       → Queries SQL
@appsignal/opentelemetry-instrumentation-bullmq → Jobs async
nestjs-otel                                 → @Span() decoradores NestJS
```

### Propagación de Contexto

```
Frontend (X-Request-Id header)
    │
    ▼
API Gateway (crea traceId si no existe, propaga via W3C traceparent)
    │
    ├──▶ auth-service (hereda traceId)
    ├──▶ photos-service (hereda traceId)
    │        │
    │        └──▶ BullMQ job (hereda traceId via instrumentación)
    │                │
    │                └──▶ S3/MinIO (hereda traceId)
    ├──▶ billing-service (hereda traceId)
    └──▶ social-service (hereda traceId)
```

### Regla Crítica
El SDK de OpenTelemetry **DEBE** inicializarse ANTES de `NestFactory.create()` en cada `main.ts`. De lo contrario, los patches de auto-instrumentación no funcionan.

### Docker Compose (dev)
Agregar Jaeger al `docker-compose.yml`:
```yaml
jaeger:
  image: jaegertracing/all-in-one:latest
  ports:
    - "16686:16686"   # UI
    - "4318:4318"     # OTLP/HTTP
```

### Sampling (Producción)
- ParentBasedSampler con ratio 10% en root
- Todas las child spans heredan la decisión del parent
- Logging: 100% (no se muestrea, siempre completo)

---

## D-17: Estrategia de Estado del Frontend (Angular)

### Problema
Angular 21 con Standalone Components no define cómo manejar el estado de la aplicación: auth tokens, datos en cache, estado de uploads.

### Decisión: **Angular Signals + Services singleton (sin NgRx para MVP)**

### Justificación
- NgRx agrega complejidad significativa (actions, reducers, effects, selectors) para un MVP
- Angular 21 Signals proveen reactividad nativa suficiente
- Services singleton con `inject()` + `signal()` cubren: auth state, upload progress, photo cache
- Migrable a NgRx si la complejidad crece en fases futuras

### Servicios de Estado

| Servicio | Responsabilidad | Signals |
|----------|----------------|---------|
| `AuthStateService` | JWT tokens, usuario actual, estado de auth | `currentUser`, `isAuthenticated`, `isLoading` |
| `UploadStateService` | Progreso de uploads, cola de archivos | `uploadQueue`, `currentUpload`, `uploadProgress` |
| `PhotoCacheService` | Cache de fotos cargadas, presigned URLs | `cachedPhotos`, `activePresignedUrls` |
| `StorageStateService` | Uso de almacenamiento, plan actual | `storageUsage`, `currentPlan`, `warnings` |

### HTTP Interceptor Chain

```
Request → AuthInterceptor (agregar Bearer token)
       → RequestIdInterceptor (agregar X-Request-Id)
       → ErrorInterceptor (manejar 401 → refresh, 403, 5xx)
       → LoadingInterceptor (señal de loading global)
```

### Manejo de Token Refresh

```
401 recibido → Pausar requests pendientes
            → POST /auth/refresh con refreshToken
            → Si OK: actualizar tokens, reintentar requests pausados
            → Si falla: logout, redirect a /auth/login
```

---

## D-18: Testing de Contratos entre Servicios

### Problema
Los message patterns están definidos en contracts/microservices.md pero nada valida que productor y consumidor respeten el contrato en runtime.

### Decisión: **Contract tests con Supertest + schemas compartidos (sin Pact para MVP)**

### Justificación
- Pact agrega un broker, infraestructura, y flujo de CI complejo
- Con un monorepo NestJS, los DTOs son compartidos en `libs/common`
- Los contract tests se implementan como integration tests que validan:
  1. El productor envía el payload correcto
  2. El consumidor procesa el payload correctamente

### Implementación

```
libs/common/
├── contracts/                    # NUEVO
│   ├── auth.contracts.ts         # Tipos para auth.validate_token, etc.
│   ├── photos.contracts.ts       # Tipos para photos.get_photo, etc.
│   ├── billing.contracts.ts      # Tipos para billing.check_storage, etc.
│   └── events.contracts.ts       # Tipos para photo.uploaded, etc.
```

Cada contrato define:
- **Pattern** (string): `'auth.validate_token'`
- **Payload** (tipo): `ValidateTokenPayload`
- **Response** (tipo): `ValidateTokenResponse`

Los tests de integración validan que:
1. El servicio acepta el payload definido en el contrato
2. El servicio retorna el response definido en el contrato
3. Campos requeridos están presentes
4. Tipos de datos son correctos

### Validación en CI
- Shared contract types → compilación TypeScript falla si hay incompatibilidad
- Integration tests validan runtime behavior
- Si un servicio cambia su API → el contrato type falla en compilación

---

## D-19: Rate Limiting Distribuido

### Problema
¿El rate limiting es por IP, por usuario, o ambos? ¿Funciona con múltiples instancias del gateway?

### Decisión: **@nestjs/throttler con Redis storage + doble capa (IP + usuario)**

### Configuración

| Recurso | Límite | Ventana | Por |
|---------|--------|---------|-----|
| Login | 5 intentos | 1 minuto | Email + IP |
| API general | 100 requests | 1 minuto | Usuario autenticado |
| API general (anónimo) | 30 requests | 1 minuto | IP |
| Upload URL | 20 requests | 5 minutos | Usuario |
| Password reset | 3 requests | 15 minutos | Email |

### Redis como Storage
`@nestjs/throttler` con `ThrottlerStorageRedisService` para estado distribuido. Todas las instancias del gateway comparten los contadores via Redis.

### Headers de Respuesta
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1709251200 (Unix timestamp)
```

---

## D-20: Versionado de API

### Decisión: **URI versioning (`/api/v1/`) + deprecation headers**

- Versión actual: `v1`
- Cuando se necesite v2: nuevo path `/api/v2/`, v1 sigue funcionando con header `Deprecation: true`
- Deadline de deprecación: mínimo 6 meses
- **Para MVP**: Solo v1, sin over-engineering

---

## D-21: Backup y Recovery

### Decisión: **PostgreSQL pg_dump diario + S3 versioning**

| Componente | Estrategia | RPO | RTO |
|------------|-----------|-----|-----|
| PostgreSQL | pg_dump diario a S3 | 24h | 1h |
| Redis | No backup (cache/efímero) | N/A | Reconstruir |
| S3/MinIO | S3 versioning habilitado | 0 (versionado) | Inmediato |

**Para MVP local**: No implementar. Documentar para producción.

---

## Resumen de Decisiones

| ID | Decisión | Impacto |
|----|----------|---------|
| D-11 | Presigned URL para uploads directos a S3 | Nuevo flujo de 3 pasos, 2 endpoints nuevos |
| D-12 | Cockatiel para circuit breaker | ResilienceService singleton, timeout/fallback por servicio |
| D-13 | BullMQ DLQ + reconciliación periódica | 2 colas por evento crítico, cron de consistencia |
| D-14 | Migraciones centralizadas con prefijo | database/migrations/ ordenadas numéricamente |
| D-15 | Threat model OWASP con 21 vectores | Validaciones específicas por componente |
| D-16 | OpenTelemetry + nestjs-otel + Jaeger | Tracing distribuido, init antes de NestFactory |
| D-17 | Angular Signals (sin NgRx) | Services singleton con signals reactivos |
| D-18 | Contract types compartidos + integration tests | libs/common/contracts/ con tipos TypeScript |
| D-19 | @nestjs/throttler con Redis storage | Rate limiting distribuido por IP + usuario |
| D-20 | URI versioning /api/v1/ | Solo v1 para MVP |
| D-21 | pg_dump + S3 versioning | Solo documentado, no implementado en MVP |
