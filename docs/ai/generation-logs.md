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
| 8 | 2026-11-06 16:25 | **C** `docs/analysis/TRAZABILIDAD.md` | 06:00 | Matriz de trazabilidad de 51 RF en cinco dimensiones. Cobertura: 38 RF con caso de uso (9 parciales), 17 RF con prueba específica, 17 solo con criterio de aceptación y 17 sin prueba, de los cuales 12 son Must y bloquean la liberación según la ETS §5.6.8. 12 hallazgos (H-01 a H-12), entre ellos tokens de verificación sin modelar, borrado de respaldos sin mecanismo y retención del historial inconsistente entre PRD y SDD. Se corrigieron dos estados de prueba y un total antes de cerrar; los conteos se verificaron por script. No se modificaron PRD, SDD ni ETS. |
| 9 | 2026-11-09 17:52 | **M** `docs/sdd/ETS-SafeRoute.md` (v1.1), **M** `docs/sdd/SDD-SafeRoute.md` (v0.4), **M** `docs/prd/PRD-SafeRoute.md` (v0.4), **M** `docs/analysis/TRAZABILIDAD.md` (v1.1, reescrito) | 09:00 | 12 hallazgos cerrados. ETS: 54 casos de prueba nuevos (de 40 a 94), plan de QA renumerado a §5.14, `account_token`, `safety_event_daily_stat`, retiro de `heartbeat`, §3.6 de reportes de fallos, códigos `AUT-006` y `AUT-007`. SDD: tareas `data-erasure` y `data-export`, caducidad de 30 s de la consulta bajo demanda, retención de respaldos de 21 días, retención de eventos a 30 días alineada con el PRD F-10, `RealtimeService`, `LocationHistoryService`, `GeoMath`, `GeofenceSyncWorker`, `RealtimeClient` y dos pantallas. PRD: CU-10 a CU-19. Resultado: 51/51 RF con caso de uso (7 parciales) y con casos de prueba especificados (no implementados). La retención de 30 días y el borrado físico siguen pendientes de validación legal. Se corrigió el conteo de casos de la matriz antes de cerrar; referencias y conteos verificados por script. |
