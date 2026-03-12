# Data Model: PhotoVault Core

**Branch**: `001-photovault-core` | **Date**: 2026-03-12

## Entity Relationship Diagram (Text)

```
┌──────────────┐     1:N     ┌──────────────┐     N:1     ┌──────────────┐
│  StoragePlan │────────────▶│ Photographer │◀────────────│   Session    │
└──────────────┘             └──────┬───────┘             └──────────────┘
                                    │
                          1:N       │       1:N
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
             ┌──────────┐   ┌──────────────┐  ┌──────────┐
             │   Photo  │   │    Album     │  │  Follow  │
             └────┬─────┘   └──────┬───────┘  └──────────┘
                  │                │
                  │    N:M         │ 1:N
                  ├────────────────┘
                  │
            ┌─────┼─────┐
            ▼           ▼
     ┌──────────┐ ┌──────────┐
     │   Like   │ │ Comment  │
     └──────────┘ └──────────┘
```

## Entities

### Photographer (auth-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| email | varchar(255) | UNIQUE, NOT NULL | Cifrado clasificación RESTRINGIDO |
| passwordHash | varchar(255) | NULL (OAuth users) | bcrypt/argon2. Clasificación RESTRINGIDO |
| displayName | varchar(100) | NOT NULL | |
| username | varchar(30) | UNIQUE, NULL | Slug del portafolio. Se configura en US3 (Portfolio). 3-30 chars, alfanumérico + guiones |
| bio | text | NULL, max 500 chars | |
| profilePictureUrl | varchar(500) | NULL | URL a S3 optimized |
| socialLinks | jsonb | NULL | Array de {platform, url}, max 5 |
| oauthProvider | varchar(20) | NULL | 'google', 'apple', NULL |
| oauthProviderId | varchar(255) | NULL | ID del proveedor OAuth |
| storagePlanId | UUID | FK → StoragePlan | |
| storageUsedBytes | bigint | NOT NULL, DEFAULT 0 | Calculado, actualizado async |
| role | enum | NOT NULL, DEFAULT 'photographer' | 'photographer', 'admin' |
| isActive | boolean | NOT NULL, DEFAULT true | Soft delete |
| emailVerified | boolean | NOT NULL, DEFAULT false | |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | |

**Validaciones**:
- email: formato válido, confirmación por email
- password: min 8 chars, 1 mayúscula, 1 número, 1 especial
- username: ^[a-z0-9][a-z0-9-]{1,28}[a-z0-9]$, no palabras reservadas

**Índices**: email (unique), username (unique), oauthProvider+oauthProviderId (unique)

---

### Photo (photos-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| photographerId | UUID | FK → Photographer, NOT NULL | Owner |
| title | varchar(100) | NULL | |
| description | text | NULL, max 1000 chars | |
| tags | varchar(50)[] | NULL, max 20 | Array de tags |
| visibility | enum | NOT NULL, DEFAULT 'private' | 'public', 'private', 'link_only' |
| originalKey | varchar(500) | NOT NULL | S3 key del original. Clasificación CONFIDENCIAL |
| optimizedKey | varchar(500) | NOT NULL | S3 key optimizado |
| thumbnailKey | varchar(500) | NOT NULL | S3 key thumbnail |
| fileSizeBytes | bigint | NOT NULL | Tamaño del original |
| mimeType | varchar(50) | NOT NULL | 'image/jpeg', 'image/png', 'image/webp' |
| width | integer | NOT NULL | Píxeles |
| height | integer | NOT NULL | Píxeles |
| shareToken | varchar(64) | UNIQUE, NULL | Token para visibility='link_only' |
| isFeatured | boolean | NOT NULL, DEFAULT false | Destacado en portafolio |
| processingStatus | enum | NOT NULL, DEFAULT 'pending' | 'pending', 'processing', 'completed', 'failed' |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | |

**Índices**: photographerId, visibility, tags (GIN), shareToken (unique), createdAt DESC

---

### Album (photos-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| photographerId | UUID | FK → Photographer, NOT NULL | Owner |
| name | varchar(100) | NOT NULL | |
| description | text | NULL, max 500 chars | |
| visibility | enum | NOT NULL, DEFAULT 'private' | 'public', 'private' |
| coverPhotoId | UUID | FK → Photo, NULL | |
| isFeatured | boolean | NOT NULL, DEFAULT false | |
| position | integer | NOT NULL, DEFAULT 0 | Orden en portafolio |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | |

