# Features del MVP — PhotoVault

## Priorización por Fases

### Fase 1 — Core (Lanzamiento mínimo)

| ID | Feature | Descripción | Prioridad |
|----|---------|-------------|-----------|
| F-01 | Auth email/password | Registro, login, logout, recuperación de contraseña | Crítica |
| F-02 | Auth OAuth | Login con Google y Apple | Crítica |
| F-03 | Subida de fotos a S3 | Upload con generación de thumbnails y versiones optimizadas | Crítica |
| F-04 | Anti-descarga | Protección contra descarga directa de imágenes | Crítica |
| F-05 | Anti-captura | Protección visual contra capturas de pantalla | Crítica |
| F-06 | Visibilidad | Configurar foto como pública, privada o por link | Crítica |
| F-07 | Portafolio público | Perfil del fotógrafo como página de portafolio | Crítica |
| F-08 | Dashboard de almacenamiento | Visualización del espacio usado vs. plan contratado | Crítica |

### Fase 2 — Social + Álbumes

| ID | Feature | Descripción | Prioridad |
|----|---------|-------------|-----------|
| F-09 | Álbumes | Crear álbumes, agrupar fotos, restricciones a nivel álbum | Alta |
| F-10 | Likes | Dar like a fotos públicas | Alta |
| F-11 | Comentarios | Comentar en fotos públicas | Alta |
| F-12 | Seguir fotógrafos | Sistema de follow/unfollow | Alta |
| F-13 | Feed | Feed de fotos públicas de fotógrafos seguidos | Alta |
| F-14 | Búsqueda | Buscar fotógrafos y fotos por tags/categorías | Alta |

### Fase 3 — Restricciones Avanzadas

| ID | Feature | Descripción | Prioridad |
|----|---------|-------------|-----------|
| F-15 | Marca de agua dinámica | Overlay automático con nombre del fotógrafo o texto custom | Media |
| F-16 | Restricción temporal | Fecha de expiración en fotos/álbumes | Media |
| F-17 | Restricción geográfica | Limitar visualización por ubicación/país | Media |
| F-18 | Protección por contraseña | Contraseña para acceder a fotos/álbumes | Media |
| F-19 | Compartir por grupos/círculos | Crear grupos de contactos y compartir con ellos | Media |
| F-20 | Compartir por link con restricciones | Links directos con restricciones configurables | Media |

### Fase 4 — Monetización + Admin

| ID | Feature | Descripción | Prioridad |
|----|---------|-------------|-----------|
| F-21 | Planes de almacenamiento | Tiers gratuito/pago basados en GB almacenados | Media |
| F-22 | Pasarela de pago | Integración con Stripe/similar para upgrades | Media |
| F-23 | Notificaciones de almacenamiento | Alertas al acercarse al límite del plan | Baja |
| F-24 | Panel de administración | Gestión de usuarios, planes, contenido reportado | Baja |
| F-25 | Métricas y reportes | Dashboard admin con métricas de uso | Baja |

---

## Detalle de Protección Anti-descarga y Anti-captura

Estas son las features más críticas y diferenciadores del producto:

### Anti-descarga (F-04)
- Deshabilitar menú contextual (clic derecho) sobre imágenes.
- Deshabilitar arrastrar imágenes.
- Servir imágenes como background de `<div>` en vez de `<img>` (dificulta "Guardar imagen como").
- URLs firmadas de S3 con expiración corta (presigned URLs).
- Protección contra hotlinking (validación de referer/origin).

### Anti-captura (F-05)
- Overlay transparente sobre la imagen que impide captura limpia.
- Marca de agua dinámica renderizada en canvas (no en la imagen original).
- CSS `user-select: none` y `-webkit-touch-callout: none`.
- Detección de herramientas de captura de pantalla (best-effort, no es infalible).
- Degradación de calidad en la versión visualizada (la original queda en S3).

> **Nota**: Ninguna protección del lado del cliente es 100% infalible. El objetivo es hacer la descarga/captura lo suficientemente difícil para disuadir al usuario promedio, no para detener a un usuario técnico determinado. La protección real está en servir versiones de menor calidad y mantener los originales en S3 con acceso controlado.
