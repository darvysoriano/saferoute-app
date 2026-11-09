# Matriz de trazabilidad — SafeRoute

| Campo | Valor |
|---|---|
| Producto | SafeRoute (aplicación móvil) |
| Documento | Matriz de trazabilidad de requerimientos funcionales |
| Versión | 1.1 |
| Estado | Borrador para revisión |
| Fecha | 2026-11-09 |
| Responsable | Darvy |

### Historial de cambios

| Versión | Fecha | Cambio |
|---|---|---|
| 1.0 | 2026-11-06 | Versión inicial sobre PRD v0.3, SDD v0.3 y ETS v1.0. Doce hallazgos (H-01 a H-12). |
| 1.1 | 2026-11-09 | Actualización sobre PRD v0.4, SDD v0.4 y ETS v1.1, que corrigen los doce hallazgos. Todos los RF tienen caso de uso y casos de prueba especificados. |

### Documentos fuente

| Documento | Versión | Fecha | Aporta |
|---|---|---|---|
| [PRD-SafeRoute.md](../prd/PRD-SafeRoute.md) | 0.4 | 2026-11-09 | Requerimientos funcionales (RF), funcionalidades (F), casos de uso (CU), criterios de aceptación (CA) |
| [SDD-SafeRoute.md](../sdd/SDD-SafeRoute.md) | 0.4 | 2026-11-09 | Modelo de datos, endpoints, tareas del Worker, estructura de la app Android, niveles de prueba |
| [ETS-SafeRoute.md](../sdd/ETS-SafeRoute.md) | 1.1 | 2026-11-09 | Prioridad MoSCoW, contratos, protocolo WebSocket, DDL, matriz de casos de prueba (QA) |

---

## Índice

