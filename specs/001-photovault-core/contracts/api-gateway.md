# API Contracts: API Gateway

**Base URL**: `/api/v1`
**Auth**: Bearer JWT (access token) unless marked PUBLIC

---

## Auth Endpoints (proxied to auth-service)

### POST /auth/register — PUBLIC
Register a new photographer account.

**Request**:
```json
{
  "email": "string (required, valid email)",
  "password": "string (required, min 8 chars, 1 upper, 1 number, 1 special)",
  "displayName": "string (required, max 100)"
}
```

**Response 201**:
```json
{
  "id": "uuid",
  "email": "string",
  "displayName": "string",
  "username": "string | null",
  "emailVerified": false,
  "accessToken": "string (JWT, 15 min — restricted access until email verified)",
  "refreshToken": "string (7 days)",
  "message": "Verification email sent"
}
```

**Errors**: 400 (validation), 409 (email exists)

---

### POST /auth/login — PUBLIC
Authenticate and receive tokens.

**Request**:
```json
{
  "email": "string (required)",
  "password": "string (required)"
}
```

**Response 200**:
```json
{
  "accessToken": "string (JWT, 15 min)",
  "refreshToken": "string (7 days)",
  "photographer": {
    "id": "uuid",
    "email": "string",
    "displayName": "string",
    "username": "string | null"
  }
}
```

**Errors**: 401 (invalid credentials), 423 (account locked), 429 (rate limited)

---

### POST /auth/oauth/:provider — PUBLIC
OAuth login/register (provider: google | apple).

**Request**:
```json
{
  "idToken": "string (required, OAuth ID token)"
}
```

**Response 200**: Same as POST /auth/login

---

### POST /auth/refresh — PUBLIC
Refresh access token.

**Request**:
```json
{
  "refreshToken": "string (required)"
}
```

**Response 200**:
```json
{
  "accessToken": "string (JWT, 15 min)",
  "refreshToken": "string (new, 7 days)"
}
```

**Errors**: 401 (invalid/expired refresh token)

---

### POST /auth/logout — AUTH REQUIRED
Invalidate current session.

**Response 200**:
```json
{ "message": "Logged out successfully" }
```

---

### POST /auth/forgot-password — PUBLIC
Request password reset email.

**Request**:
```json
{ "email": "string (required)" }
```

**Response 200**:
```json
{ "message": "If the email exists, a reset link has been sent" }
```

---

### POST /auth/reset-password — PUBLIC
Reset password with token.

**Request**:
```json
{
  "token": "string (required, from email)",
  "newPassword": "string (required, same validation as register)"
}
```

**Response 200**:
```json
{ "message": "Password reset successfully" }
```

---

## Profile Endpoints (proxied to auth-service)

### GET /profile/me — AUTH REQUIRED
Get current photographer profile.

**Response 200**:
```json
{
  "id": "uuid",
  "email": "string",
  "displayName": "string",
  "username": "string | null",
  "bio": "string | null",
  "profilePictureUrl": "string | null",
  "socialLinks": [{ "platform": "string", "url": "string" }],
  "storagePlan": { "name": "string", "storageLimitBytes": "number" },
  "storageUsedBytes": "number",
  "createdAt": "ISO 8601"
}
```

---

### PATCH /profile/me — AUTH REQUIRED
Update profile.

**Request** (all optional):
```json
{
  "displayName": "string (max 100)",
  "username": "string (3-30 chars, alphanumeric + hyphens)",
  "bio": "string (max 500)",
  "socialLinks": [{ "platform": "string", "url": "string" }]
}
```

**Response 200**: Updated profile object
**Errors**: 400 (validation), 409 (username taken)

---

### PUT /profile/me/picture — AUTH REQUIRED
Upload profile picture. Multipart form data.

**Request**: `multipart/form-data` with field `picture` (JPEG/PNG/WebP, max 5MB)

**Response 200**:
```json
{ "profilePictureUrl": "string" }
```

---

## Photo Endpoints (proxied to photos-service)

### POST /photos/upload-url — AUTH REQUIRED
Request presigned PUT URLs for direct upload to S3/MinIO.

**Request**:
```json
{
  "files": [  // max 20 items per batch (StoragePlan.maxBatchUpload)
    {
      "filename": "string (required)",
      "contentType": "string (required, image/jpeg | image/png | image/webp)",
      "sizeBytes": "number (required, max 52428800)"
    }
  ],
  "visibility": "string (optional, default 'private')",
  "albumId": "uuid (optional)"
}
```

**Response 200**:
```json
{
  "uploads": [
    {
      "photoId": "uuid",
      "presignedUrl": "string (PUT URL, 5 min expiry)",
      "objectKey": "string (server-generated UUID key)",
      "expiresIn": 300
    }
  ]
}
```

**Errors**: 400 (invalid content type), 413 (file too large), 507 (storage full)

---

### POST /photos/confirm-upload — AUTH REQUIRED
Confirm that file was uploaded to S3 and trigger processing.

**Request**:
```json
{
  "photoId": "uuid (required)",
  "etag": "string (required, from S3 PUT response)"
}
```

