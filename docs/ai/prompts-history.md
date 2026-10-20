# Historial de prompts — SafeRoute

Registro de los prompts usados con asistentes de IA para generar y modificar el proyecto. Se agrega una entrada por cada prompt ejecutado, en orden cronológico.

| Campo | Contenido |
|---|---|
| Fecha | Día de ejecución (AAAA-MM-DD) |
| Descripción | Qué se pidió y qué produjo |
| Archivos afectados | Archivos creados o modificados |
| Prompt | Texto exacto, sin corregir |

## Índice

| # | Fecha | Descripción | Archivos afectados |
|---|---|---|---|
| 1 | 2026-10-20 | Estructura de carpetas de documentación | `docs/`, `docs/prd/`, `docs/sdd/`, `docs/ai/` |
| 2 | 2026-10-20 | Generación del PRD | `docs/prd/PRD-SafeRoute.md` |
| 3 | 2026-10-20 | Generación del SDD | `docs/sdd/SDD-SafeRoute.md` |
| 4 | 2026-10-20 | Cambio de arquitectura a Java y componentes sin licencia | `docs/sdd/SDD-SafeRoute.md`, `docs/prd/PRD-SafeRoute.md` |
| 5 | 2026-10-20 | Creación de este historial de prompts | `docs/ai/prompts-history.md` |
| 6 | 2026-10-20 | Creación del registro de logs de generación | `docs/ai/generation-logs.md` |

---

## 1. Estructura de carpetas de documentación

- **Fecha:** 2026-10-20
- **Descripción:** Creación de los directorios de documentación del proyecto, sin contenido.
- **Archivos afectados:** `docs/`, `docs/prd/`, `docs/sdd/`, `docs/ai/`

```text
Crea dentro del proyecto las carpetas /docs, /docs/prd, /docs/sdd y /docs/ai. No generes contenido aún, solo la estructura de directorios.
```

## 2. Generación del PRD

- **Fecha:** 2026-10-20
- **Descripción:** Product Requirements Document v0.1: visión, objetivos, alcance, funcionalidades, casos de uso, requerimientos funcionales y no funcionales, criterios de aceptación, métricas y riesgos.
- **Archivos afectados:** `docs/prd/PRD-SafeRoute.md`

```text
Genera el archivo /docs/prd/PRD-SafeRoute.md con un PRD completo que incluya: visión del producto, objetivos, alcance, funcionalidades detalladas, casos de uso, requerimientos funcionales, requerimientos no funcionales, criterios de aceptación, métricas de éxito y riesgos del proyecto. Usa lenguaje técnico claro y estructura profesional.
```

## 3. Generación del SDD

- **Fecha:** 2026-10-20
- **Descripción:** Software Design Document v0.1: arquitectura, componentes, backend, app móvil, modelo de datos, APIs, flujo de validación de llegada, verificación cada 20 minutos, notificaciones, decisiones técnicas, dependencias y estándares (stack inicial React Native, .NET y Azure).
- **Archivos afectados:** `docs/sdd/SDD-SafeRoute.md`

```text
Genera el archivo /docs/sdd/SDD-SafeRoute.md con un SDD completo que incluya: arquitectura general del sistema, diagramas de componentes, diseño del backend, diseño del frontend móvil, modelo de datos, APIs y endpoints, flujo de validación de llegada, flujo de verificación cada 20 minutos, manejo de notificaciones, decisiones técnicas, dependencias y estándares de desarrollo. Usa formato técnico profesional.
```

## 4. Cambio de arquitectura a Java y componentes sin licencia

- **Fecha:** 2026-10-20
- **Descripción:** Rediseño del SDD a v0.2 con Android nativo en Java, Spring Boot, PostgreSQL, db-scheduler, FCM y OpenStreetMap autoalojado; se agregan el análisis de licenciamiento y la guía de ejecución local. El PRD pasa a v0.2: MVP solo Android, sin inicio de sesión con Apple, RF-044 y requerimientos de licenciamiento y ejecución local.
- **Archivos afectados:** `docs/sdd/SDD-SafeRoute.md`, `docs/prd/PRD-SafeRoute.md`

```text
La arquitectura general debe ser con Java y servicios gratuitos sin licencia para poder llevar acabo su ejecución tanto en frontend movil, backend, base de datos, notificaciones, mapas, realiza estas modificaciones en la documentación y el analisis para realizar la ejecución sin problemas de licenciamiento al momento de ser ejecutados y poder llevar a pruebas en un ambiente local.  continua usando un formato técnico profesional
```

## 5. Creación de este historial de prompts

- **Fecha:** 2026-10-20
- **Descripción:** Creación de este archivo con los prompts anteriores. Se establece que se actualiza con cada prompt nuevo.
- **Archivos afectados:** `docs/ai/prompts-history.md`

```text
Crea el archivo /docs/ai/prompts-history.md y registra los prompts utilizados hasta ahora, incluyendo fecha, descripción y el prompt exacto. Este archivo debe actualizarse cada vez que se ejecute un nuevo prompt.
```

## 6. Creación del registro de logs de generación

- **Fecha:** 2026-10-20
- **Descripción:** Creación de `generation-logs.md` con el formato de registro (fecha, archivo generado, tiempo de ejecución, observaciones) y las ejecuciones 1 a 6.
- **Archivos afectados:** `docs/ai/generation-logs.md`, `docs/ai/prompts-history.md`

```text
Crea el archivo /docs/ai/generation-logs.md y agrega una sección para registrar logs de cada ejecución, incluyendo: fecha, archivo generado, tiempo de ejecución y observaciones.
```
