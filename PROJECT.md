# PhotoVault — Plataforma de Fotos con Restricciones para Fotógrafos

## Visión

Plataforma donde fotógrafos profesionales comparten su trabajo con controles avanzados de protección: previsualización sin descarga, protección contra capturas de pantalla, compartición pública o restringida, y un portafolio profesional integrado. El modelo de negocio se basa en cobro por almacenamiento (peso de fotos en S3).

## Problema

Los fotógrafos necesitan mostrar su trabajo a clientes potenciales sin arriesgar el uso no autorizado de sus imágenes. Las plataformas actuales no ofrecen protección real contra descarga ni capturas de pantalla, y las soluciones de portafolio profesional no integran compartición social con restricciones granulares.

## Solución

Una plataforma que combina:
- **Portafolio profesional** para fotógrafos
- **Previsualización protegida** (sin descarga, protección anti-captura de pantalla)
- **Compartición flexible** (pública, por grupos/círculos, por link directo)
- **Restricciones combinables** por foto/álbum (temporal, geográfica, contraseña, marca de agua)
- **Modelo de cobro** basado en el peso de almacenamiento en S3

## Tipo de Producto

Producto real orientado al mercado.

## Usuarios Objetivo

| Rol | Descripción |
|-----|-------------|
| **Fotógrafo** | Usuario principal. Sube fotos, gestiona portafolio, configura restricciones, paga por almacenamiento. |
| **Visitante/Cliente** | Visualiza fotos compartidas, interactúa con likes/comentarios, no puede descargar contenido protegido. |
| **Administrador** | Gestiona la plataforma, planes de almacenamiento, usuarios reportados. |

## Modelo de Negocio

- Cobro basado en el **peso total de fotos almacenadas** en S3.
- Posibles tiers: plan gratuito con límite de almacenamiento, planes de pago con mayor capacidad.

## Modelo Social

- **Grupos y Círculos**: Compartir contenido con audiencias específicas.
- **Links directos**: Compartir fotos o álbumes mediante enlaces (con o sin restricciones).
- **Portafolio público**: Página de perfil del fotógrafo como portafolio.

## Alcance del MVP

Alcance **moderado**:
- Autenticación (email/password + OAuth social)
- Subida y gestión de fotos con almacenamiento en S3
- Sistema de álbumes
- Motor de restricciones (visibilidad, anti-descarga, anti-captura, temporal, geográfica, marca de agua, contraseña)
- Portafolio público por fotógrafo
- Feed de contenido público
- Perfiles de usuario
- Comentarios y likes
- Búsqueda de fotógrafos y contenido
- Sistema de cobro por almacenamiento

## Estado

- **Fase**: Definición y diseño
- **Repositorio**: `redSocialFoto`
- **Stack tecnológico**: Por definir
- **Trello**: [Tarjeta del proyecto](https://trello.com/c/S7UP7Wc5)
