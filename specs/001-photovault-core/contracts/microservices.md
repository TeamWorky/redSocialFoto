# Microservices Communication Contracts

**Transport**: Redis (NestJS @MessagePattern / @EventPattern)

---

## Message Patterns (Request/Response — sync)

### auth-service

| Pattern | Payload | Response | Called by |
|---------|---------|----------|-----------|
| `auth.validate_token` | `{ token: string }` | `{ valid: boolean, photographerId: string, role: string }` | api-gateway |
| `auth.get_photographer` | `{ id: string }` | `Photographer` object | photos-service, social-service |
| `auth.get_photographers_bulk` | `{ ids: string[] }` | `Photographer[]` | social-service (feed) |

### photos-service

| Pattern | Payload | Response | Called by |
|---------|---------|----------|-----------|
| `photos.get_photo` | `{ id: string, requesterId?: string }` | `Photo` object with access check | social-service |
| `photos.get_storage_usage` | `{ photographerId: string }` | `{ usedBytes: number, breakdown: [] }` | billing-service |
| `photos.get_public_photos` | `{ photographerId: string, limit: number, cursor?: string }` | `Photo[]` | social-service (feed) |

### billing-service

| Pattern | Payload | Response | Called by |
|---------|---------|----------|-----------|
| `billing.check_storage` | `{ photographerId: string, additionalBytes: number }` | `{ allowed: boolean, currentUsage: number, limit: number }` | photos-service (pre-upload) |
| `billing.get_plan` | `{ photographerId: string }` | `StoragePlan` object | auth-service (profile) |

---

## Event Patterns (Fire-and-forget — async)

| Event | Payload | Emitted by | Consumed by |
|-------|---------|------------|-------------|
| `photo.uploaded` | `{ photoId, photographerId, fileSizeBytes }` | photos-service | billing-service (update usage) |
| `photo.deleted` | `{ photoId, photographerId, fileSizeBytes }` | photos-service | billing-service (update usage), social-service (cleanup likes/comments) |
| `photo.processed` | `{ photoId, status: 'completed' \| 'failed' }` | photos-service (worker) | — (websocket notification future) |
| `photographer.registered` | `{ photographerId, email }` | auth-service | billing-service (assign free plan) |
| `storage.threshold_reached` | `{ photographerId, percentage: 80 \| 95 \| 100 }` | billing-service | — (notification future) |

---

## Inter-service Data Flow

```
                          ┌─────────────────┐
                          │   API Gateway   │
                          │  (port 3000)    │
                          └───────┬─────────┘
                                  │ HTTP
                    ┌─────────────┼─────────────┐─────────────┐
                    ▼             ▼             ▼             ▼
             ┌────────────┐┌────────────┐┌────────────┐┌────────────┐
             │auth-service││photos-svc  ││social-svc  ││billing-svc │
             │  (3001)    ││  (3002)    ││  (3003)    ││  (3004)    │
             └─────┬──────┘└─────┬──────┘└─────┬──────┘└─────┬──────┘
                   │             │             │             │
              ┌────┴────┐  ┌────┴────┐   ┌────┴────┐  ┌────┴────┐
              │schema:  │  │schema:  │   │schema:  │  │schema:  │
              │auth     │  │photos   │   │social   │  │billing  │
              └─────────┘  └─────────┘   └─────────┘  └─────────┘
                   └────────────┴──────────────┴────────────┘
                                    │
                              ┌─────┴─────┐
                              │PostgreSQL │
                              │(1 instance│
                              │4 schemas) │
                              └───────────┘
```
