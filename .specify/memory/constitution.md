# PhotoVault Constitution

## Core Principles

### I. Spec-Driven Development (SDD) — NON-NEGOTIABLE
Toda funcionalidad comienza con una especificación aprobada antes de escribir código.
- Las specs definen el **qué** y el **por qué** antes del **cómo**.
- Flujo obligatorio: Spec → Plan → Tasks → Tests → Implementación.
- Ningún código de producción se escribe sin una spec aprobada que lo respalde.
- Las specs son documentos vivos que se actualizan con los cambios de requisitos.

### II. Test-Driven Development (TDD) — NON-NEGOTIABLE
El ciclo Red-Green-Refactor se aplica estrictamente en todo el proyecto.
- **Red**: Escribir tests que fallen basados en los acceptance scenarios de la spec.
- **Green**: Implementar el código mínimo para que los tests pasen.
- **Refactor**: Mejorar el código manteniendo todos los tests en verde.
- Cobertura mínima obligatoria: **80% de líneas**, **70% de branches**.
- Los tests se escriben ANTES del código de producción, sin excepciones.
- Tipos de tests requeridos:
  - **Unit tests**: Para toda lógica de negocio y servicios.
  - **Integration tests**: Para endpoints API, acceso a datos, servicios externos (S3, OAuth).
  - **E2E tests**: Para flujos críticos de usuario (auth, subida de fotos, restricciones).

### III. Security-First (OWASP Top 10) — NON-NEGOTIABLE
Cada línea de código se evalúa contra las vulnerabilidades del OWASP Top 10 (2021).
- **A01 - Broken Access Control**: Validar permisos en CADA endpoint. Los fotógrafos solo acceden a SUS recursos. Visitantes solo ven contenido autorizado.
- **A02 - Cryptographic Failures**: Contraseñas con bcrypt/argon2 (nunca plaintext). HTTPS obligatorio. Tokens JWT firmados con algoritmos seguros (RS256 o EdDSA). Datos sensibles cifrados en reposo.
- **A03 - Injection**: Queries parametrizadas obligatorias (ORM). Sanitización de todo input del usuario. Validación estricta de nombres de archivo en uploads.
- **A04 - Insecure Design**: Threat modeling antes de implementar features críticas. Principio de mínimo privilegio en roles y permisos.
- **A05 - Security Misconfiguration**: Headers de seguridad (CSP, HSTS, X-Frame-Options). Deshabilitar features innecesarias. Configuración segura de CORS.
- **A06 - Vulnerable Components**: Auditoría de dependencias en cada build. No se permiten dependencias con CVEs conocidos de severidad alta/crítica.
- **A07 - Authentication Failures**: Rate limiting en login (máx 5 intentos/minuto). Tokens con expiración corta (access: 15min, refresh: 7d). Logout invalida tokens en el servidor.
- **A08 - Data Integrity Failures**: Validar integridad de archivos subidos (tipo MIME, magic bytes). Firmar URLs de S3 con expiración. No deserializar datos no confiables.
- **A09 - Security Logging & Monitoring**: Loggear todos los eventos de autenticación, acceso a recursos protegidos y acciones administrativas. Alertas ante patrones anómalos.
- **A10 - Server-Side Request Forgery (SSRF)**: Validar y sanitizar todas las URLs externas. Restringir acceso a metadata de cloud desde la aplicación.

### IV. ISO 27001 Compliance
El desarrollo cumple con los controles relevantes de ISO 27001:2022 (Anexo A).
- **A.5 - Políticas de seguridad**: Este documento constituye la política de seguridad del desarrollo.
- **A.8 - Gestión de activos**: Clasificación de datos (fotos originales = CONFIDENCIAL, thumbnails públicos = PÚBLICO, credenciales = RESTRINGIDO).
- **A.8.10 - Eliminación de información**: Los archivos eliminados se purgan de S3 y de los backups dentro del período de retención.
- **A.8.24 - Uso de criptografía**: TLS 1.2+ para tránsito, AES-256 para reposo, bcrypt/argon2 para contraseñas.
- **A.8.25 - Ciclo de vida de desarrollo seguro**: SDD + TDD + revisión de seguridad en cada PR.
- **A.8.26 - Requisitos de seguridad de aplicaciones**: Validación de input, output encoding, gestión segura de sesiones.
- **A.8.28 - Codificación segura**: Revisión OWASP en cada PR. Análisis estático de seguridad (SAST) en CI/CD.
- **A.5.23 - Seguridad en servicios cloud**: Configuración segura de S3 (buckets privados, políticas de acceso, cifrado).
- **A.8.15 - Logging**: Logs de auditoría inmutables para eventos de seguridad.
- **A.8.16 - Monitorización**: Monitoreo de accesos, uso de almacenamiento, intentos de acceso no autorizado.

### V. Simplicidad y Pragmatismo
- YAGNI: No implementar funcionalidades que no estén en la spec actual.
- DRY: Evitar duplicación, pero no crear abstracciones prematuras.
- KISS: La solución más simple que cumpla los requisitos y pase los tests.
- Código autodocumentado sobre comentarios excesivos.

### VI. Protección de Contenido como Diferenciador
- Las restricciones de fotos se validan SIEMPRE en el servidor, nunca solo en el cliente.
- Las imágenes originales NUNCA se sirven directamente; se sirven versiones procesadas.
- Las URLs de acceso a imágenes son firmadas y con expiración corta.
- La marca de agua se aplica en el servidor, no en el cliente.

## Quality Gates

### Gate 1: Pre-Commit
- Linting y formateo automático.
- Tests unitarios relacionados con los archivos modificados.
- No secrets en el código (detección automática).

### Gate 2: Pull Request
- Todos los tests pasan (unit + integration).
- Cobertura no disminuye respecto a la rama principal.
- Revisión de seguridad OWASP (checklist automatizado).
- Al menos 1 aprobación de code review.

### Gate 3: Pre-Deploy
- Suite completa de tests (unit + integration + E2E).
- Auditoría de dependencias sin CVEs críticas.
- Análisis estático de seguridad (SAST) aprobado.
- Validación de cumplimiento ISO 27001 (controles relevantes).

## Data Classification

| Clasificación | Ejemplos | Tratamiento |
|---------------|----------|-------------|
| **CONFIDENCIAL** | Fotos originales, datos de pago, tokens | Cifrado en reposo y tránsito. Acceso con autenticación + autorización. Logs de acceso. |
| **RESTRINGIDO** | Emails, contraseñas hash, API keys | Cifrado en reposo. Acceso solo por servicios internos. Nunca en logs. |
| **INTERNO** | Perfiles de usuario, metadata de fotos, álbumes | Acceso autenticado. Validación de permisos. |
| **PÚBLICO** | Portafolios públicos, thumbnails públicos, perfiles públicos | Acceso sin autenticación. Cache en CDN. |

## Governance

- Esta constitución es el documento rector del proyecto y **supersede cualquier otra práctica**.
- Toda modificación requiere: documentación del cambio, justificación, y actualización de version.
- Todo PR/review debe verificar cumplimiento con esta constitución.
- Las excepciones a los principios NON-NEGOTIABLE requieren documentación explícita y aprobación.

**Version**: 1.0.0 | **Ratified**: 2026-03-12 | **Last Amended**: 2026-03-12
