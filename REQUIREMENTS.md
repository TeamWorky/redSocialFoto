# Requisitos — PhotoVault

## RF: Requisitos Funcionales

### RF-01: Autenticación y Autorización
- RF-01.1: Registro con email y contraseña.
- RF-01.2: Login con OAuth social (Google, Apple, Facebook).
- RF-01.3: Recuperación de contraseña por email.
- RF-01.4: Roles de usuario: Fotógrafo, Visitante/Cliente, Administrador.
- RF-01.5: Tokens JWT (access + refresh).

### RF-02: Gestión de Fotos
- RF-02.1: Subida de fotos con almacenamiento en AWS S3.
- RF-02.2: Soporte de múltiples formatos (JPEG, PNG, WebP, RAW).
- RF-02.3: Generación automática de thumbnails y versiones optimizadas.
- RF-02.4: Metadatos EXIF opcionales (el fotógrafo decide si se muestran).
- RF-02.5: Eliminación de fotos con liberación de espacio en S3.

### RF-03: Sistema de Álbumes
- RF-03.1: Crear, editar y eliminar álbumes.
- RF-03.2: Agrupar fotos en álbumes.
- RF-03.3: Aplicar restricciones a nivel de álbum (heredadas por las fotos).
- RF-03.4: Álbumes públicos y privados.

### RF-04: Motor de Restricciones
- RF-04.1: **Anti-descarga** — Deshabilitar clic derecho, arrastrar, y servir imágenes de forma que no sean fácilmente descargables.
- RF-04.2: **Anti-captura de pantalla** — Técnicas de protección visual (overlay, CSS, watermark dinámico en canvas) para disuadir capturas.
- RF-04.3: **Visibilidad** — Público, privado, solo grupos/círculos seleccionados, solo por link.
- RF-04.4: **Temporal** — Fecha de expiración tras la cual la foto deja de ser visible.
- RF-04.5: **Geográfica** — Solo visible desde ubicaciones/países permitidos (basado en IP o geolocalización del navegador).
- RF-04.6: **Marca de agua** — Marca de agua dinámica superpuesta al visualizar (nombre del fotógrafo, texto personalizado).
- RF-04.7: **Contraseña** — Proteger fotos o álbumes con contraseña de acceso.
- RF-04.8: Las restricciones deben ser **combinables** (aplicar varias a la vez).

### RF-05: Portafolio del Fotógrafo
- RF-05.1: Página de perfil pública que funciona como portafolio.
- RF-05.2: Personalización básica (bio, foto de perfil, redes sociales).
- RF-05.3: Organización de álbumes destacados en el portafolio.
- RF-05.4: URL pública accesible (slug personalizado o username).

### RF-06: Interacción Social
- RF-06.1: Likes en fotos.
- RF-06.2: Comentarios en fotos.
- RF-06.3: Seguir a fotógrafos.
- RF-06.4: Feed de fotos públicas de fotógrafos seguidos.

### RF-07: Compartición
- RF-07.1: Compartir por **grupos/círculos** (crear grupos de contactos y compartir con ellos).
- RF-07.2: Compartir por **link directo** (con restricciones opcionales aplicadas al link).
- RF-07.3: Compartir en redes sociales (generar preview sin revelar contenido protegido).

### RF-08: Búsqueda
- RF-08.1: Buscar fotógrafos por nombre/username.
- RF-08.2: Buscar fotos públicas por tags/categorías.
- RF-08.3: Filtros por categoría, popularidad, fecha.

### RF-09: Sistema de Cobro por Almacenamiento
- RF-09.1: Cálculo del espacio utilizado por fotógrafo (peso total en S3).
- RF-09.2: Tiers/planes de almacenamiento (ej: gratuito hasta X GB, planes de pago).
- RF-09.3: Dashboard de uso de almacenamiento para el fotógrafo.
- RF-09.4: Notificaciones al acercarse al límite del plan.
- RF-09.5: Pasarela de pago para upgrades de plan.

### RF-10: Administración
- RF-10.1: Panel de administración para gestión de usuarios.
- RF-10.2: Gestión de planes de almacenamiento.
- RF-10.3: Reportes de uso y métricas.
- RF-10.4: Moderación de contenido reportado.

---

## RNF: Requisitos No Funcionales

### RNF-01: Rendimiento
- Las imágenes deben cargar en menos de 3 segundos (versión optimizada).
- El feed debe paginar con scroll infinito o paginación eficiente.
- Las thumbnails se sirven desde CDN.

### RNF-02: Seguridad
- Las imágenes protegidas se sirven mediante URLs firmadas con expiración (S3 presigned URLs).
- Las restricciones se validan del lado del servidor (nunca solo en el cliente).
- Protección contra hotlinking.
- Rate limiting en APIs.

### RNF-03: Escalabilidad
- Almacenamiento en S3 con estructura organizada por usuario.
- Procesamiento de imágenes asíncrono (thumbnails, marcas de agua).
- Base de datos optimizada para consultas de feed y búsqueda.

### RNF-04: Usabilidad
- Diseño responsive (mobile-first).
- Interfaz intuitiva para configurar restricciones.
- Preview en tiempo real de cómo se verá la foto con las restricciones aplicadas.

### RNF-05: Disponibilidad
- Uptime objetivo del 99.5%.
- Backups automáticos de la base de datos.
- Monitoreo y alertas.