**Response 202** (accepted for processing):
```json
{
  "photoId": "uuid",
  "processingStatus": "pending"
}
```

**Errors**: 400 (photo not found or not pending), 404 (object not in S3)

---

### GET /photos — AUTH REQUIRED
List own photos with filtering.

**Query params**: `page`, `limit`, `visibility`, `albumId`, `tags`, `search`, `sortBy`, `order`

**Response 200**:
```json
{
  "data": [{
    "id": "uuid",
    "title": "string | null",
    "thumbnailUrl": "string (presigned, 15 min)",
    "visibility": "string",
    "fileSizeBytes": "number",
    "processingStatus": "string",
    "createdAt": "ISO 8601"
  }],
  "meta": { "total": "number", "page": "number", "limit": "number", "totalPages": "number" }
}
```

---

### GET /photos/:id — AUTH REQUIRED (owner) or PUBLIC (if visibility allows)
Get photo detail with viewing URL.

**Response 200**:
```json
{
  "id": "uuid",
  "title": "string | null",
  "description": "string | null",
  "tags": ["string"],
  "visibility": "string",
  "viewUrl": "string (presigned URL, optimized version, 15 min)",
  "thumbnailUrl": "string (presigned, 15 min)",
  "fileSizeBytes": "number",
  "width": "number",
  "height": "number",
  "isFeatured": "boolean",
  "photographer": { "id": "uuid", "displayName": "string", "username": "string" },
  "likesCount": "number",
  "commentsCount": "number",
  "createdAt": "ISO 8601"
}
```

---

### PATCH /photos/:id — AUTH REQUIRED (owner only)
Update photo metadata.

**Request** (all optional):
```json
{
  "title": "string (max 100)",
  "description": "string (max 1000)",
  "tags": ["string (max 20 tags, max 50 chars each)"],
  "visibility": "public | private | link_only",
  "isFeatured": "boolean"
}
```

**Response 200**: Updated photo object

---

### DELETE /photos/:id — AUTH REQUIRED (owner only)
Delete photo permanently.

**Response 200**:
```json
{ "message": "Photo deleted", "freedBytes": "number" }
```

---

### GET /photos/shared/:shareToken — PUBLIC
Access a link-only photo.

**Response 200**: Same as GET /photos/:id (but without owner-specific fields)
**Errors**: 404 (invalid token)

---

## Album Endpoints (proxied to photos-service)

### POST /albums — AUTH REQUIRED
### GET /albums — AUTH REQUIRED (own albums)
### GET /albums/:id — AUTH REQUIRED or PUBLIC
### PATCH /albums/:id — AUTH REQUIRED (owner)
### DELETE /albums/:id — AUTH REQUIRED (owner)
### POST /albums/:id/photos — AUTH REQUIRED (add photos)
### DELETE /albums/:id/photos/:photoId — AUTH REQUIRED (remove photo)

*(Contracts follow same pattern as photos, simplified for brevity)*

---

## Portfolio Endpoints — PUBLIC

### GET /photographer/:username
Get public portfolio.

**Response 200**:
```json
{
  "photographer": {
    "displayName": "string",
    "username": "string",
    "bio": "string | null",
    "profilePictureUrl": "string | null",
    "socialLinks": [{ "platform": "string", "url": "string" }],
    "followersCount": "number",
    "photosCount": "number"
  },
  "featuredPhotos": [{ "id": "uuid", "thumbnailUrl": "string", "title": "string | null" }],
  "albums": [{ "id": "uuid", "name": "string", "coverUrl": "string | null", "photosCount": "number" }]
}
```

---

## Storage Endpoints (proxied to billing-service)

### GET /storage — AUTH REQUIRED
Get storage usage.

**Response 200**:
```json
{
  "plan": { "name": "string", "storageLimitBytes": "number" },
  "storageUsedBytes": "number",
  "usagePercentage": "number",
  "breakdown": [
    { "albumId": "uuid | null", "albumName": "string | 'Unorganized'", "sizeBytes": "number", "photosCount": "number" }
  ],
  "warnings": ["approaching_limit" | "critical_limit" | null]
}
```

---

## Social Endpoints (proxied to social-service)

### POST /photos/:id/like — AUTH REQUIRED
### DELETE /photos/:id/like — AUTH REQUIRED
### GET /photos/:id/comments — PUBLIC
### POST /photos/:id/comments — AUTH REQUIRED
### PATCH /comments/:id — AUTH REQUIRED (author)
### DELETE /comments/:id — AUTH REQUIRED (author or photo owner)
### POST /photographers/:id/follow — AUTH REQUIRED
### DELETE /photographers/:id/follow — AUTH REQUIRED
### GET /feed — AUTH REQUIRED

*(Contracts follow standard REST patterns)*

---

## Common Response Patterns

### Error Response
```json
{
  "statusCode": "number",
  "message": "string | string[]",
  "error": "string"
}
```

### Pagination
All list endpoints support: `?page=1&limit=20&sortBy=createdAt&order=desc`

### Rate Limiting Headers
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1709251200
```