1. [Propósito y alcance](#1-propósito-y-alcance)
2. [Convenciones](#2-convenciones)
3. [Resumen de cobertura](#3-resumen-de-cobertura)
4. [RF → Caso de uso](#4-rf--caso-de-uso)
5. [RF → Entidades](#5-rf--entidades)
6. [RF → Endpoints](#6-rf--endpoints)
7. [RF → Android](#7-rf--android)
8. [RF → Tests](#8-rf--tests)
9. [Hallazgos y brechas](#9-hallazgos-y-brechas)
10. [Mantenimiento de la matriz](#10-mantenimiento-de-la-matriz)

---

## 1. Propósito y alcance

Esta matriz enlaza cada requerimiento funcional del PRD con los elementos que lo realizan y lo verifican: casos de uso, entidades de datos, endpoints, componentes de la app Android y pruebas. Sirve para:

- Comprobar que ningún requerimiento queda sin diseño ni sin prueba.
- Evaluar el impacto de un cambio: qué tablas, endpoints, pantallas y pruebas toca un RF.
- Respaldar la regla de liberación de la ETS §5.14.8: un requerimiento **Must** sin prueba etiquetada bloquea la liberación.

Cubre los **51 requerimientos funcionales** del PRD v0.4 (RF-001 a RF-063). Los requerimientos no funcionales quedan fuera; su trazabilidad al diseño está en el SDD §17.

Es un documento de análisis: no contiene código. Los nombres de clases, tablas y rutas son los que definen el SDD y la ETS.

**Alcance de "Cubierto".** El proyecto todavía no tiene código. Que un RF figure como cubierto significa que sus casos de prueba están **especificados** en la ETS o el SDD, no que se hayan implementado ni ejecutado. La verificación real llegará con el reporte de CI (§10).

---

## 2. Convenciones

### 2.1 Prioridad

Se usa la prioridad MoSCoW de la ETS §1.1, equivalente a la del PRD:

| MoSCoW | PRD | Significado |
|---|---|---|
| **M** (Must) | P0 | Imprescindible para el MVP |
| **S** (Should) | P1 | Deseable en el MVP |

Ningún RF del MVP es Could ni Won't; los elementos fuera de alcance (W-01 a W-05) no se trazan.

### 2.2 Tipo de relación con el caso de uso

| Marca | Significado |
|---|---|
| **Directa** | El caso de uso describe el comportamiento del RF en su flujo principal o alterno. |
| **Parcial** | El caso de uso lo implica (precondición, postcondición o paso relacionado) sin describirlo. |
| **Sin CU** | Ningún caso de uso lo cubre. |

### 2.3 Estado de prueba

| Estado | Criterio |
|---|---|
| **Cubierto** | Al menos un caso de la matriz de QA de la ETS (`QA-*`), un escenario local reproducible del SDD §15.4 (`ESC`) o una prueba unitaria nombrada específica del RF. |
| **Parcial** | Solo un criterio de aceptación del PRD (`CA-*`) o una prueba genérica sin caso concreto. |
| **Sin prueba** | Solo el criterio de aceptación de la tabla SRS de la ETS §1.2; ningún caso de prueba definido. |

### 2.4 Abreviaturas

| Abreviatura | Significado |
|---|---|
| `T` / `M` | Modo Tutor / Modo Menor de la app Android |
| `WS` | Mensaje del canal WebSocket `/ws/v1` (ETS §3.5) |
| `FCM` | Mensaje de datos enviado por Firebase Cloud Messaging |
| Tarea | Tarea de db-scheduler ejecutada por `saferoute-worker` (SDD §4.7) |
| `ESC` | Escenario local reproducible (SDD §15.4) |
| Nivel U / I / E / C | Prueba unitaria / de integración / en emulador / de campo (ETS §5.1) |

Todas las rutas REST tienen el prefijo `/api/v1`, salvo `/ws/v1`.

---

## 3. Resumen de cobertura

### 3.1 Por dimensión

| Dimensión | Con trazabilidad completa | Parcial | Sin trazabilidad | Observación |
|---|---|---|---|---|
| RF → Caso de uso | 44 (86 %) | 7 (14 %) | 0 | Los parciales son RF que un caso de uso implica sin describir paso a paso (política de contraseña, perfiles de menores, transparencia, desvinculación, zona horaria, idempotencia y contenido de la notificación). |
| RF → Entidades | 50 (98 %) | — | 1 (2 %) | RF-018 es solo de presentación; no requiere entidad (por diseño). |
| RF → Endpoints | 48 (94 %) | — | 3 (6 %) | Por diseño: RF-018 es solo de la app; RF-028 y RF-034 se realizan con tareas del Worker. |
| RF → Android | 51 (100 %) | — | 0 | — |
| RF → Tests | 51 (100 %) | 0 | 0 | Todos los RF tienen casos de prueba especificados; ningún RF **Must** queda sin prueba. |

### 3.2 Evolución respecto de la versión 1.0

| Dimensión | v1.0 | v1.1 |
|---|---|---|
| RF con caso de uso (directo o parcial) | 38 | 51 |
| RF con caso de uso directo | 29 | 44 |
| RF con prueba especificada (Cubierto) | 17 | 51 |
| RF Must sin prueba | 12 | 0 |
| Casos de prueba en la ETS | 40 | 94 |
| Hallazgos abiertos | 12 | 0 |

### 3.3 Por área funcional

| Área | RF | Con CU | Con prueba (Cubierto) |
|---|---|---|---|
| Cuenta y autenticación | 6 | 6 (1 parcial) | 6 |
| Grupo familiar y menores | 10 | 10 (3 parciales) | 10 |
| Destinos | 9 | 9 (1 parcial) | 9 |
| Eventos y notificaciones | 10 | 10 (2 parciales) | 10 |
| Estado del dispositivo, tiempo real y verificación | 7 | 7 | 7 |
| Ubicación, historial y preferencias | 5 | 5 | 5 |
| Privacidad | 4 | 4 | 4 |
| **Total** | **51** | **51 (7 parciales)** | **51** |

---

## 4. RF → Caso de uso

Fuente: PRD §6 (CU-01 a CU-19) y §9 (CA). La columna "Pasos" indica el paso o flujo alterno del CU donde aparece el comportamiento.

### 4.1 Cuenta y autenticación

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-001 | M | Registro con correo | F-01 | CU-01 | 2–4 | Directa | — |
| RF-002 | M | Acceso con Google | F-01 | CU-01 | 2 | Directa | — |
| RF-003 | M | Verificación de correo | F-01 | CU-01 | 4–5, 5a | Directa | — |
| RF-004 | M | Recuperación de contraseña | F-01 | CU-01 | 2a | Directa | — |
| RF-005 | M | Política de contraseña | F-01 | CU-01 | 2 | Parcial | — |
| RF-006 | M | Eliminación de cuenta | F-01, F-12 | CU-09 | 1–2 | Directa | CA-09 |

### 4.2 Grupo familiar y menores

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-010 | M | Grupo familiar | F-02 | CU-01 | 6 | Directa | — |
| RF-011 | M | Invitación de tutores | F-02 | CU-08 | 1–2, 2a | Directa | — |
| RF-012 | M | Revocación de tutores | F-02 | CU-11 | 1–3, 3a | Directa | — |
| RF-013 | S | Varios grupos | F-02 | CU-12 | 1–3, 1a, 3a | Directa | — |
| RF-014 | M | Perfiles de menores | F-03 | CU-01 | 6 (asistente de alta del primer menor) | Parcial | — |
| RF-015 | M | Código de vinculación | F-03 | CU-02 | 2, 4, 4a | Directa | CA-01 |
| RF-016 | M | Un dispositivo por menor | F-03 | CU-02 | 6a | Directa | — |
| RF-017 | M | Asistente de permisos | F-03 | CU-02 | 5, 5a | Directa | CA-01 |
| RF-018 | M | Transparencia | F-03, F-12 | CU-02 | Postcondición | Parcial | — |
| RF-019 | M | Desvinculación | F-03 | CU-09 | 3 (al eliminar la cuenta) | Parcial | CA-09 |

### 4.3 Destinos

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-020 | M | Alta de destino | F-04 | CU-03 | 1–2 | Directa | CA-02 |
| RF-021 | M | Radio de geocerca | F-04 | CU-03 | 3 | Directa | CA-02 |
| RF-022 | M | Asignación a menores | F-04 | CU-03 | 5 | Directa | — |
| RF-023 | M | Ventana y hora esperada | F-04 | CU-03 | 6, 6a | Directa | — |
| RF-024 | M | Recurrencia | F-04 | CU-03 | 6 | Directa | — |
| RF-025 | M | Gestión de destinos | F-04 | CU-10 | 1–4, 3a | Directa | — |
| RF-026 | M | Límite de destinos | F-04 | CU-03, CU-10 | 7a; 2a | Directa | CA-02 |
| RF-027 | M | Sincronización | F-04 | CU-03, CU-10 | 7, 7b; 4, 4a | Directa | CA-02 |
| RF-028 | M | Zona horaria | F-04 | CU-03 | 6 | Parcial | — |

### 4.4 Eventos y notificaciones

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-030 | M | Llegada | F-05 | CU-04 | 1–4 | Directa | CA-03 |
| RF-031 | M | Llegada única | F-05 | CU-04 | 3 | Directa | CA-03 |
| RF-032 | M | Salida | F-06 | CU-05 | 1–3, 2a | Directa | CA-04 |
| RF-033 | S | Reingreso | F-06 | CU-05 | 4a | Directa | CA-03 |
| RF-034 | M | No llegada | F-07 | CU-06 | 1, 3 | Directa | CA-05 |
| RF-035 | M | Confirmación antes de alertar | F-07 | CU-06 | 2, 2a, 2b | Directa | CA-05 |
| RF-036 | M | Cola sin conexión | F-05, F-06 | CU-04 | 2a | Directa | CA-03 |
| RF-037 | M | Idempotencia | F-05, F-06 | CU-04 | 3 | Parcial | — |
| RF-038 | M | Contenido de notificación | F-05, F-06, F-07 | CU-04 | 4–5 | Parcial | — |
| RF-039 | M | Destinatarios | F-05, F-06, F-07, F-11 | CU-04, CU-05, CU-06 | 4; 3; 3 | Directa | — |

### 4.5 Estado del dispositivo, tiempo real y verificación

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-040 | M | Permiso revocado | F-08 | CU-13 | 3 | Directa | CA-06 |
| RF-041 | M | Dispositivo sin respuesta | F-08 | CU-13 | 3a | Directa | CA-06 |
| RF-042 | S | Batería baja | F-08 | CU-13 | 3b | Directa | — |
| RF-043 | M | Estado del dispositivo | F-08 | CU-13 | 1–2 | Directa | — |
| RF-044 | M | Verificación cada 20 min | F-05, F-06, F-08 | CU-14 | 1–3, 2a, 3a | Directa | CA-11 |
| RF-045 | S | Estado en tiempo real | F-13 | CU-15 | 1–2, 2a, 2b | Directa | CA-13 |
| RF-046 | S | Historial de ubicaciones puntuales | F-09, F-10 | CU-16 | 4, 4a | Directa | CA-13 |

### 4.6 Ubicación, historial y preferencias

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-050 | S | Ubicación bajo demanda | F-09 | CU-07 | 1–4, 3a | Directa | CA-07 |
| RF-051 | S | Registro de consultas | F-09 | CU-07, CU-16 | Postcondición; 2 | Directa | CA-07 |
| RF-052 | M | Historial de eventos | F-10 | CU-16 | 1–3, 3a | Directa | CA-08 |
| RF-053 | S | Preferencias | F-11 | CU-17 | 1–2, 2a | Directa | — |
| RF-054 | S | Alertas prioritarias | F-11 | CU-17, CU-05 | 3, 3a; 3 | Directa | — |

### 4.7 Privacidad

| RF | Pri. | Nombre | Funcionalidad | Caso de uso | Pasos | Relación | Criterio PRD |
|---|---|---|---|---|---|---|---|
| RF-060 | M | Consentimiento | F-12 | CU-01 | 3 | Directa | CA-09 |
| RF-061 | M | Nuevo consentimiento | F-12 | CU-18 | 1–3, 2a | Directa | — |
| RF-062 | S | Exportación | F-12 | CU-19 | 1–3, 1a, 3a | Directa | — |
| RF-063 | M | Eliminación de datos | F-12 | CU-09 | 1–3, 3a | Directa | CA-09 |

### 4.8 Índice inverso: caso de uso → RF

| Caso de uso | RF con relación directa | RF con relación parcial |
|---|---|---|
| CU-01 Registrar cuenta y crear grupo familiar | RF-001, RF-002, RF-003, RF-004, RF-010, RF-060 | RF-005, RF-014 |
| CU-02 Vincular el dispositivo del menor | RF-015, RF-016, RF-017 | RF-018 |
| CU-03 Registrar un destino | RF-020, RF-021, RF-022, RF-023, RF-024, RF-026, RF-027 | RF-028 |
| CU-04 Recibir notificación de llegada | RF-030, RF-031, RF-036, RF-039 | RF-037, RF-038 |
| CU-05 Recibir alerta de salida | RF-032, RF-033, RF-039, RF-054 | — |
| CU-06 Recibir alerta de no llegada | RF-034, RF-035, RF-039 | — |
| CU-07 Consultar ubicación bajo demanda | RF-050, RF-051 | — |
| CU-08 Invitar a un tutor secundario | RF-011 | — |
| CU-09 Eliminar cuenta y datos | RF-006, RF-063 | RF-019 |
| CU-10 Gestionar un destino existente | RF-025, RF-026, RF-027 | — |
| CU-11 Revocar a un tutor secundario | RF-012 | — |
| CU-12 Cambiar de grupo familiar | RF-013 | — |
| CU-13 Supervisar el estado del dispositivo del menor | RF-040, RF-041, RF-042, RF-043 | — |
| CU-14 Verificar periódicamente la posición del menor | RF-044 | — |
| CU-15 Ver el estado en tiempo real | RF-045 | — |
| CU-16 Consultar el historial | RF-046, RF-051, RF-052 | — |
| CU-17 Configurar preferencias de notificación | RF-053, RF-054 | — |
| CU-18 Aceptar una nueva versión del aviso de privacidad | RF-061 | — |
| CU-19 Exportar los datos del grupo | RF-062 | — |

---

## 5. RF → Entidades

Fuente: SDD §4.2 (entidades de dominio), SDD §6.3 y ETS §4.2 (tablas, columnas y restricciones). La columna "Entidad de dominio" solo lista las entidades que el SDD nombra en `saferoute-domain`; el resto de los datos vive únicamente como tabla.

### 5.1 Cuenta y autenticación

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-001 | Guardian, FamilyGroup | `guardian`, `family_group`, `family_membership`, `consent`, `account_token` | `uq_guardian_email`, `ck_guardian_email_lowercase` |
| RF-002 | Guardian | `guardian`, `external_login` | `external_login.provider` (`google`), único `(provider, subject)` |
| RF-003 | Guardian | `guardian`, `account_token` | `is_email_verified`; `account_token.purpose = 'email_verification'`, vigencia 24 h, `used_at` |
| RF-004 | Guardian | `guardian`, `account_token`, `refresh_token` | `password_hash`; `account_token.purpose = 'password_reset'`, vigencia 60 min, `used_at`; revocación de `refresh_token` |
| RF-005 | Guardian | `guardian` | `password_hash` (Argon2id); la regla se valida en el DTO, no en la base |
| RF-006 | Guardian | `guardian`, `refresh_token` | `deletion_requested_at`, `refresh_token.revoked_at` |

### 5.2 Grupo familiar y menores

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-010 | FamilyGroup | `family_group`, `family_membership` | `owner_guardian_id`, `role = 'owner'` |
| RF-011 | FamilyGroup | `invitation`, `family_membership` | `invitation.token_hash`, `expires_at`, `accepted_at` |
| RF-012 | FamilyGroup | `family_membership` | `is_active` (baja lógica) |
| RF-013 | FamilyGroup | `family_membership` | Único `(family_group_id, guardian_id)`; un tutor en N grupos |
| RF-014 | Child | `child` | `display_name`, `birth_year` (`ck_child_birth_year`), `photo_url` |
| RF-015 | Child | `pairing_code` | `code_hash`, `expires_at` (+15 min), `redeemed_at` |
| RF-016 | Device | `device` | `uq_device_child_id_active` (índice único parcial) |
| RF-017 | Device | `device` | `location_permission`, `is_precise_location`, `is_battery_optimization_exempt`, `pairing_status` |
| RF-018 | — | — | Solo presentación en la app del menor (por diseño) |
| RF-019 | Device | `device`, `refresh_token` | `device.unpaired_at`, `is_active`; `refresh_token.revoked_at` |

### 5.3 Destinos

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-020 | Destination | `destination` | `latitude`, `longitude`, `address`; `ck_destination_latitude`, `ck_destination_longitude` |
| RF-021 | Destination | `destination` | `radius_meters`, `ck_destination_radius_meters` (100–1 000) |
| RF-022 | Destination | `destination_child` | Único `(destination_id, child_id)` |
| RF-023 | DestinationSchedule | `destination_schedule` | `window_start_time`, `window_end_time`, `expected_arrival_time`, `tolerance_minutes`; `ck_destination_schedule_window_order`, `ck_destination_schedule_expected_arrival_in_window` |
| RF-024 | DestinationSchedule | `destination_schedule` | `recurrence_type`, `specific_date`, `days_of_week`; `ck_destination_schedule_recurrence` |
| RF-025 | Destination, WatchSession | `destination`, `watch_session` | `destination.status`, `is_active`, `config_version`; `watch_session.status = 'cancelled'` |
| RF-026 | Destination | `destination_child`, `destination` | Límite de 10 activos validado en el dominio (sin restricción en la base) |
| RF-027 | Destination, Device | `destination`, `device` | `destination.config_version`, `device.applied_config_version` |
| RF-028 | Destination, WatchSession | `destination`, `watch_session` | `time_zone_id`; `session_date`, `window_starts_at`, `window_ends_at` (UTC) |

### 5.4 Eventos y notificaciones

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-030 | WatchSession, SafetyEvent | `device_event`, `watch_session`, `safety_event`, `location_sample` | `device_event.type = 'GEOFENCE_DWELL'`; `watch_session.status`, `arrived_at`; `safety_event` `ARRIVAL` |
| RF-031 | SafetyEvent | `safety_event` | `dedup_key = ARRIVAL:{watch_session_id}`, `uq_safety_event_dedup_key` |
| RF-032 | WatchSession, SafetyEvent | `device_event`, `watch_session`, `safety_event` | `GEOFENCE_EXIT`; `pending_departure_since`, `last_departed_at`; `DEPARTURE`, `detected_by` |
| RF-033 | WatchSession, SafetyEvent | `watch_session`, `safety_event` | Transición `away → present`; `RETURN` |
| RF-034 | WatchSession, DestinationSchedule, SafetyEvent | `watch_session`, `destination_schedule`, `safety_event` | `arrival_deadline_at`, `status = 'overdue'`; `tolerance_minutes`; `NO_ARRIVAL` |
| RF-035 | WatchSession | `location_request`, `location_sample` | `location_request.reason = 'arrival_deadline'`, `status` |
| RF-036 | — | `device_event` | `occurred_at`, `received_at` (cola local Room en la app) |
| RF-037 | SafetyEvent, Notification | `device_event`, `safety_event`, `notification` | `device_event_id` generado en el dispositivo; `uq_safety_event_dedup_key`; `uq_notification_safety_event_id_guardian_id` |
| RF-038 | SafetyEvent, Child, Destination | `safety_event`, `cat_event_type`, `child`, `destination` | `occurred_at`; nombres resueltos desde la caché local de la app |
| RF-039 | Notification | `notification`, `notification_preference`, `guardian_push_token`, `family_membership` | `notification.status`, `guardian_push_token.invalidated_at` |

### 5.5 Estado del dispositivo, tiempo real y verificación

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-040 | Device, SafetyEvent | `device`, `device_event`, `safety_event` | `location_permission`; `PERMISSION_CHANGED`; `PERMISSION_REVOKED` |
| RF-041 | Device, WatchSession, SafetyEvent | `device`, `watch_session`, `safety_event` | `last_seen_at`; `consecutive_failed_checks`, `is_offline_alerted`; `DEVICE_OFFLINE`, `DEVICE_RECONNECTED` |
| RF-042 | Device, SafetyEvent | `device`, `device_event`, `safety_event` | `battery_level` (`ck_device_battery_level`); `BATTERY_LOW` |
| RF-043 | Device | `device` | `location_permission`, `last_seen_at`, `battery_level` |
| RF-044 | WatchSession | `watch_session`, `location_request`, `location_sample`, `scheduled_tasks` | `verification_cycle`, `last_verified_at`, `pending_departure_since`; `reason = 'verification'`; `source = 'verification'` |
| RF-045 | WatchSession, SafetyEvent | `watch_session`, `safety_event` | `status`; cursor = `safety_event_id` (UUID v7); canal `NOTIFY saferoute_realtime` (sin tabla) |
| RF-046 | SafetyEvent | `location_sample`, `safety_event` | `location_sample.source IN ('event', 'on_demand')`, `child_id`, `ix_location_sample_child_id_captured_at`; `safety_event.location_sample_id` |

### 5.6 Ubicación, historial y preferencias

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-050 | — | `location_request`, `location_sample` | `reason = 'on_demand'`, `status` (`pending`, `fulfilled`, `timed_out`); `source = 'on_demand'` |
| RF-051 | SafetyEvent | `safety_event` | `LOCATION_QUERY`, `actor_guardian_id` |
| RF-052 | SafetyEvent | `safety_event`, `cat_event_type`, `safety_event_daily_stat` | `ix_safety_event_child_id_occurred_at`; retención de 30 días y agregado anónimo posterior |
| RF-053 | — | `notification_preference`, `notification` | `is_enabled`; único `(guardian_id, destination_id, event_type_id)`; `notification.status = 'skipped'` |
| RF-054 | Notification | `notification_preference`, `cat_event_type`, `notification` | `is_priority`; `is_priority_by_default`; `notification.priority` |

### 5.7 Privacidad

| RF | Entidad de dominio | Tablas | Columnas y restricciones clave |
|---|---|---|---|
| RF-060 | — | `consent`, `privacy_notice` | `scope` (`terms`, `child_location`), `granted_at`; `privacy_notice.version` |
| RF-061 | — | `privacy_notice`, `consent` | Comparación de la última versión publicada con el consentimiento vigente |
| RF-062 | — | `data_export` | `status`, `file_path`, `expires_at` (7 días) |
| RF-063 | Todas las del agregado | Todas las tablas del titular | Borrado físico por la tarea `data-erasure` (SDD ADR-010, §4.12); `guardian.deletion_requested_at`; respaldos con retención de 21 días (caducidad máxima a los 28 días) |

---

## 6. RF → Endpoints

Fuente: SDD §7 y ETS §3. La columna "Procesamiento asíncrono" lista las tareas del Worker y los mensajes FCM y WebSocket que intervienen; varios RF del motor de vigilancia no tienen endpoint propio porque se realizan en segundo plano.

### 6.1 Cuenta y autenticación

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-001 | `POST /auth/register` | Pública | Correo de verificación (puerto `Mailer`) | `VAL-001`, `AUT-004` |
| RF-002 | `POST /auth/google` | Pública | — | `AUT-001` |
| RF-003 | `POST /auth/email/verify`, `POST /auth/email/resend`; bloqueo en `POST /family-groups/{familyGroupId}/children` | Pública / Tutor | Correo de verificación | `AUT-002`, `AUT-006` |
| RF-004 | `POST /auth/password/forgot`, `POST /auth/password/reset` | Pública | Correo de recuperación | `VAL-001`, `AUT-006` |
| RF-005 | `POST /auth/register`, `POST /auth/password/reset` | Pública | — | `VAL-001` |
| RF-006 | `DELETE /me` | Tutor | Tarea `data-erasure` | `AUT-007` |

### 6.2 Grupo familiar y menores

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-010 | `POST /auth/register` (devuelve `familyGroupId`), `GET /family-groups` | Pública / Tutor | — | — |
| RF-011 | `POST /family-groups/{familyGroupId}/invitations`, `POST /invitations/{token}/accept` | `FamilyOwner` / Tutor | — | `FAM-002`, `FAM-003` |
| RF-012 | `DELETE /family-groups/{familyGroupId}/members/{guardianId}` | `FamilyOwner` | Cierre de WebSocket con `4403` | `FAM-003`, `FAM-004` |
| RF-013 | `GET /family-groups` | Tutor | — | — |
| RF-014 | `GET`, `POST /family-groups/{familyGroupId}/children`; `GET`, `PATCH`, `DELETE /children/{childId}` | `GuardianOfFamily` / `FamilyOwner` | Tarea `data-erasure` al eliminar | `CHD-001`, `FAM-004` |
| RF-015 | `POST /children/{childId}/pairing-codes`, `POST /device/pair` | `GuardianOfFamily` / Pública | — | `DEV-001`, `DEV-003`, `DEV-004` |
| RF-016 | `POST /device/pair` | Pública | Revocación de tokens del dispositivo anterior | — |
| RF-017 | `POST /device/heartbeat`, `POST /device/events` (`PERMISSION_CHANGED`) | `ChildDevice` | — | `DEV-004` |
| RF-018 | — (solo app, por diseño) | — | — | — |
| RF-019 | `DELETE /children/{childId}/device` | `GuardianOfFamily` | Revocación de tokens; el dispositivo recibe `401` | `DEV-002` |

### 6.3 Destinos

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-020 | `POST /family-groups/{familyGroupId}/destinations`, `GET /geocoding/search`, `GET /geocoding/reverse` | `GuardianOfFamily` | Proxy a Photon con caché | `GEO-001` |
| RF-021 | `POST /family-groups/{familyGroupId}/destinations`, `PUT /destinations/{destinationId}` | `GuardianOfFamily` | — | `VAL-001` |
| RF-022 | `POST /family-groups/{familyGroupId}/destinations`, `PUT /destinations/{destinationId}` (`childIds`) | `GuardianOfFamily` | `FCM CONFIG_CHANGED` | — |
| RF-023 | Ídem (`schedules`) | `GuardianOfFamily` | Tarea `session-materializer` | `VAL-001`, `DST-003` |
| RF-024 | Ídem (`recurrenceType`, `daysOfWeek`, `specificDate`) | `GuardianOfFamily` | Tarea `session-materializer` | `VAL-001` |
| RF-025 | `PUT /destinations/{destinationId}`, `PATCH /destinations/{destinationId}/status`, `DELETE /destinations/{destinationId}` | `GuardianOfFamily` | `FCM CONFIG_CHANGED`; rematerialización de sesiones | `DST-001` |
| RF-026 | `POST /family-groups/{familyGroupId}/destinations`, `PATCH /destinations/{destinationId}/status` | `GuardianOfFamily` | — | `DST-002` |
| RF-027 | `GET /family-groups/{familyGroupId}/destinations` (`syncStatus`), `GET /device/geofences`, `POST /device/heartbeat` (`configVersion`) | `GuardianOfFamily` / `ChildDevice` | `FCM CONFIG_CHANGED` | — |
| RF-028 | — (dato `timeZoneId` en la creación del destino; por diseño) | — | Tarea `session-materializer` | — |

### 6.4 Eventos y notificaciones

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-030 | `POST /device/events` (`GEOFENCE_DWELL`) | `ChildDevice` | Tareas `process-device-event`, `dispatch-notification`; `WS alert:arrival`, `geofence:status` | `DEV-002`, `DST-001` |
| RF-031 | `POST /device/events` | `ChildDevice` | Tarea `process-device-event` | — |
| RF-032 | `POST /device/events` (`GEOFENCE_EXIT`) | `ChildDevice` | Tareas `process-device-event`, `departure-confirmation`; `WS alert:movement_detected` | — |
| RF-033 | `POST /device/events` (`GEOFENCE_DWELL` con sesión `away`) | `ChildDevice` | Tarea `process-device-event`; `WS alert:arrival` (`RETURN`) | — |
| RF-034 | — (por diseño) | — | Tareas `arrival-deadline`, `verification-timeout`; `WS alert:no_arrival` | — |
| RF-035 | `POST /device/locations` | `ChildDevice` | Tareas `arrival-deadline`, `verification-timeout` (2 min); `FCM LOCATION_REQUEST` | `LOC-002` |
| RF-036 | `POST /device/events` (lotes de hasta 100 con `occurredAt`) | `ChildDevice` | — | — |
| RF-037 | `POST /device/events` (respuesta `duplicates`) | `ChildDevice` | Identificadores deterministas de tarea | — |
| RF-038 | `GET /me/notifications` | Tutor | Tarea `dispatch-notification` (FCM de datos) | — |
| RF-039 | `GET`, `PUT /me/notification-preferences`; `PUT`, `DELETE /me/push-tokens/{token}` | Tutor | Tareas `dispatch-notification`, `push-token-cleanup` | `NTF-001` |

### 6.5 Estado del dispositivo, tiempo real y verificación

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-040 | `POST /device/events` (`PERMISSION_CHANGED`), `POST /device/heartbeat` | `ChildDevice` | Tarea `process-device-event`; `WS alert:device` | — |
| RF-041 | `POST /device/heartbeat` | `ChildDevice` | Tarea `verification-due` (reintento a +10 min); `WS alert:device` | — |
| RF-042 | `POST /device/events` (`BATTERY_LOW`), `POST /device/heartbeat` | `ChildDevice` | Tarea `process-device-event`; `WS alert:device` | — |
| RF-043 | `GET /family-groups/{familyGroupId}/children`, `GET /children/{childId}` | `GuardianOfFamily` | — | — |
| RF-044 | `POST /device/locations` | `ChildDevice` | Tareas `window-opened`, `verification-due`, `verification-timeout` (5 min), `departure-confirmation`, `window-closed`; `FCM LOCATION_REQUEST` | — |
| RF-045 | `GET /ws/v1` (`child:subscribe`, `sync:request`, `auth:refresh`), `GET /children/{childId}/watch-sessions` | Tutor | `NOTIFY saferoute_realtime`; `WS geofence:status`, `alert:*`, `sync:events` | `FAM-004`; cierres `4401`, `4403`, `4408` |
| RF-046 | `GET /locations/history/{childId}` | `GuardianOfFamily` | — | `VAL-001`, `CHD-001`, `FAM-004` |

### 6.6 Ubicación, historial y preferencias

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-050 | `POST /children/{childId}/location-requests`, `GET /location-requests/{locationRequestId}`, `POST /device/locations`; `WS location:request` / `location:ping` | `GuardianOfFamily` / `ChildDevice` | `FCM LOCATION_REQUEST`; tarea `verification-timeout` a 30 s | `LOC-001` |
| RF-051 | `POST /children/{childId}/location-requests`, `GET /children/{childId}/events` | `GuardianOfFamily` | — | — |
| RF-052 | `GET /children/{childId}/events` | `GuardianOfFamily` | Tarea `retention-purge` (30 días + agregado) | — |
| RF-053 | `GET`, `PUT /me/notification-preferences` | Tutor | — | `VAL-001` |
| RF-054 | `PUT /me/notification-preferences` (`isPriority`) | Tutor | Tarea `dispatch-notification` (canal `priority_alerts`) | — |

### 6.7 Privacidad

| RF | Endpoints | Política | Procesamiento asíncrono | Códigos de error |
|---|---|---|---|---|
| RF-060 | `GET /privacy/notice`, `POST /me/consents`, `POST /auth/register` (`acceptedPrivacyNoticeVersion`) | Pública / Tutor | — | `PRV-001` |
| RF-061 | `GET /me` (estado de consentimiento), `GET /privacy/notice`, `POST /me/consents` | Tutor | — | `PRV-001` |
| RF-062 | `POST /family-groups/{familyGroupId}/exports`, `GET /exports/{dataExportId}` | `FamilyOwner` | Tarea `data-export` | `FAM-003` |
| RF-063 | `DELETE /me`, `DELETE /children/{childId}`, `POST /family-groups/{familyGroupId}/ownership-transfer` | Tutor / `FamilyOwner` | Tareas `data-erasure` y `retention-purge`; retención de respaldos de pgBackRest | `AUT-007` |

---

## 7. RF → Android

Fuente: SDD §5 y ETS §2.4. Las pantallas son las del SDD §5.4 (modo Tutor) y §5.3 (modo Menor); los componentes, los del SDD §5.2.

### 7.1 Cuenta y autenticación

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-001 | T | `feature/auth` — Inicio de sesión / registro | `core/network` (`ApiClient`), `core/auth` (`SessionManager`, `SecureTokenStore`) |
| RF-002 | T | `feature/auth` — Inicio de sesión / registro | Credential Manager + Google Identity Services, `core/auth` |
| RF-003 | T | `feature/auth` — verificación pendiente | `core/network` (`ProblemParser`, códigos `AUT-002`, `AUT-006`) |
| RF-004 | T | `feature/auth` — recuperación de contraseña | `core/network` |
| RF-005 | T | `feature/auth` — validación del formulario | `ProblemParser` (errores `VAL-001` por campo) |
| RF-006 | T | `feature/settings` — Ajustes → eliminar cuenta (con reautenticación) | `core/auth` (limpieza de sesión) |

### 7.2 Grupo familiar y menores

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-010 | T | `feature/family` — Grupo familiar | — |
| RF-011 | T | `feature/family` — invitaciones; enlace profundo de aceptación | — |
| RF-012 | T | `feature/family` — revocar tutor | `core/realtime` (`RealtimeClient`, cierre `4403`) |
| RF-013 | T | `feature/family` — Selector de grupo familiar | — |
| RF-014 | T | `feature/children` — Detalle de menor | — |
| RF-015 | T, M | T: `feature/family` — generación del QR. M: `feature/childmode` — Vinculación | ZXing Android Embedded |
| RF-016 | M | `feature/childmode` — Vinculación (reemplazo de dispositivo) | `core/auth` |
| RF-017 | M | `feature/childmode` — Asistente de permisos | `core/permissions` (`PermissionChecker`, `OemBatteryGuide`) |
| RF-018 | M | `feature/childmode` — Pantalla de transparencia | — |
| RF-019 | T, M | T: `feature/children`. M: limpieza tras `401` | `core/location` (`GeofenceManager`), `core/data` (cola Room), WorkManager |

### 7.3 Destinos

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-020 | T | `feature/destinations` — Editor de destino | `shared/map` (`MapViewFactory`, MapLibre) |
| RF-021 | T | Editor de destino — control del radio con alternativa numérica para TalkBack | `shared/map` (`GeofenceOverlay`) |
| RF-022 | T | Editor de destino — selección de menores | — |
| RF-023 | T | Editor de destino — horario y tolerancia | — |
| RF-024 | T | Editor de destino — recurrencia | — |
| RF-025 | T, M | T: Lista de destinos. M: retiro de geocercas | `core/location` (`GeofenceManager`), `SafeRouteMessagingService` (`CONFIG_CHANGED`) |
| RF-026 | T | Editor de destino — mensaje de límite (`DST-002`) | `ProblemParser` |
| RF-027 | T, M | T: Lista de destinos ("pendiente de sincronizar"). M: sincronización | `core/location` (`GeofenceSyncWorker`, `HeartbeatWorker`, `GeofenceManager`) |
| RF-028 | T | Formato de horas en la zona del destino | `shared/util` |

### 7.4 Eventos y notificaciones

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-030 | M, T | M: segundo plano. T: notificación | M: `GeofenceManager` (geocerca `:in`, `DWELL`), `GeofenceBroadcastReceiver`, `EventUploadWorker`. T: `NotificationRenderer` |
| RF-031 | T | — (regla del servidor) | `NotificationRenderer` descarta duplicados por `safetyEventId` |
| RF-032 | M, T | M: segundo plano. T: alerta prioritaria | M: geocerca `:out` (`EXIT`), `ExitConfirmationWorker`, `GeoMath`. T: `NotificationChannels` (`priority_alerts`) |
| RF-033 | M, T | M: segundo plano. T: notificación | `GeofenceBroadcastReceiver`, `NotificationRenderer` |
| RF-034 | T | Notificación de no llegada | `NotificationRenderer`, canal `priority_alerts` |
| RF-035 | M | Segundo plano | `SafeRouteMessagingService`, `LocationRequestWorker` |
| RF-036 | M | Segundo plano | `core/data` (`SafeRouteDatabase`, cola Room), `EventUploadWorker` |
| RF-037 | M | Segundo plano | `eventId` UUID v7 generado en el dispositivo; `EventUploadWorker` retira los `duplicates` |
| RF-038 | T | Notificación | `NotificationRenderer` (caché local de nombres de menores y destinos) |
| RF-039 | T | — | `SafeRouteMessagingService` (`onNewToken`) |

### 7.5 Estado del dispositivo, tiempo real y verificación

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-040 | M, T | M: segundo plano. T: alerta | `core/permissions` (`PermissionChecker`), `HeartbeatWorker`, `NotificationRenderer` |
| RF-041 | M, T | M: segundo plano. T: Inicio ("sin conexión") | `HeartbeatWorker`, `NotificationRenderer` |
| RF-042 | M, T | M: segundo plano. T: notificación | `HeartbeatWorker`, canal `device_status` |
| RF-043 | T | Inicio, Detalle de menor | — |
| RF-044 | M | Segundo plano | `SafeRouteMessagingService`, `LocationRequestWorker` (*expedited*, `ForegroundInfo` de tipo `location`), `HeartbeatWorker`, `GeoMath` |
| RF-045 | T | Inicio (actualización en vivo) | `core/realtime` (`RealtimeClient`) |
| RF-046 | T | Historial — vista de ubicaciones puntuales | `shared/map` |

### 7.6 Ubicación, historial y preferencias

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-050 | T, M | T: `feature/location` — Detalle de menor ("Ver ubicación actual"). M: segundo plano | T: `shared/map`, `RealtimeClient` (`location:ping`). M: `LocationRequestWorker` |
| RF-051 | T | `feature/events` — Historial | — |
| RF-052 | T | `feature/events` — Historial con filtros | — |
| RF-053 | T | `feature/settings` — preferencias de notificación | — |
| RF-054 | T | `feature/settings` — alertas prioritarias | `NotificationChannels` (`priority_alerts`, `setBypassDnd`), permiso `ACCESS_NOTIFICATION_POLICY` |

### 7.7 Privacidad

| RF | Modo | Feature / pantalla | Componentes `core` y `shared` |
|---|---|---|---|
| RF-060 | T | `feature/auth` — aceptación en el registro; `feature/settings` — privacidad | — |
| RF-061 | T | `feature/settings` — Nuevo aviso de privacidad (bloquea la app hasta aceptar) | — |
| RF-062 | T | `feature/settings` — exportación | — |
| RF-063 | T, M | T: `feature/settings` (eliminar cuenta), `feature/children` (eliminar menor). M: limpieza tras desvinculación | `core/auth`, `core/location`, `core/data` |

---

## 8. RF → Tests

Fuente: ETS §5 (matriz `QA-*` y plan), SDD §15.4 (escenarios locales `ESC`) y SDD §16.6 (pruebas unitarias nombradas). Toda prueba debe etiquetarse con su RF (`@Tag("RF-xxx")`, ETS §5.14.8).

### 8.1 Cuenta y autenticación

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-001 | M | — | QA-AUT-01, QA-AUT-02 | — | Cubierto |
| RF-002 | M | — | QA-AUT-04, QA-AUT-05 | — | Cubierto |
| RF-003 | M | — | QA-AUT-06, QA-AUT-07 | — | Cubierto |
| RF-004 | M | — | QA-AUT-08, QA-AUT-09, QA-AUT-10 | — | Cubierto |
| RF-005 | M | — | QA-AUT-03 | Validadores (SDD §16.6) | Cubierto |
| RF-006 | M | CA-09 | QA-AUT-11, QA-AUT-12 | — | Cubierto |

### 8.2 Grupo familiar y menores

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-010 | M | — | QA-AUT-01 | — | Cubierto |
| RF-011 | M | — | QA-FAM-01, QA-FAM-02, QA-FAM-03 | — | Cubierto |
| RF-012 | M | — | QA-FAM-04, QA-WS-05 | — | Cubierto |
| RF-013 | S | — | QA-FAM-05 | — | Cubierto |
| RF-014 | M | — | QA-FAM-06, QA-PRV-06 | — | Cubierto |
| RF-015 | M | CA-01 | QA-FAM-07, QA-FAM-08 | — | Cubierto |
| RF-016 | M | — | QA-FAM-09 | — | Cubierto |
| RF-017 | M | CA-01 | QA-FAM-10, QA-FAM-11 | — | Cubierto |
| RF-018 | M | — | QA-FAM-12 | Prueba instrumentada (Espresso) | Cubierto |
| RF-019 | M | CA-09 | QA-FAM-09, QA-FAM-13 | — | Cubierto |

### 8.3 Destinos

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-020 | M | CA-02 | QA-DST-01, QA-DST-02 | — | Cubierto |
| RF-021 | M | CA-02 | QA-DST-03 | Restricción `ck_destination_radius_meters` | Cubierto |
| RF-022 | M | — | QA-CYC-10 | — | Cubierto |
| RF-023 | M | — | QA-DST-04 | Validadores | Cubierto |
| RF-024 | M | — | QA-CYC-01, QA-CYC-09, QA-DST-04 | `ScheduleWindow` (SDD §16.6) | Cubierto |
| RF-025 | M | — | QA-CYC-04, QA-CYC-05 | — | Cubierto |
| RF-026 | M | CA-02 | QA-DST-05 | — | Cubierto |
| RF-027 | M | CA-02 | QA-CYC-05 | — | Cubierto |
| RF-028 | M | — | QA-CYC-09 | `ScheduleWindow` | Cubierto |

### 8.4 Eventos y notificaciones

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-030 | M | CA-03 | QA-GPS-01, QA-GPS-02, QA-GPS-07, QA-GPS-10 | `GeofenceEvaluatorTest` (ETS §1.4.4); ESC Llegada | Cubierto |
| RF-031 | M | CA-03 | QA-EVT-01, QA-EVT-02 | `WatchSessionStateMachine` | Cubierto |
| RF-032 | M | CA-04 | QA-GPS-03, QA-GPS-04, QA-GPS-05, QA-GPS-06, QA-GPS-09, QA-NET-08 | `GeofenceEvaluatorTest`; ESC Salida | Cubierto |
| RF-033 | S | CA-03 | QA-EVT-03 | — | Cubierto |
| RF-034 | M | CA-05 | QA-NET-01 | ESC No llegada | Cubierto |
| RF-035 | M | CA-05 | QA-EVT-04, QA-EVT-05 | — | Cubierto |
| RF-036 | M | CA-03 | QA-NET-01, QA-NET-02, QA-NET-06, QA-NET-08, QA-NET-10 | ESC Evento diferido | Cubierto |
| RF-037 | M | — | QA-NET-05, QA-CYC-07, QA-EVT-02 | — | Cubierto |
| RF-038 | M | — | QA-NTF-01, QA-NTF-02 | `NotificationRenderer` (Robolectric) | Cubierto |
| RF-039 | M | — | QA-NTF-03, QA-NTF-04 | — | Cubierto |

### 8.5 Estado del dispositivo, tiempo real y verificación

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-040 | M | CA-06 | — | ESC Permiso revocado | Cubierto |
| RF-041 | M | CA-06 | QA-NET-03, QA-NET-04, QA-GPS-08 | ESC Sin conexión | Cubierto |
| RF-042 | S | — | QA-DEV-01, QA-DEV-02 | — | Cubierto |
| RF-043 | M | — | QA-DEV-03 | — | Cubierto |
| RF-044 | M | CA-11 | QA-CYC-01 a QA-CYC-13, QA-GPS-04, QA-GPS-06, QA-NET-07 | ESC Salida no reportada | Cubierto |
| RF-045 | S | CA-13 | QA-WS-01 a QA-WS-06, QA-NET-09 | — | Cubierto |
| RF-046 | S | CA-13 | QA-WS-07 | — | Cubierto |

### 8.6 Ubicación, historial y preferencias

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-050 | S | CA-07 | QA-CYC-08 | — | Cubierto |
| RF-051 | S | CA-07 | QA-HIS-01 | — | Cubierto |
| RF-052 | M | CA-08 | QA-HIS-02, QA-HIS-03 | — | Cubierto |
| RF-053 | S | — | QA-NTF-03, QA-NTF-05 | — | Cubierto |
| RF-054 | S | — | QA-NTF-06, QA-NTF-07 | — | Cubierto |

### 8.7 Privacidad

| RF | Pri. | Criterio PRD | Casos ETS | Pruebas unitarias / escenarios | Estado |
|---|---|---|---|---|---|
| RF-060 | M | CA-09 | QA-AUT-01, QA-PRV-01 | — | Cubierto |
| RF-061 | M | — | QA-PRV-02 | — | Cubierto |
| RF-062 | S | — | QA-PRV-03 | — | Cubierto |
| RF-063 | M | CA-09 | QA-AUT-12, QA-PRV-04, QA-PRV-05, QA-PRV-06 | — | Cubierto |

### 8.8 Pruebas transversales

Estas pruebas no se asocian a un RF concreto, pero protegen a varios a la vez:

| Prueba | Fuente | Protege |
|---|---|---|
| `ErrorsTest` (unicidad del catálogo de errores) | SDD §4.9 | Todos los RF que devuelven códigos de error |
| ArchUnit (dependencias entre capas y features) | SDD §3.1, §5.2 | Estructura de backend y app |
| Verificación de licencias del SBOM | SDD §14.4 | RNF-LIC (fuera de esta matriz) |
| Vectores compartidos `testing/geo-test-vectors.json` | ETS §1.4.5, QA-GPS-10 | RF-030, RF-032, RF-033, RF-044 |
| Prueba de campo (piloto de 20 familias) | ETS §5.14.2, PRD CA-10 | RF-030, RF-032, RF-044 y RNF de fiabilidad y batería |

---

## 9. Hallazgos y brechas

Los doce hallazgos de la versión 1.0 están cerrados. Se conservan con la versión del documento que los resuelve.

| ID | Severidad | Hallazgo | Resolución | Resuelto en | Estado |
|---|---|---|---|---|---|
| H-01 | Media | 13 RF sin caso de uso. | Casos de uso CU-10 a CU-19. | PRD v0.4 §6 | Cerrado |
| H-02 | Alta | 17 RF sin prueba definida, 12 de ellos Must. | 54 casos nuevos: `QA-AUT` (12), `QA-FAM` (13), `QA-DST` (5), `QA-EVT` (5), `QA-NTF` (7), `QA-DEV` (3), `QA-HIS` (3), `QA-PRV` (6). | ETS v1.1 §5.6 a §5.13 | Cerrado |
| H-03 | Media | Tokens de verificación de correo y de recuperación de contraseña sin modelar. | Tabla `account_token` (hash, propósito, vigencia, uso único) y código `AUT-006`. | SDD v0.4 §4.8, §6.3; ETS v1.1 §4.2 | Cerrado |
| H-04 | Alta | Borrado en ≤ 30 días, incluidos respaldos, sin mecanismo. | Tarea `data-erasure`; retención de respaldos de 21 días con caducidad máxima a los 28 días; verificación trimestral (QA-PRV-05). | SDD v0.4 §4.7, §4.12 | Cerrado |
| H-05 | Baja | Caducidad de 30 s de la consulta bajo demanda sin componente asignado. | `verification-timeout` aplica también a `on_demand` con 30 s y la programa el API. | SDD v0.4 §4.7, §7.7 | Cerrado |
| H-06 | Media | Retención del historial inconsistente entre PRD y SDD. | El SDD se alinea con el PRD F-10 y RNF-PRI-03: eventos, sesiones y notificaciones se eliminan a los 30 días y solo queda el agregado anónimo `safety_event_daily_stat`. La validación legal sigue pendiente en PT-07. | SDD v0.4 §4.12, §6.3, §6.4; ETS v1.1 §4.2, §4.5, §4.8 | Cerrado |
| H-07 | Baja | Endpoint de reportes de fallos fuera del catálogo. | `POST /api/v1/crash-reports` especificado. | SDD v0.4 §7.11; ETS v1.1 §3.6 | Cerrado |
| H-08 | Baja | Origen `heartbeat` en `location_sample` sin uso. | Se retira el valor; el latido no transporta ubicación. | SDD v0.4 §6.3; ETS v1.1 §4.2, §4.3 | Cerrado |
| H-09 | Baja | RF-045 y RF-046 sin servicio; `partition-maintenance` fuera del diagrama. | `RealtimeService` y `LocationHistoryService` en la tabla de módulos y en el diagrama, junto con las tareas nuevas. | SDD v0.4 §3.1, §4.4 | Cerrado |
| H-10 | Baja | `GeoMath`, `GeofenceSyncWorker` y cliente WebSocket fuera de la estructura Android. | Agregados a `core/location` y nuevo paquete `core/realtime` con `RealtimeClient`. | SDD v0.4 §3.2, §5.2 | Cerrado |
| H-11 | Media | Sin pantallas para elegir grupo ni para aceptar un nuevo aviso de privacidad. | Pantallas "Selector de grupo familiar" y "Nuevo aviso de privacidad", y su lugar en el flujo de arranque. | SDD v0.4 §5.3, §5.4 | Cerrado |
| H-12 | Baja | Exportación sin tarea asíncrona. | Tarea `data-export` con enlace válido 7 días. | SDD v0.4 §4.7 | Cerrado |

### 9.1 Observaciones abiertas

No son brechas de trazabilidad, pero condicionan la liberación:

| ID | Observación | Responsable |
|---|---|---|
| O-01 | Los casos de prueba están especificados, no implementados. La columna "Estado" de la §8 deberá confirmarse con el reporte de CI cuando exista código (§10). | Desarrollo y QA |
| O-02 | La retención de 30 días del historial (H-06) y el borrado físico (ADR-010) requieren validación legal (SDD PT-07, PRD PA-01). | Legal |
| O-03 | Siete RF tienen solo relación parcial con un caso de uso (§3.1). Es suficiente para la trazabilidad, pero conviene detallarlos si cambian sus reglas. | Producto |

---

## 10. Mantenimiento de la matriz

- **Cuándo se actualiza:** con cada cambio de versión del PRD, el SDD o la ETS, y en el PR que agregue un RF, un endpoint, una tabla o un caso de prueba.
- **Quién la actualiza:** el autor del cambio; la revisión del PR verifica que la matriz siga coherente.
- **Verificación automática prevista:** cuando exista el código, el reporte de CI agrupará las pruebas por la etiqueta `RF-xxx` (ETS §5.14.8) y la columna "Estado" de la §8 deberá coincidir con ese reporte. Un RF Must sin prueba etiquetada bloquea la liberación.
- **Cierre de hallazgos:** un hallazgo de la §9 se marca como cerrado indicando la versión del documento que lo resuelve; no se elimina de la tabla.