**Índices**: photographerId, visibility

---

### AlbumPhoto (photos-service) — Tabla pivote

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| albumId | UUID | FK → Album, NOT NULL | PK compuesto |
| photoId | UUID | FK → Photo, NOT NULL | PK compuesto |
| position | integer | NOT NULL, DEFAULT 0 | Orden dentro del álbum |
| addedAt | timestamp | NOT NULL, DEFAULT now() | |

---

### Session (auth-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| photographerId | UUID | FK → Photographer, NOT NULL | |
| refreshToken | varchar(500) | NOT NULL | Hashed. Clasificación RESTRINGIDO |
| ipAddress | varchar(45) | NOT NULL | IPv4/IPv6 |
| userAgent | varchar(500) | NULL | |
| expiresAt | timestamp | NOT NULL | |
| isRevoked | boolean | NOT NULL, DEFAULT false | |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |

**Índices**: photographerId, refreshToken (hash), expiresAt

---

### LoginAttempt (auth-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| email | varchar(255) | NOT NULL | |
| ipAddress | varchar(45) | NOT NULL | |
| success | boolean | NOT NULL | |
| failureReason | varchar(100) | NULL | 'invalid_password', 'account_locked', etc. |
| createdAt | timestamp | NOT NULL, DEFAULT now() | ISO 27001 A.8.15 audit log |

**Índices**: email+createdAt (para rate limiting), ipAddress+createdAt

---

### StoragePlan (billing-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| name | varchar(50) | UNIQUE, NOT NULL | 'free', 'basic', 'pro', 'enterprise' |
| storageLimitBytes | bigint | NOT NULL | Límite en bytes |
| priceMonthly | decimal(10,2) | NOT NULL | En USD |
| priceYearly | decimal(10,2) | NULL | Descuento anual |
| maxPhotoSizeMb | integer | NOT NULL, DEFAULT 50 | |
| maxBatchUpload | integer | NOT NULL, DEFAULT 20 | |
| features | jsonb | NULL | Features adicionales por plan |
| isActive | boolean | NOT NULL, DEFAULT true | |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |

**Seed data**:
- Free: 1 GB, $0/mes
- Basic: 10 GB, $9.99/mes
- Pro: 50 GB, $24.99/mes
- Enterprise: 500 GB, $99.99/mes

---

### Like (social-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| photographerId | UUID | NOT NULL | PK compuesto. Quien da like |
| photoId | UUID | NOT NULL | PK compuesto |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |

**Constraint**: UNIQUE(photographerId, photoId)

---

### Comment (social-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| id | UUID | PK, auto-generated | |
| photographerId | UUID | NOT NULL | Autor del comentario |
| photoId | UUID | NOT NULL | |
| content | text | NOT NULL, max 1000 chars | Sanitizado (OWASP A03) |
| isDeleted | boolean | NOT NULL, DEFAULT false | Soft delete |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |
| updatedAt | timestamp | NOT NULL, DEFAULT now() | |

**Índices**: photoId+createdAt DESC

---

### Follow (social-service)

| Field | Type | Constraints | Notes |
|-------|------|-------------|-------|
| followerId | UUID | NOT NULL | PK compuesto. Quien sigue |
| followingId | UUID | NOT NULL | PK compuesto. A quien sigue |
| createdAt | timestamp | NOT NULL, DEFAULT now() | |

**Constraints**: UNIQUE(followerId, followingId), CHECK(followerId != followingId)

---

## Redis Data Structures

| Key Pattern | Type | TTL | Purpose |
|-------------|------|-----|---------|
| `auth:blacklist:{jti}` | string | 15 min | JWT blacklist (logout) |
| `auth:rate:{email}` | sorted set | 1 min | Login rate limiting |
| `auth:lockout:{email}` | string | 15 min | Account lockout |
| `portfolio:{username}` | hash | 5 min | Cache de portafolio público |
| `storage:{photographerId}` | string | 5 min | Cache de uso de almacenamiento |
| `photo:signed:{photoId}` | string | 15 min | Cache de presigned URL activa |
| `feed:{photographerId}` | list | 10 min | Cache del feed personalizado |

## Schema Ownership per Service

| Service | Schemas/Tables |
|---------|---------------|
| auth-service | Photographer, Session, LoginAttempt |
| photos-service | Photo, Album, AlbumPhoto |
| social-service | Like, Comment, Follow |
| billing-service | StoragePlan |
