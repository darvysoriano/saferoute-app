# Logs de generación — SafeRoute

Registro de cada ejecución de un asistente de IA sobre el proyecto: qué archivo generó o modificó, cuánto tardó y qué hay que tener en cuenta. Complementa [prompts-history.md](prompts-history.md), que guarda el texto de cada prompt; el número de ejecución coincide con el número de prompt.

## Formato de registro

| Campo | Contenido |
|---|---|
| # | Número de ejecución, igual al del prompt en `prompts-history.md` |
| Fecha | Fecha y hora de fin de la ejecución (AAAA-MM-DD HH:MM, hora local UTC−06:00) |
| Archivo generado | Archivo o directorio creado (**C**) o modificado (**M**) |
| Tiempo de ejecución | Duración desde el inicio hasta el fin de la ejecución (mm:ss) |
| Observaciones | Resultado, decisiones relevantes, pendientes o incidencias |

Cómo se mide el tiempo: se toma la hora del sistema al empezar y al terminar cada ejecución. Las ejecuciones 1 a 5 son anteriores a este registro y no se midieron; su fecha y hora salen de las marcas de tiempo de los archivos cuando todavía se conservan.

## Registro de ejecuciones

| # | Fecha | Archivo generado | Tiempo de ejecución | Observaciones |
|---|---|---|---|---|
| 1 | 2026-10-20 12:14 | **C** `docs/`, `docs/prd/`, `docs/sdd/`, `docs/ai/` | No medido | Solo estructura, sin contenido. Las carpetas vacías no las registra git. |
| 2 | 2026-10-20 (hora no conservada) | **C** `docs/prd/PRD-SafeRoute.md` | No medido | PRD v0.1. El contexto del producto se tomó solo del README; los valores de negocio (radios, tiempos, metas) son propuestas a validar. |
| 3 | 2026-10-20 (hora no conservada) | **C** `docs/sdd/SDD-SafeRoute.md` | No medido | SDD v0.1 con React Native, .NET y Azure, alineado con los estándares de Ryndem. Introdujo la verificación cada 20 minutos, que no estaba en el PRD. Reemplazado por completo en la ejecución 4. |
| 4 | 2026-10-20 13:00 | **M** `docs/prd/PRD-SafeRoute.md`, **M** `docs/sdd/SDD-SafeRoute.md` (reescrito) | No medido | PRD y SDD a v0.2: Java, componentes sin licencia y ejecución local. Impacto en el alcance: iOS fuera del MVP; se retira el inicio de sesión con Apple. Se agregan RF-044, §8.8 de licenciamiento y riesgos R-13 a R-15. |
| 5 | 2026-10-20 13:31 | **C** `docs/ai/prompts-history.md` | No medido | Historial de los prompts 1 a 5. La actualización por cada prompt depende del asistente; está pendiente decidir si se automatiza con un hook. |
| 6 | 2026-10-20 13:34 | **C** `docs/ai/generation-logs.md`, **M** `docs/ai/prompts-history.md` | ≈ 00:30 | Creación de este registro con las ejecuciones anteriores. Primera ejecución con tiempo medido; el inicio se tomó en el primer comando de la ejecución, así que no incluye el tiempo previo de análisis. |
| 7 | 2026-10-22 16:59 | **C** `docs/sdd/ETS-SafeRoute.md`, **M** `docs/sdd/SDD-SafeRoute.md`, **M** `docs/prd/PRD-SafeRoute.md` | 17:01 | ETS v1.0 (≈ 3 200 líneas). El pedido contradecía el stack validado (Node.js, MongoDB Atlas M0 con licencia SSPL, Foreground Service permanente); por decisión del usuario se alineó a Java/Spring Boot/PostgreSQL y se agregó WebSocket solo para la app del tutor. PRD y SDD a v0.3: RF-045, RF-046, RNF-PER-06, RNF-FIA-07, CA-13, ADR-017 a ADR-019. Se corrigió un vector de prueba de Haversine mal calculado antes de cerrar. Incluye el tiempo de espera de la respuesta del usuario. |
