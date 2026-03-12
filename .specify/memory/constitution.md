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

## Git Flow — NON-NEGOTIABLE

### Branches Permanentes

| Branch | Propósito | Protección |
|--------|-----------|------------|
| `main` | Producción. Código estable y desplegado. | PR + 1 aprobación obligatoria. No push directo. No force push. |
| `development` | Integración. Branch por defecto en GitHub. | PR + 1 aprobación obligatoria. No push directo. No force push. |

### Branches Temporales

| Tipo | Naming Convention | Se crea desde | Se mergea a | Ejemplo |
|------|-------------------|---------------|-------------|---------|
| Feature | `feature/F-<id>-<descripcion>` | `development` | `development` | `feature/F-04-anti-descarga` |
| Bugfix | `bugfix/BUG-<id>-<descripcion>` | `development` | `development` | `bugfix/BUG-12-fix-upload` |
| Hotfix | `hotfix/HOT-<id>-<descripcion>` | `main` | `main` + `development` | `hotfix/HOT-01-fix-auth-crash` |
| Release | `release/<version>` | `development` | `main` + `development` | `release/1.0.0` |

### Flujo de Trabajo

```
feature/* ──PR──► development
bugfix/*  ──PR──► development
release/* ──PR──► main
   └─────PR──► development
hotfix/*  ──PR──► main
   └─────PR──► development
```

### Buenas Prácticas para Pull Requests

#### Antes de crear el PR
1. Asegurar que la branch está actualizada con su rama base (`git pull origin development`).
2. Resolver conflictos localmente antes de abrir el PR.
3. Todos los tests deben pasar localmente (`unit` + `integration`).
4. Ejecutar linter y formateo.
5. Revisar el diff propio antes de abrir el PR (auto-review).

#### Estructura del PR

```markdown
## Título
<tipo>(<alcance>): <ID> - <descripción concisa>

Tipos: feat, fix, docs, style, refactor, test, chore, hotfix, security
Ejemplo: feat(photos): F-04 - add anti-download protection for image viewer
```

```markdown
## Cuerpo del PR (template obligatorio)

### Descripción
Resumen claro de qué cambia y por qué.

### Spec relacionada
Link a la spec en `.specify/` que respalda este cambio.

### Tipo de cambio
- [ ] Feature nueva (cambio no-breaking que agrega funcionalidad)
- [ ] Bug fix (cambio no-breaking que corrige un issue)
- [ ] Breaking change (cambio que alteraría funcionalidad existente)
- [ ] Hotfix (corrección urgente en producción)
- [ ] Refactor (cambio de código sin alterar comportamiento)
- [ ] Docs (solo documentación)

### Checklist de calidad
- [ ] Tests escritos y pasando (TDD: tests primero)
- [ ] Cobertura no disminuyó
- [ ] Sin warnings de linter
- [ ] Revisión OWASP completada (para features con input de usuario, auth, o datos sensibles)
- [ ] Datos sensibles clasificados según la tabla de Data Classification
- [ ] Documentación actualizada si aplica

### Checklist de seguridad (si aplica)
- [ ] Input validado y sanitizado
- [ ] Queries parametrizadas (no concatenación)
- [ ] Permisos verificados en endpoints
- [ ] No hay secrets hardcodeados
- [ ] URLs firmadas con expiración para recursos S3

### Screenshots / Evidencia
(Si aplica, capturas de la funcionalidad o output de tests)
```

#### Durante la revisión
- El reviewer verifica cumplimiento con la constitución (SDD, TDD, OWASP, ISO 27001).
- Comentarios constructivos y específicos (referenciar línea de código).
- Bloquear merge si hay vulnerabilidades de seguridad o tests faltantes.
- Resolver TODOS los threads de conversación antes de aprobar.

#### Después del merge
- Eliminar la branch temporal (automático en GitHub si está configurado).
- Verificar que el CI/CD pasa en la rama destino.
- Actualizar la tarjeta de Trello correspondiente.

### Commits Convencionales

Formato obligatorio para mensajes de commit:

```
<tipo>(<alcance>): <ID> - <descripción>

[cuerpo opcional]

[footer opcional: Closes #42, Refs T001]
```

| Tipo | Uso |
|------|-----|
| `feat` | Nueva funcionalidad |
| `fix` | Corrección de bug |
| `docs` | Solo documentación |
| `style` | Formateo, punto y coma faltante (no cambia lógica) |
| `refactor` | Refactorización sin cambiar comportamiento |
| `test` | Agregar o corregir tests |
| `chore` | Mantenimiento, dependencias, configuración |
| `hotfix` | Corrección urgente de producción |
| `security` | Corrección de vulnerabilidad de seguridad |

Ejemplo:
```
feat(restrictions): RF-04 - add password protection for albums

Albums can now be protected with a password. Visitors must
enter the correct password before viewing album contents.

Spec: .specify/specs/RF-04-restrictions.md
OWASP: A02 - password hashed with bcrypt before storage
Closes #42
```

## Governance

- Esta constitución es el documento rector del proyecto y **supersede cualquier otra práctica**.
- Toda modificación requiere: documentación del cambio, justificación, y actualización de version.
- Todo PR/review debe verificar cumplimiento con esta constitución.
- Las excepciones a los principios NON-NEGOTIABLE requieren documentación explícita y aprobación.

**Version**: 1.1.0 | **Ratified**: 2026-03-12 | **Last Amended**: 2026-03-12
