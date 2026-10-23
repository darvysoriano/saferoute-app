# ETS — SafeRoute

**Especificación Técnica del Sistema**

| Campo | Valor |
|---|---|
| Producto | SafeRoute (aplicación móvil) |
| Documento | Especificación Técnica del Sistema: requisitos, arquitectura, contratos, modelo de datos y plan de QA |
| Versión | 1.0 |
| Estado | Borrador para revisión |
| Fecha | 2026-10-22 |
| Responsable | Darvy |
| Documentos base | [PRD-SafeRoute.md](../prd/PRD-SafeRoute.md) v0.3 · [SDD-SafeRoute.md](SDD-SafeRoute.md) v0.3 |

---

## Índice

0. [Validación contra el PRD y el SDD](#0-validación-contra-el-prd-y-el-sdd)
1. [Especificación de requisitos de software (SRS)](#1-especificación-de-requisitos-de-software-srs)
2. [Arquitectura del sistema e infraestructura gratuita](#2-arquitectura-del-sistema-e-infraestructura-gratuita)
3. [Especificación de interfaces y contratos](#3-especificación-de-interfaces-y-contratos)
4. [Modelo de datos detallado](#4-modelo-de-datos-detallado)
5. [Casos de prueba críticos y plan de aseguramiento de calidad](#5-casos-de-prueba-críticos-y-plan-de-aseguramiento-de-calidad)

---

## 0. Validación contra el PRD y el SDD

Este documento detalla, sin contradecirlos, el PRD v0.3 y el SDD v0.3. La solicitud que lo originó proponía algunos componentes distintos de los ya validados; cada diferencia se resolvió a favor de la documentación aprobada, con estas equivalencias:

| Solicitud original | Documentación validada | Resolución en este documento |
|---|---|---|
| Backend Node.js/Express | Java 25 + Spring Boot 4 (PRD §3.4, SDD ADR-003) | Backend en Java/Spring Boot. |
| MongoDB Atlas Free M0 | PostgreSQL 18 autoalojado (SDD ADR-006). MongoDB usa licencia SSPL, prohibida por RNF-LIC-03; Atlas no se ejecuta en local (RNF-OPE-01) | Modelo en PostgreSQL con entidades JPA y DDL de Flyway (§4). |
| Esquemas Mongoose | Entidades JPA + migraciones Flyway | Código completo de entidades y DDL (§4). |
| Índice `2dsphere` | Las geocercas se evalúan por destino, sin consultas de cercanía | Se documenta por qué no hace falta índice espacial y cuál usar si aparece la necesidad (§4.4). |
| Índice TTL para no superar 512 MB de Atlas M0 | Retención de 30 días por partición mensual (RNF-PRI-03, SDD §4.12) | Purga por `DROP` de partición y estimación de almacenamiento (§4.5, §4.8). |
| Servidor WebSocket | No existía en el SDD v0.2 | **Se incorpora** un canal WebSocket **solo para la app del tutor** mientras está abierta (RF-045, SDD ADR-017). |
| Seguimiento cada 20 min con Foreground Service permanente | Verificación cada 20 min iniciada por el servidor con FCM (RF-044, SDD §9) | Se mantiene el ciclo iniciado por el servidor. El Foreground Service se usa solo durante cada lectura, de forma breve, a través de WorkManager (§2.4, SDD ADR-018). |
| Registro y login de padre e hijo | El menor no tiene cuenta: su dispositivo se vincula con un código (RF-015) | Login del tutor y "login" del menor por vinculación (§3.2). |
| `POST /api/v1/destinations` | `POST /api/v1/family-groups/{familyGroupId}/destinations` (SDD §7.6) | Se especifica la ruta validada. |
| `GET /api/v1/locations/history/:childId` (historial de rutas) | El PRD excluye el rastreo continuo de rutas (§3.2) | **Se incorpora** el historial de **ubicaciones puntuales** (eventos y consultas), no una ruta continua (RF-046). |
| Roles `TUTOR`, `CHILD` en una colección `User` | Tablas `guardian` y `child` + `device` (SDD §6.3) | Se explica la correspondencia de roles (§4.1). |

Cambios introducidos en la documentación base por esta especificación:

| Documento | Cambio |
|---|---|
| PRD v0.3 | F-13 y RF-045 (estado en tiempo real por WebSocket), RF-046 (historial de ubicaciones puntuales), RNF-PER-06 (latencia WebSocket), RNF-FIA-07 (tolerancia a pérdida de red), CA-13. |
| SDD v0.3 | Canal en tiempo real (§4.13), endpoint de historial de ubicaciones, tarea `partition-maintenance`, ADR-017 y ADR-018, cifrado en reposo autoalojado, identificadores UUID v7 generados en la aplicación. |

---

## 1. Especificación de requisitos de software (SRS)

### 1.1 Convenciones

- Los identificadores conservan la numeración del PRD (`RF-001`) para mantener la trazabilidad.
- Prioridad **MoSCoW**, con equivalencia directa a la prioridad del PRD:

| MoSCoW | PRD | Significado |
|---|---|---|
| **Must** | P0 | Imprescindible para el MVP |
| **Should** | P1 | Deseable en el MVP; se libera si el calendario lo permite |
| **Could** | P2 | Posterior |
| **Won't** | Fuera de alcance | No se construye en el MVP |

### 1.2 Requisitos funcionales

#### Cuenta y autenticación

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-001 | Registro con correo | El tutor se registra con correo y contraseña. | Must | Con datos válidos se crea la cuenta (`201`) y se envía el correo de verificación; con un correo ya registrado responde `409` con `AUT-004`. |
| RF-002 | Acceso con Google | El tutor se registra o inicia sesión con su cuenta de Google. | Must | Un ID token válido de Google crea o recupera la cuenta y devuelve tokens; un token inválido responde `401` con `AUT-001`. |
| RF-003 | Verificación de correo | El correo se verifica antes de permitir el alta de menores. | Must | Sin verificar, `POST /children` responde `403` con `AUT-002`; tras verificar, la operación se permite. |
| RF-004 | Recuperación de contraseña | Restablecimiento mediante enlace enviado al correo. | Must | La respuesta es idéntica exista o no el correo; el enlace caduca a los 60 min y es de un solo uso. |
| RF-005 | Política de contraseña | Mínimo 8 caracteres con letras y números. | Must | Una contraseña que no cumple responde `400` con `VAL-001` y el detalle del campo. |
| RF-006 | Eliminación de cuenta | Eliminación tras reautenticación. | Must | Sin reautenticación en los últimos 5 min responde `401`; con ella, la cuenta queda marcada y sus datos se eliminan según RF-063. |

#### Grupo familiar y menores

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-010 | Grupo familiar | Se crea un grupo al registrar al tutor principal. | Must | Tras el registro, `GET /family-groups` devuelve un grupo con el tutor como `owner`. |
| RF-011 | Invitación de tutores | El principal invita tutores con un enlace válido por 7 días. | Must | El enlace aceptado agrega al tutor como `member`; uno vencido responde `410` con `FAM-002`. |
| RF-012 | Revocación de tutores | El principal revoca a un tutor secundario. | Must | El tutor revocado pierde acceso de inmediato (`403` en la siguiente petición) y su WebSocket se cierra con `4403`. |
| RF-013 | Varios grupos | Un tutor pertenece a más de un grupo. | Should | Un tutor invitado a un segundo grupo ve ambos en `GET /family-groups`. |
| RF-014 | Perfiles de menores | Alta, edición y baja de menores. | Must | Las operaciones devuelven el perfil actualizado; la baja elimina sus datos según RF-063. |
| RF-015 | Código de vinculación | Código de un solo uso, 15 min de vigencia, como texto y QR. | Must | Un código usado o vencido responde `400` con `DEV-001`. |
| RF-016 | Un dispositivo por menor | Como máximo un dispositivo vinculado a la vez. | Must | Vincular un segundo dispositivo desvincula el anterior y revoca sus tokens. |
| RF-017 | Asistente de permisos | La app del menor guía los permisos de ubicación en segundo plano, notificaciones y batería. | Must | Si falta "Permitir todo el tiempo", la vinculación queda `incomplete` y el tutor recibe un aviso. |
| RF-018 | Transparencia | La app del menor muestra que su ubicación se usa para avisos. | Must | La pantalla principal del modo Menor muestra el aviso en todo momento. |
| RF-019 | Desvinculación | Cualquier tutor desvincula el dispositivo. | Must | Tras desvincular, el dispositivo recibe `401` y elimina sus geocercas. |

#### Destinos

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-020 | Alta de destino | Por búsqueda de dirección o punto en el mapa. | Must | La búsqueda devuelve sugerencias en ≤ 1 s (p95); el destino se crea con sus coordenadas. |
| RF-021 | Radio de geocerca | Entre 100 y 1 000 m, 150 m por defecto. | Must | Fuera de rango responde `400` con `VAL-001`; la restricción `ck_destination_radius_meters` lo impide también en la base. |
| RF-022 | Asignación a menores | Un destino aplica a uno o varios menores. | Must | Cada menor asignado recibe la geocerca en su siguiente sincronización. |
| RF-023 | Ventana y hora esperada | Ventana de vigilancia y hora esperada opcional dentro de ella. | Must | Una hora esperada fuera de la ventana responde `422` con `DST-003`. |
| RF-024 | Recurrencia | Fecha única o días de la semana. | Must | Se materializan sesiones solo en los días configurados. |
| RF-025 | Gestión de destinos | Pausar, reanudar, editar y eliminar. | Must | Al pausar, las sesiones pendientes pasan a `cancelled` y no se envían más avisos. |
| RF-026 | Límite de destinos | Máximo 10 destinos activos por menor. | Must | El undécimo responde `422` con `DST-002`. |
| RF-027 | Sincronización | Los cambios llegan al dispositivo; si no hay conexión quedan pendientes. | Must | El tutor ve `syncStatus: pending` hasta que el dispositivo confirma la `configVersion`. |
| RF-028 | Zona horaria | Los horarios se evalúan en la zona del destino. | Must | Una sesión de 07:00 en `America/Mexico_City` abre a las 13:00 UTC. |

#### Eventos y notificaciones

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-030 | Llegada | Se confirma con permanencia ≥ 60 s dentro de la geocerca y en ventana. | Must | Se registra `ARRIVAL` y se notifica a los tutores (p95 ≤ 60 s). |
| RF-031 | Llegada única | Una notificación de llegada por destino, menor y ventana. | Must | Un segundo `DWELL` en la misma sesión no genera notificación (`dedup_key`). |
| RF-032 | Salida | Tras la llegada, fuera del radio + 50 m durante ≥ 3 min dentro de la ventana. | Must | Se registra `DEPARTURE` con alerta prioritaria; una salida de menos de 3 min no genera alerta. |
| RF-033 | Reingreso | Se notifica el regreso al destino tras una salida. | Should | Tras `DEPARTURE`, un `DWELL` en el destino genera `RETURN`. |
| RF-034 | No llegada | Alerta si no hay llegada a la hora esperada + tolerancia (5–60 min). | Must | Se emite `NO_ARRIVAL` a más tardar a hora esperada + tolerancia + 2 min. |
| RF-035 | Confirmación antes de alertar | Antes de `NO_ARRIVAL` se pide una ubicación fresca. | Must | Si el menor está dentro, se registra `ARRIVAL` y no `NO_ARRIVAL`. |
| RF-036 | Cola sin conexión | El dispositivo encola eventos con su hora original. | Must | Tras reconectar, los eventos se procesan con su `occurredAt` original. |
| RF-037 | Idempotencia | Los reenvíos no duplican notificaciones. | Must | Reenviar el mismo `eventId` devuelve `duplicates` y no crea filas nuevas. |
| RF-038 | Contenido de notificación | Menor, destino, tipo y hora real. | Must | El texto construido en la app incluye los cuatro datos; el mensaje FCM no transporta nombres (RNF-PRI-07). |
| RF-039 | Destinatarios | Todos los tutores con el aviso activo. | Must | Con dos tutores y uno con el aviso desactivado, solo uno recibe el push; ambos lo ven en la bandeja. |

#### Estado del dispositivo, tiempo real y verificación

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-040 | Permiso revocado | Alerta al revocar o reducir el permiso de ubicación. | Must | Alerta `PERMISSION_REVOKED` en < 15 min. |
| RF-041 | Dispositivo sin respuesta | Alerta si no reporta en > 30 min durante una ventana activa. | Must | `DEVICE_OFFLINE` una vez por sesión; al volver, `DEVICE_RECONNECTED`. |
| RF-042 | Batería baja | Alerta bajo 15 % en ventana activa, una vez por ventana. | Should | Una sola alerta `BATTERY_LOW` por sesión. |
| RF-043 | Estado del dispositivo | La app del tutor muestra permisos, última conexión y batería. | Must | El detalle del menor muestra los tres datos con su antigüedad. |
| RF-044 | Verificación cada 20 min | Comprobación periódica de la posición durante la ventana. | Must | Con la geocerca de salida deshabilitada, la salida se detecta en ≤ 23 min (CA-11). |
| RF-045 | Estado en tiempo real | Con la app del tutor abierta, los cambios de estado y alertas llegan por WebSocket sin recargar. | Should | Con la app abierta, un `ARRIVAL` aparece en pantalla en ≤ 2 s (p95) desde su registro en el servidor. |
| RF-046 | Historial de ubicaciones puntuales | El tutor consulta las ubicaciones asociadas a eventos y consultas de los últimos 30 días. | Should | `GET /locations/history/{childId}` devuelve solo puntos de origen `event` u `on_demand`; nunca lecturas de verificación ni puntos de más de 30 días. |

#### Ubicación, historial, preferencias y privacidad

| ID | Nombre | Descripción | Prioridad | Criterio de aceptación |
|---|---|---|---|---|
| RF-050 | Ubicación bajo demanda | Ubicación actual o, a los 30 s, la última conocida. | Should | Con conexión, ubicación de < 60 s en ≤ 30 s; sin respuesta, la última con `isFresh: false`. |
| RF-051 | Registro de consultas | Cada consulta queda en el historial con el tutor que la hizo. | Should | Se crea un `LOCATION_QUERY` visible para los demás tutores. |
| RF-052 | Historial de eventos | Lista cronológica con filtros por menor, destino, tipo y fechas. | Must | Los filtros devuelven solo coincidencias, en orden descendente. |
| RF-053 | Preferencias | Activar o desactivar cada tipo de aviso por destino. | Should | Un tipo desactivado se registra como `skipped` y no se envía push. |
| RF-054 | Alertas prioritarias | Salida, no llegada y permiso revocado como prioritarias. | Should | Llegan por el canal `priority_alerts`, que salta "No molestar" si el tutor lo autorizó. |
| RF-060 | Consentimiento | Registro con fecha y versión del aviso. | Must | Sin consentimiento vigente, el alta de menores responde `403` con `PRV-001`. |
| RF-061 | Nuevo consentimiento | Se pide al cambiar la versión del aviso. | Must | Tras publicar una versión nueva, la app la solicita antes de continuar. |
| RF-062 | Exportación | El principal exporta los datos del grupo en JSON. | Should | La exportación contiene todos los menores, destinos y eventos del grupo. |
| RF-063 | Eliminación de datos | Borrado de datos personales y de ubicación en ≤ 30 días. | Must | Auditoría de base de datos y respaldos sin datos del grupo a los 30 días. |

#### Fuera de alcance

| ID | Nombre | Prioridad | Motivo |
|---|---|---|---|
| W-01 | Rastreo continuo y reproducción de rutas | Won't | Principio "eventos, no vigilancia" (PRD §1.4) y RNF-PRI-04. |
| W-02 | App para iOS | Won't | Requiere el Apple Developer Program, de pago (PRD §3.2). |
| W-03 | Botón SOS | Won't | Posterior (PA-06). |
| W-04 | Chat tutor–menor | Won't | Posterior. |
| W-05 | Versión web para tutores | Won't | Posterior (PA-07). |

### 1.3 Requisitos no funcionales

Los RNF completos están en el PRD §8. Esta sección detalla los cuatro que la arquitectura debe garantizar de forma explícita.

#### 1.3.1 Rendimiento de batería

| ID | Requisito | Medición |
|---|---|---|
| RNF-PER-03 | Consumo adicional en el dispositivo del menor ≤ 5 % diario en un día escolar típico (2 destinos, 2 ventanas). | Batterystats (`adb shell dumpsys batterystats`) y Battery Historian en la prueba de campo, por fabricante. |

Presupuesto de energía:

| Fuente | Estimación diaria | Control de diseño |
|---|---|---|
| Geocercas de Google Play Services | 0,5–1,5 % | Geocercas nativas, sin GPS continuo; `setNotificationResponsiveness(30 s)`. |
| Verificación cada 20 min (≤ 21 lecturas por sesión) | 0,4–1,0 % | Solo en ventana; reutiliza lecturas de < 5 min; cada lectura dura ≤ 25 s. |
| Foreground Service breve por lectura | Incluido arriba | Solo durante la lectura; nunca permanente (§2.4). |
| Latido y sincronización | 0,1–0,3 % | WorkManager con restricción de red, 15 min solo en ventana. |
| Red (eventos, FCM) | 0,1–0,3 % | Lotes de eventos; ≤ 10 MB mensuales (RNF-PER-05). |
| **Total estimado** | **1,1–3,1 %** | Margen frente al límite de 5 %. |

La app del menor **no mantiene conexión WebSocket** ni servicio en primer plano permanente: son las dos causas más comunes de consumo excesivo en apps de localización.

#### 1.3.2 Seguridad de los datos de menores

| ID | Requisito | Implementación |
|---|---|---|
| RNF-PRI-02 | Ubicación del menor tratada como dato sensible, con consentimiento expreso del tutor. | `consent` con `scope = child_location`; sin él no se crea el menor (RF-060). |
| RNF-PRI-03 | Ubicaciones puntuales retenidas ≤ 30 días. | Particiones mensuales y `DROP` de las vencidas; anulación de coordenadas en `safety_event` (§4.5). |
| RNF-PRI-04 | Sin rastreo continuo. | Solo geocercas, verificación en ventana y consultas bajo demanda. Las lecturas de verificación no se muestran al tutor (RF-046). |
| RNF-PRI-07 | El proveedor de push no recibe datos personales. | FCM solo transporta identificadores; el texto se construye en el dispositivo. |
| RNF-SEG-01 | TLS 1.2+ y *certificate pinning*. | Caddy con TLS 1.3 preferente; OkHttp `CertificatePinner` en las variantes no locales. Aplica también al WebSocket (`wss://`). |
| RNF-SEG-02 | Cifrado en reposo. | Volumen de datos de PostgreSQL con LUKS2 (AES-256-XTS); respaldos de pgBackRest cifrados (`repo-cipher-type=aes-256-cbc`). |
| RNF-SEG-04 | Autorización por pertenencia en cada operación. | Comprobación de grupo familiar en cada servicio y en cada suscripción WebSocket. |
| RNF-SEG-06 | Credenciales del dispositivo protegidas. | Tokens cifrados con Tink y clave en Android Keystore. |
| RNF-SEG-08 | Sin datos personales en logs. | Los logs usan identificadores; un filtro de Logback enmascara coordenadas y correos por si se cuelan. |
| — | Mínimo de datos del menor. | Solo alias, año de nacimiento opcional y foto opcional; sin teléfono, correo ni fecha de nacimiento completa. |
| — | Acceso del personal. | Acceso a la base de producción solo con rol de solo lectura auditado; ninguna consulta de soporte devuelve coordenadas sin ticket. |

#### 1.3.3 Latencia de WebSockets

| ID | Requisito | Valor |
|---|---|---|
| RNF-PER-06 | Latencia desde el registro de un evento en la base hasta su entrega por WebSocket a un tutor conectado. | p95 ≤ 2 s, p99 ≤ 5 s |
| RNF-PER-06a | Tiempo de establecimiento de la conexión (handshake + `connection:ready`). | p95 ≤ 1 s en 4G |
| RNF-PER-06b | Intervalo de latido de aplicación | 25 s; la conexión se cierra si no hay latido en 60 s |
| RNF-PER-06c | Capacidad por instancia del API | ≥ 10 000 conexiones simultáneas |
| RNF-PER-06d | Tamaño máximo de mensaje | 16 KB |

El WebSocket es un canal de **conveniencia**: la notificación push y la bandeja siguen siendo la vía garantizada. Si el canal falla, ninguna alerta se pierde.

#### 1.3.4 Tolerancia a pérdida de red móvil

| ID | Requisito | Implementación |
|---|---|---|
| RNF-FIA-07a | Ningún evento del dispositivo del menor se pierde por falta de red. | Cola Room persistente de hasta 7 días o 5 000 eventos; envío con reintento exponencial de WorkManager y restricción de red. |
| RNF-FIA-07b | Los eventos diferidos conservan su hora real. | `occurredAt` del dispositivo; corrección si supera la hora de recepción en más de 2 min. |
| RNF-FIA-07c | Los reenvíos no duplican efectos. | `eventId` UUID v7 generado en el dispositivo y deduplicación en el servidor (RF-037). |
| RNF-FIA-07d | La app del tutor se recupera de cortes sin perder avisos. | Reconexión con espera exponencial y `sync:request` desde el último cursor (§3.5.6); la bandeja es la fuente de verdad. |
| RNF-FIA-07e | La ausencia de red no se confunde con una salida. | Sin respuesta se clasifica como `no_response`, nunca como `outside`; tras 30 min sin contacto, `DEVICE_OFFLINE` (RF-041). |

### 1.4 Algoritmo de cálculo de geocercas

#### 1.4.1 Modelo

Un destino es un círculo sobre la superficie terrestre con centro $C = (\varphi_c, \lambda_c)$ y radio $r$ en metros. Una lectura de ubicación es $P = (\varphi_p, \lambda_p)$ con precisión horizontal $a$ en metros (radio del círculo de confianza del 68 % que reporta Android). Se modela la Tierra como una esfera de radio medio $R = 6\,371\,008{,}8$ m (radio medio IUGG).

#### 1.4.2 Fórmula de Haversine

Con las latitudes $\varphi$ y longitudes $\lambda$ en radianes:

$$
\Delta\varphi = \varphi_p - \varphi_c, \qquad \Delta\lambda = \lambda_p - \lambda_c
$$

$$
h = \sin^2\!\left(\frac{\Delta\varphi}{2}\right) + \cos\varphi_c \,\cos\varphi_p \,\sin^2\!\left(\frac{\Delta\lambda}{2}\right)
$$

$$
d = 2R \cdot \operatorname{atan2}\!\left(\sqrt{h},\ \sqrt{1-h}\right)
$$

donde $d$ es la distancia del círculo máximo entre $C$ y $P$. Se usa la forma con $\operatorname{atan2}$ en lugar de $2R\arcsin\sqrt{h}$ porque es numéricamente estable para cualquier $h \in [0, 1]$; además $h$ se acota a $[0, 1]$ para absorber errores de redondeo.

**Error del modelo esférico.** Frente al elipsoide WGS84, Haversine tiene un error relativo máximo de ~0,5 %. Para los radios del producto (100–1 000 m) eso son ≤ 5 m, muy por debajo de la precisión típica del GPS (5–50 m). Por eso no se justifica Vincenty ni Karney.

#### 1.4.3 Clasificación de una lectura

Constantes del sistema (SDD §8.2 y §9.3):

| Símbolo | Constante | Valor |
|---|---|---|
| $a_{max}$ | Precisión máxima aceptada | 150 m |
| $m_{max}$ | Margen máximo por precisión | 50 m |
| $h_{s}$ | Histéresis de salida | 50 m |

$$
\text{clasificar}(d, r, a) =
\begin{cases}
\textsf{INSUFFICIENT\_ACCURACY} & \text{si } a > a_{max} \\
\textsf{INSIDE} & \text{si } d \le r + \min(a,\, m_{max}) \\
\textsf{OUTSIDE} & \text{si } d - a > r + h_s \\
\textsf{AMBIGUOUS} & \text{en otro caso}
\end{cases}
$$

- **INSIDE** es tolerante: acepta la lectura si el círculo de confianza toca el destino, con un margen máximo de 50 m.
- **OUTSIDE** es estricto: el menor está fuera aun en el caso más favorable de error del GPS, y además supera la histéresis.
- **AMBIGUOUS** no cambia el estado de la sesión. Es la banda que evita los falsos positivos por fluctuación del GPS cerca del borde.

| Estado de la sesión | Clasificación que provoca transición | Transición |
|---|---|---|
| `awaiting`, `overdue` | `INSIDE` | → `present` (llegada) |
| `present` | `OUTSIDE` dos veces separadas ≥ 3 min | → `away` (salida) |
| `away` | `INSIDE` | → `present` (reingreso) |
| Cualquiera | `INSUFFICIENT_ACCURACY` | Se pide una lectura nueva; sin transición |

#### 1.4.4 Implementación en el backend (Java)

`saferoute-domain/src/main/java/mx/saferoute/domain/common/GeoPoint.java`

```java
package mx.saferoute.domain.common;

/**
 * Punto geográfico en grados decimales WGS84.
 *
 * <p>Valida los rangos al construirse para que ninguna regla reciba coordenadas imposibles.
 */
public record GeoPoint(double latitude, double longitude) {

  public GeoPoint {
    if (Double.isNaN(latitude) || latitude < -90.0 || latitude > 90.0) {
      throw new IllegalArgumentException("Latitud fuera de rango: " + latitude);
    }
    if (Double.isNaN(longitude) || longitude < -180.0 || longitude > 180.0) {
      throw new IllegalArgumentException("Longitud fuera de rango: " + longitude);
    }
  }
}
```

`saferoute-domain/src/main/java/mx/saferoute/domain/common/LocationFix.java`

```java
package mx.saferoute.domain.common;

import java.time.Instant;

/** Lectura de ubicación reportada por el dispositivo del menor. */
public record LocationFix(GeoPoint point, double accuracyMeters, Instant capturedAt, boolean isMock) {

  public LocationFix {
    if (Double.isNaN(accuracyMeters) || accuracyMeters < 0.0) {
      throw new IllegalArgumentException("Precisión inválida: " + accuracyMeters);
    }
  }
}
```

`saferoute-domain/src/main/java/mx/saferoute/domain/rule/GeofencePosition.java`

```java
package mx.saferoute.domain.rule;

/** Resultado de comparar una lectura con la geocerca de un destino. */
public enum GeofencePosition {
  INSIDE,
  OUTSIDE,
  AMBIGUOUS,
  INSUFFICIENT_ACCURACY
}
```

`saferoute-domain/src/main/java/mx/saferoute/domain/rule/GeofenceEvaluator.java`

```java
package mx.saferoute.domain.rule;

import mx.saferoute.domain.common.GeoPoint;
import mx.saferoute.domain.common.LocationFix;

/**
 * Cálculo de distancias y clasificación de lecturas respecto de una geocerca circular.
 *
 * <p>Función pura: no depende del reloj ni de infraestructura. Las constantes coinciden con las
 * que el backend entrega al dispositivo en {@code GET /device/geofences}.
 */
public final class GeofenceEvaluator {

  /** Radio medio de la Tierra (IUGG), en metros. */
  public static final double EARTH_MEAN_RADIUS_METERS = 6_371_008.8;

  /** Precisión horizontal a partir de la cual una lectura no se usa para decidir. */
  public static final double MAX_ACCEPTED_ACCURACY_METERS = 150.0;

  /** Margen máximo que la precisión puede sumar al radio para considerar al menor dentro. */
  public static final double MAX_ACCURACY_MARGIN_METERS = 50.0;

  /** Distancia adicional al radio que debe superarse para considerar una salida. */
  public static final double DEPARTURE_HYSTERESIS_METERS = 50.0;

  private GeofenceEvaluator() {}

  /** Distancia del círculo máximo entre dos puntos, en metros, con la fórmula de Haversine. */
  public static double distanceMeters(GeoPoint from, GeoPoint to) {
    double fromLatitude = Math.toRadians(from.latitude());
    double toLatitude = Math.toRadians(to.latitude());
    double deltaLatitude = toLatitude - fromLatitude;
    double deltaLongitude = Math.toRadians(to.longitude() - from.longitude());

    double sinHalfDeltaLatitude = Math.sin(deltaLatitude / 2.0);
    double sinHalfDeltaLongitude = Math.sin(deltaLongitude / 2.0);
    double haversine =
        sinHalfDeltaLatitude * sinHalfDeltaLatitude
            + Math.cos(fromLatitude)
                * Math.cos(toLatitude)
                * sinHalfDeltaLongitude
                * sinHalfDeltaLongitude;

    // Acota errores de redondeo para que sqrt(1 - h) nunca reciba un negativo.
    double boundedHaversine = Math.min(1.0, Math.max(0.0, haversine));
    double centralAngle =
        2.0 * Math.atan2(Math.sqrt(boundedHaversine), Math.sqrt(1.0 - boundedHaversine));
    return EARTH_MEAN_RADIUS_METERS * centralAngle;
  }

  /** Clasifica una lectura respecto de la geocerca de centro {@code center} y radio dado. */
  public static GeofencePosition classify(GeoPoint center, int radiusMeters, LocationFix fix) {
    double accuracy = fix.accuracyMeters();
    if (accuracy > MAX_ACCEPTED_ACCURACY_METERS) {
      return GeofencePosition.INSUFFICIENT_ACCURACY;
    }

    double distance = distanceMeters(center, fix.point());
    if (distance <= radiusMeters + Math.min(accuracy, MAX_ACCURACY_MARGIN_METERS)) {
      return GeofencePosition.INSIDE;
    }
    if (distance - accuracy > radiusMeters + DEPARTURE_HYSTERESIS_METERS) {
      return GeofencePosition.OUTSIDE;
    }
    return GeofencePosition.AMBIGUOUS;
  }
}
```

Pruebas unitarias (JUnit 5 + AssertJ), con el nombrado `<unidad>_<escenario>_<resultado>`:

```java
package mx.saferoute.domain.rule;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.within;

import java.time.Instant;
import mx.saferoute.domain.common.GeoPoint;
import mx.saferoute.domain.common.LocationFix;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;

class GeofenceEvaluatorTest {

  private static final GeoPoint SCHOOL = new GeoPoint(19.371234, -99.178456);
  private static final int RADIUS = 150;

  /** Un grado de latitud sobre la esfera de radio medio mide R·π/180 ≈ 111 195,08 m. */
  @Test
  void distanceMeters_unGradoDeLatitud_mideRadioPorPiEntre180() {
    double distance = GeofenceEvaluator.distanceMeters(new GeoPoint(0, 0), new GeoPoint(1, 0));

    assertThat(distance).isCloseTo(111_195.08, within(0.1));
  }

  @Test
  void distanceMeters_mismoPunto_esCero() {
    assertThat(GeofenceEvaluator.distanceMeters(SCHOOL, SCHOOL)).isZero();
  }

  @Test
  void distanceMeters_cruceDelAntimeridiano_usaElCaminoCorto() {
    double distance =
        GeofenceEvaluator.distanceMeters(new GeoPoint(0, 179.9995), new GeoPoint(0, -179.9995));

    assertThat(distance).isCloseTo(111.195, within(0.01));
  }

  @ParameterizedTest(name = "desplazamiento {0}°, precisión {1} m → {2}")
  @CsvSource({
    "0.0010, 10.0, INSIDE",               // ~111 m del centro
    "0.0016, 30.0, INSIDE",               // ~178 m: dentro por el margen de precisión (150 + 30)
    "0.0019, 20.0, AMBIGUOUS",            // ~211 m: ni dentro ni fuera con histéresis
    "0.0030, 20.0, OUTSIDE",              // ~334 m: 334 − 20 > 150 + 50
    "0.0030, 140.0, AMBIGUOUS",           // ~334 m con mala precisión: no se puede afirmar la salida
    "0.0010, 200.0, INSUFFICIENT_ACCURACY"
  })
  void classify_desplazamientoYPrecision_clasificaSegunLasBandas(
      double latitudeOffset, double accuracy, GeofencePosition expected) {
    GeoPoint point = new GeoPoint(SCHOOL.latitude() + latitudeOffset, SCHOOL.longitude());
    LocationFix fix = new LocationFix(point, accuracy, Instant.EPOCH, false);

    assertThat(GeofenceEvaluator.classify(SCHOOL, RADIUS, fix)).isEqualTo(expected);
  }
}
```

#### 1.4.5 Coherencia entre backend y dispositivo

El dispositivo confirma las salidas localmente (SDD §5.5) con la misma clasificación. Para no duplicar el conocimiento en dos implementaciones que diverjan:

- Las constantes (`MAX_ACCEPTED_ACCURACY_METERS`, `MAX_ACCURACY_MARGIN_METERS`, `DEPARTURE_HYSTERESIS_METERS`) las entrega el backend en `GET /device/geofences`; la app no las codifica.
- Ambas implementaciones (`GeofenceEvaluator` en el backend y `GeoMath` en la app) se verifican con el mismo archivo de vectores de prueba versionado, `testing/geo-test-vectors.json`, que contiene pares de puntos, distancias esperadas y clasificaciones. Una diferencia rompe el build de cualquiera de los dos.

---

## 2. Arquitectura del sistema e infraestructura gratuita

### 2.1 Diagrama de contenedores (C4, nivel 2)

```text
                         ┌──────────────────────────┐          ┌──────────────────────────┐
                         │ [Persona] Tutor           │          │ [Persona] Menor           │
                         │ Padre, madre o tutor      │          │ Usuario del dispositivo   │
                         └────────────┬─────────────┘          └────────────┬─────────────┘
                                      │ usa                                  │ porta (sin interacción)
                                      ▼                                      ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│ [Sistema] SafeRoute                                                                              │
│                                                                                                  │
│  ┌──────────────────────────────┐                        ┌──────────────────────────────┐       │
│  │ [Contenedor] App Tutor        │                        │ [Contenedor] App Menor        │       │
│  │ Android · Java · Views        │                        │ Android · Java (mismo APK)    │       │
│  │ Destinos, historial, avisos,  │                        │ Geocercas, cola offline,      │       │
│  │ estado en vivo                │                        │ respuestas de ubicación       │       │
│  └───┬───────────┬──────────┬───┘                        └───────────┬──────────────┬───┘       │
│      │ HTTPS/JSON│ WSS/JSON │ HTTPS (teselas)                          │ HTTPS/JSON    │           │
│      │ REST      │ tiempo   │                                          │ REST          │           │
│      ▼           ▼ real     ▼                                          ▼               │           │
│  ┌────────────────────────────────────────┐     ┌──────────────────────────────┐     │           │
│  │ [Contenedor] Caddy                      │     │ [Contenedor] Martin           │     │           │
│  │ Proxy inverso, TLS (Let's Encrypt)      │────▶│ Teselas vectoriales OSM       │     │           │
│  └───────────────────┬────────────────────┘     └──────────────────────────────┘     │           │
│                      │ HTTP / WebSocket                                                │           │
│                      ▼                                                                 │           │
│  ┌──────────────────────────────────────────────────┐     ┌────────────────────────┐ │           │
│  │ [Contenedor] Backend API — saferoute-api          │────▶│ [Contenedor] Photon     │ │           │
│  │ Java 25 · Spring Boot 4                           │HTTP │ Geocodificador OSM      │ │           │
│  │ • REST /api/v1 (tutores y dispositivos)           │     └────────────────────────┘ │           │
│  │ • Servidor WebSocket /ws/v1 (solo tutores)        │                                 │           │
│  │ • Programa tareas en la cola de PostgreSQL        │◀────────────────────────────────┘           │
│  │ • LISTEN saferoute_realtime                       │                                             │
│  └───────────────┬───────────────────────────────────┘                                             │
│                  │ JDBC (SQL, LISTEN/NOTIFY)                                                       │
│                  ▼                                                                                  │
│  ┌──────────────────────────────────────────────────┐     ┌────────────────────────────────────┐  │
│  │ [Contenedor de datos] PostgreSQL 18               │◀────│ [Contenedor] Worker — saferoute-worker│  │
│  │ Datos transaccionales + scheduled_tasks           │JDBC │ Java 25 · Spring Boot · db-scheduler│  │
│  │ (cola de tareas) + canal NOTIFY                   │     │ Sesiones, verificación 20 min,      │  │
│  └──────────────────────────────────────────────────┘     │ despacho de notificaciones, NOTIFY  │  │
│                                                            └──────────────────┬─────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┼────────────────────┘
                                                                                │ HTTPS (FCM HTTP v1)
                                                                                ▼
                                              ┌───────────────────────────────────────────────────┐
                                              │ [Sistema externo] Firebase Cloud Messaging         │
                                              │ Plan gratuito · mensajes de datos                   │
                                              │ → App Tutor: avisos (texto construido en la app)    │
                                              │ → App Menor: LOCATION_REQUEST, CONFIG_CHANGED       │
                                              └───────────────────────────────────────────────────┘
```

Relaciones:

| Origen | Destino | Protocolo | Propósito |
|---|---|---|---|
| App Tutor | Backend API | HTTPS REST | Cuenta, grupo, menores, destinos, historial, consultas de ubicación |
| App Tutor | Backend API (servidor WebSocket) | WSS, JSON | Estado y alertas en vivo mientras la app está abierta (RF-045) |
| App Tutor | Martin (vía Caddy) | HTTPS | Teselas, estilo, fuentes |
| App Menor | Backend API | HTTPS REST | Vinculación, eventos, ubicaciones, latido, configuración de geocercas |
| Backend API | PostgreSQL | JDBC | Persistencia, programación de tareas, `LISTEN` |
| Backend API | Photon | HTTP | Búsqueda de direcciones (proxy con caché) |
| Worker | PostgreSQL | JDBC | Ejecución de tareas, `NOTIFY` de eventos en tiempo real |
| Worker | FCM | HTTPS | Push de datos a tutores y al menor |
| FCM | App Tutor / App Menor | Canal de Google Play Services | Entrega de mensajes |

**Por qué el servidor WebSocket vive en el contenedor del API.** Comparte autenticación, autorización y acceso a datos con el REST, y el volumen esperado (≤ 10 000 conexiones por instancia) no justifica un servicio aparte. Si hiciera falta escalarlo por separado, la misma imagen se despliega con el perfil `realtime`, que expone solo `/ws/v1`.

**Distribución de eventos entre instancias.** El Worker que registra un evento ejecuta `NOTIFY saferoute_realtime` con un mensaje mínimo (`{kind, familyGroupId, childId, refId}`). Cada instancia del API mantiene un `LISTEN` en una conexión dedicada, carga el detalle por identificador y lo envía a sus sesiones WebSocket suscritas. Así no hace falta un broker (SDD ADR-005) y PostgreSQL sigue siendo el único componente con estado.

### 2.2 Diagrama de contenedores en Mermaid

```mermaid
C4Container
    title SafeRoute — Diagrama de contenedores
    Person(tutor, "Tutor", "Padre, madre o tutor")
    Person(menor, "Menor", "Porta el dispositivo vinculado")

    System_Boundary(sr, "SafeRoute") {
        Container(appTutor, "App Tutor", "Android, Java", "Destinos, historial, avisos y estado en vivo")
        Container(appMenor, "App Menor", "Android, Java", "Geocercas, cola offline y respuestas de ubicación")
        Container(caddy, "Caddy", "Proxy inverso", "TLS y enrutamiento")
        Container(api, "Backend API", "Java 25, Spring Boot 4", "REST /api/v1 y servidor WebSocket /ws/v1")
        Container(worker, "Worker", "Java 25, Spring Boot, db-scheduler", "Sesiones, verificación cada 20 min, notificaciones")
        ContainerDb(db, "PostgreSQL 18", "Base de datos", "Datos, cola de tareas y canal NOTIFY")
        Container(martin, "Martin", "Servidor de teselas", "Mapa base OSM")
        Container(photon, "Photon", "Geocodificador Java", "Búsqueda de direcciones OSM")
    }
    System_Ext(fcm, "Firebase Cloud Messaging", "Plan gratuito")

    Rel(tutor, appTutor, "Usa")
    Rel(menor, appMenor, "Porta")
    Rel(appTutor, caddy, "REST, WSS, teselas", "HTTPS")
    Rel(appMenor, caddy, "REST", "HTTPS")
    Rel(caddy, api, "Proxy", "HTTP/WebSocket")
    Rel(caddy, martin, "Proxy", "HTTP")
    Rel(api, db, "SQL, LISTEN", "JDBC")
    Rel(api, photon, "Búsqueda", "HTTP")
    Rel(worker, db, "Tareas, NOTIFY", "JDBC")
    Rel(worker, fcm, "Push de datos", "HTTPS")
    Rel(fcm, appTutor, "Avisos")
    Rel(fcm, appMenor, "Solicitudes de ubicación")
```

### 2.3 Infraestructura gratuita

Todo el software es de código abierto o gratuito (SDD §14). Opciones de alojamiento, sin costo de licencia:

| Ambiente | Alojamiento | Costo | Observaciones |
|---|---|---|---|
| Local (`development`) | Equipo del desarrollador: Docker Engine en WSL2 o Linux, o Podman (SDD §15) | Ninguno | Ejecución completa con un comando; emulador Android con Google Play. |
| QA / UAT | Servidor Linux existente de la organización, o una VM de un nivel gratuito permanente de nube (p. ej., Oracle Cloud Always Free, instancias Ampere A1) | Ninguno | El nivel gratuito exige registrar una cuenta y puede reclamar instancias inactivas; válido para QA, no recomendado para producción. |
| Producción | Servidores Linux propios (on-premise) o VPS | Solo infraestructura (hardware o VPS), sin licencias | Topología del SDD §3.3: Caddy, API ×2, Worker ×2, PostgreSQL primario + réplica, pgBackRest, Martin, Photon, Prometheus/Grafana/Loki. |
| Notificaciones | Firebase Cloud Messaging, plan Spark | Gratuito, sin límite de mensajes | Proyecto separado por ambiente. |
| TLS | Let's Encrypt vía Caddy | Gratuito | Renovación automática. |
| Mapas y búsqueda | Martin + Photon autoalojados con datos OSM | Gratuito | Atribución ODbL obligatoria. |

Dimensionamiento inicial (piloto de 20 familias y crecimiento hasta ~5 000):

| Recurso | Piloto | ~5 000 familias |
|---|---|---|
| API + Worker | 2 vCPU, 4 GB RAM | 4 vCPU, 8 GB RAM (2 instancias de cada uno) |
| PostgreSQL | 2 vCPU, 4 GB RAM, 20 GB SSD | 4 vCPU, 16 GB RAM, 200 GB SSD |
| Photon (índice de México) | 2 vCPU, 6 GB RAM, 15 GB SSD | Igual |
| Martin + teselas de México | 1 vCPU, 1 GB RAM, 3 GB SSD | Igual + caché de Caddy |

### 2.4 Persistencia y tareas en segundo plano en Android

#### 2.4.1 Restricciones del sistema operativo

| Restricción | Desde | Efecto sobre SafeRoute |
|---|---|---|
| Doze y App Standby | Android 6 | Sin red ni alarmas para apps en reposo, salvo FCM de prioridad alta y ventanas de mantenimiento. |
| Límites de ubicación en segundo plano | Android 8 | Unas pocas lecturas por hora en segundo plano; suficiente para 3 lecturas/hora. |
| Permiso `ACCESS_BACKGROUND_LOCATION` separado | Android 10 | Necesario para geocercas y lecturas con la app cerrada. |
| Prohibición de iniciar un Foreground Service desde segundo plano | Android 12 | Solo con exenciones: **mensaje FCM de prioridad alta**, **transición de geocerca**, trabajo *expedited* de WorkManager, entre otras. |
| Tipo obligatorio de Foreground Service y permiso `FOREGROUND_SERVICE_LOCATION` | Android 14 | El servicio debe declarar `foregroundServiceType="location"` y tener el permiso de ubicación concedido al iniciarse. |
| *Buckets* de App Standby | Android 9 | Limitan trabajos y alarmas según el uso; la app del menor casi no se abre, así que cae en buckets restrictivos. |
| Capas de ahorro de energía de fabricantes | Variable | Terminan procesos y bloquean el arranque automático (Xiaomi, Oppo, Vivo, Samsung). |

#### 2.4.2 Por qué no un Foreground Service permanente

Un servicio en primer plano de ubicación continuo evita que el sistema mate el proceso, pero:

- Muestra una notificación fija en el teléfono del menor todo el día.
- Mantiene el GPS o el proveedor fusionado activos y rompe el presupuesto de batería (§1.3.1).
- Contradice RNF-PRI-04 (sin rastreo continuo).
- Exige justificar ante Google Play una declaración de servicio de ubicación continua, con más riesgo de rechazo (R-04).

En su lugar, SafeRoute combina tres mecanismos que el sistema **no elimina** porque son las vías previstas para este caso:

| Mecanismo | Rol | Por qué sobrevive |
|---|---|---|
| Geocercas de Google Play Services | Señal principal de llegada y salida | Las gestiona el proceso de Play Services, no la app; despiertan la app con un `PendingIntent` aunque esté detenida. |
| FCM de prioridad alta | Disparador del ciclo de 20 min y de las consultas | Atraviesa Doze y es una exención para iniciar trabajo en primer plano. |
| WorkManager *expedited* con `ForegroundInfo` | Ejecución de cada lectura y envío | WorkManager persiste los trabajos en su base de datos, los reprograma tras reinicios y ejecuta el trabajo *expedited* como Foreground Service breve cuando el sistema lo requiere. |

El **Foreground Service sí se usa, pero solo durante cada lectura** (≤ 30 s), con tipo `location`, a través de `getForegroundInfo()` de WorkManager. El ciclo de 20 minutos lo marca el servidor (SDD §9), no un temporizador del dispositivo, así que no depende de que el proceso siga vivo.

#### 2.4.3 Ciclo de 20 minutos: secuencia en el dispositivo

```mermaid
sequenceDiagram
    autonumber
    participant W as saferoute-worker
    participant FCM as FCM
    participant MS as SafeRouteMessagingService
    participant WM as WorkManager
    participant LW as LocationRequestWorker<br/>(expedited, FGS tipo location)
    participant FLP as FusedLocationProviderClient
    participant API as saferoute-api

    W->>FCM: LOCATION_REQUEST (prioridad alta, ttl 300 s)
    FCM->>MS: onMessageReceived (app despertada aunque esté detenida)
    MS->>WM: enqueueUniqueWork("location-request", REPLACE, expedited)
    WM->>LW: Ejecuta; en Android < 12 o sin cuota, como Foreground Service
    LW->>LW: setForeground(ForegroundInfo tipo LOCATION)
    LW->>FLP: getCurrentLocation(PRIORITY_HIGH_ACCURACY, 25 s)
    FLP-->>LW: Location (lat, lon, accuracy, isMock)
    LW->>API: POST /device/locations
    alt Sin red
        LW->>LW: Guarda en Room y encola EventUploadWorker con restricción de red
    end
    LW-->>WM: Result.success() → termina el Foreground Service
```

#### 2.4.4 Activación y desactivación del ciclo

| Momento | Servidor | Dispositivo |
|---|---|---|
| Apertura de ventana | `window-opened` programa `verification-due` a +20 min. | Recibe `CONFIG_CHANGED` o ya tiene la ventana en `GET /device/geofences`; programa `HeartbeatWorker` único con `PeriodicWorkRequest` de 15 min. |
| Durante la ventana | Un ciclo cada 20 min (SDD §9.3). | Responde a cada `LOCATION_REQUEST`; envía latidos. |
| Cierre de ventana | `window-closed` cierra la sesión; las tareas de ciclo posteriores se descartan como obsoletas. | `HeartbeatWorker` comprueba que no queda ventana activa y se cancela con `cancelUniqueWork("heartbeat")`. |
| Destino pausado, editado o desasignado | Sesión `cancelled`; `CONFIG_CHANGED` al dispositivo. | Retira las geocercas del destino y recalcula sus ventanas. |
| Reinicio del dispositivo | Sin cambios; el ciclo sigue programado en el servidor. | `BootReceiver` vuelve a registrar las geocercas; WorkManager restaura sus trabajos. |
| Dispositivo desvinculado | Tokens revocados. | Elimina geocercas, cola y trabajos. |

#### 2.4.5 Implementación

`AndroidManifest.xml` (extracto)

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools">

    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
    <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
    <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />
    <uses-permission android:name="android.permission.ACCESS_NOTIFICATION_POLICY" />

    <application android:name=".SafeRouteApplication" ...>

        <receiver
            android:name=".core.location.GeofenceBroadcastReceiver"
            android:exported="false" />

        <receiver
            android:name=".core.location.BootReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
                <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
            </intent-filter>
        </receiver>

        <service
            android:name=".core.notifications.SafeRouteMessagingService"
            android:exported="false">
            <intent-filter>
                <action android:name="com.google.firebase.MESSAGING_EVENT" />
            </intent-filter>
        </service>

        <!-- Servicio de WorkManager que ejecuta los trabajos en primer plano: se declara el tipo location. -->
        <service
            android:name="androidx.work.impl.foreground.SystemForegroundService"
            android:foregroundServiceType="location"
            tools:node="merge" />
    </application>
</manifest>
```

`core/location/GeofenceManager.java` (registro de las dos geocercas por destino)

```java
package mx.saferoute.app.core.location;

import android.annotation.SuppressLint;
import android.app.PendingIntent;
import android.content.Context;
import android.content.Intent;
import com.google.android.gms.location.Geofence;
import com.google.android.gms.location.GeofencingClient;
import com.google.android.gms.location.GeofencingRequest;
import com.google.android.gms.location.LocationServices;
import com.google.android.gms.tasks.Task;
import java.util.ArrayList;
import java.util.List;

/** Registra en Google Play Services las geocercas de los destinos asignados al menor. */
public final class GeofenceManager {

  private static final int DWELL_DELAY_MILLIS = 60_000;
  private static final int RESPONSIVENESS_MILLIS = 30_000;

  private final Context context;
  private final GeofencingClient geofencingClient;

  public GeofenceManager(Context context) {
    this.context = context.getApplicationContext();
    this.geofencingClient = LocationServices.getGeofencingClient(this.context);
  }

  /** Reemplaza todas las geocercas por las de la configuración recibida del servidor. */
  @SuppressLint("MissingPermission") // El llamador verifica los permisos con PermissionChecker.
  public Task<Void> replaceAll(GeofenceConfig config) {
    List<Geofence> geofences = new ArrayList<>();
    for (GeofenceConfig.Destination destination : config.destinations()) {
      geofences.add(
          new Geofence.Builder()
              .setRequestId("dst:" + destination.destinationId() + ":in")
              .setCircularRegion(
                  destination.latitude(), destination.longitude(), destination.radiusMeters())
              .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_DWELL)
              .setLoiteringDelay(DWELL_DELAY_MILLIS)
              .setNotificationResponsiveness(RESPONSIVENESS_MILLIS)
              .setExpirationDuration(Geofence.NEVER_EXPIRE)
              .build());
      geofences.add(
          new Geofence.Builder()
              .setRequestId("dst:" + destination.destinationId() + ":out")
              .setCircularRegion(
                  destination.latitude(),
                  destination.longitude(),
                  destination.radiusMeters() + (float) config.departureHysteresisMeters())
              .setTransitionTypes(Geofence.GEOFENCE_TRANSITION_EXIT)
              .setNotificationResponsiveness(RESPONSIVENESS_MILLIS)
              .setExpirationDuration(Geofence.NEVER_EXPIRE)
              .build());
    }

    if (geofences.isEmpty()) {
      return geofencingClient.removeGeofences(pendingIntent());
    }

    GeofencingRequest request =
        new GeofencingRequest.Builder()
            .setInitialTrigger(GeofencingRequest.INITIAL_TRIGGER_DWELL)
            .addGeofences(geofences)
            .build();

    return geofencingClient
        .removeGeofences(pendingIntent())
        .continueWithTask(ignored -> geofencingClient.addGeofences(request, pendingIntent()));
  }

  private PendingIntent pendingIntent() {
    Intent intent = new Intent(context, GeofenceBroadcastReceiver.class);
    // Play Services necesita un PendingIntent mutable para adjuntar los datos de la transición.
    return PendingIntent.getBroadcast(
        context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT | PendingIntent.FLAG_MUTABLE);
  }
}
```

`core/location/LocationRequestWorker.java` (lectura breve en primer plano)

```java
package mx.saferoute.app.core.location;

import android.annotation.SuppressLint;
import android.app.Notification;
import android.content.Context;
import android.content.pm.ServiceInfo;
import android.location.Location;
import android.os.Build;
import androidx.annotation.NonNull;
import androidx.core.app.NotificationCompat;
import androidx.hilt.work.HiltWorker;
import androidx.work.ForegroundInfo;
import androidx.work.Worker;
import androidx.work.WorkerParameters;
import com.google.android.gms.location.CurrentLocationRequest;
import com.google.android.gms.location.FusedLocationProviderClient;
import com.google.android.gms.location.Priority;
import com.google.android.gms.tasks.CancellationTokenSource;
import com.google.android.gms.tasks.Tasks;
import dagger.assisted.Assisted;
import dagger.assisted.AssistedInject;
import java.util.concurrent.TimeUnit;
import mx.saferoute.app.R;
import mx.saferoute.app.core.notifications.NotificationChannels;

/**
 * Obtiene una ubicación de alta precisión en respuesta a un LOCATION_REQUEST y la envía al API.
 *
 * <p>Se encola como trabajo expedited. Cuando el sistema lo ejecuta como Foreground Service, la
 * notificación es silenciosa y dura solo lo que tarda la lectura.
 */
@HiltWorker
public final class LocationRequestWorker extends Worker {

  public static final String KEY_LOCATION_REQUEST_ID = "locationRequestId";
  private static final int NOTIFICATION_ID = 2001;
  private static final long LOCATION_TIMEOUT_SECONDS = 25;
  private static final long MAX_CACHED_AGE_MILLIS = 60_000;

  private final FusedLocationProviderClient locationClient;
  private final DeviceLocationUploader uploader;

  @AssistedInject
  public LocationRequestWorker(
      @Assisted @NonNull Context context,
      @Assisted @NonNull WorkerParameters params,
      FusedLocationProviderClient locationClient,
      DeviceLocationUploader uploader) {
    super(context, params);
    this.locationClient = locationClient;
    this.uploader = uploader;
  }

  @NonNull
  @Override
  public ForegroundInfo getForegroundInfo() {
    Notification notification =
        new NotificationCompat.Builder(getApplicationContext(), NotificationChannels.LOCATION_CHECK)
            .setSmallIcon(R.drawable.ic_location_check)
            .setContentTitle(getApplicationContext().getString(R.string.location_check_title))
            .setPriority(NotificationCompat.PRIORITY_MIN)
            .setSilent(true)
            .build();
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q) {
      return new ForegroundInfo(
          NOTIFICATION_ID, notification, ServiceInfo.FOREGROUND_SERVICE_TYPE_LOCATION);
    }
    return new ForegroundInfo(NOTIFICATION_ID, notification);
  }

  @SuppressLint("MissingPermission") // El permiso se verifica antes de encolar el trabajo.
  @NonNull
  @Override
  public Result doWork() {
    String locationRequestId = getInputData().getString(KEY_LOCATION_REQUEST_ID);
    CancellationTokenSource cancellation = new CancellationTokenSource();
    CurrentLocationRequest request =
        new CurrentLocationRequest.Builder()
            .setPriority(Priority.PRIORITY_HIGH_ACCURACY)
            .setMaxUpdateAgeMillis(MAX_CACHED_AGE_MILLIS)
            .setDurationMillis(TimeUnit.SECONDS.toMillis(LOCATION_TIMEOUT_SECONDS))
            .build();
    try {
      Location location =
          Tasks.await(
              locationClient.getCurrentLocation(request, cancellation.getToken()),
              LOCATION_TIMEOUT_SECONDS + 5,
              TimeUnit.SECONDS);
      // Si el envío falla por red, el uploader deja la lectura en la cola Room para reintento.
      uploader.uploadOrEnqueue(locationRequestId, location);
      return Result.success();
    } catch (Exception e) {
      cancellation.cancel();
      uploader.reportUnavailable(locationRequestId);
      return Result.success(); // Sin lectura no se reintenta: el ciclo siguiente lo cubre.
    }
  }
}
```

`core/notifications/SafeRouteMessagingService.java` (extracto del modo Menor)

```java
@Override
public void onMessageReceived(@NonNull RemoteMessage message) {
  Map<String, String> data = message.getData();
  if ("LOCATION_REQUEST".equals(data.get("kind"))) {
    OneTimeWorkRequest work =
        new OneTimeWorkRequest.Builder(LocationRequestWorker.class)
            .setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)
            .setInputData(
                new Data.Builder()
                    .putString(
                        LocationRequestWorker.KEY_LOCATION_REQUEST_ID,
                        data.get("locationRequestId"))
                    .build())
            .build();
    WorkManager.getInstance(this)
        .enqueueUniqueWork("location-request", ExistingWorkPolicy.REPLACE, work);
  } else if ("CONFIG_CHANGED".equals(data.get("kind"))) {
    GeofenceSyncWorker.enqueueNow(this);
  }
}
```

#### 2.4.6 Medidas frente a capas de fabricantes

| Medida | Detalle |
|---|---|
| Exclusión de optimización de batería | `ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS` en el asistente de vinculación, con explicación. Permitido por la política de Google Play para apps cuya función principal depende de ello; se justifica en la declaración de la ficha. |
| Guía por fabricante (`OemBatteryGuide`) | Pantallas específicas de "Inicio automático" y "Sin restricciones" para Xiaomi, Oppo, Vivo, Huawei y Samsung, con enlace directo a la pantalla del sistema cuando existe. |
| Detección | El servidor mide `saferoute.verification.outcome` y `device.reporting_gap` por fabricante y modelo; un menor sin respuesta repetida genera `DEVICE_OFFLINE` y la app del tutor muestra la guía. |
| Autorreparación | `HeartbeatWorker` vuelve a registrar las geocercas si detecta que no existen (`GEOFENCE_NOT_AVAILABLE`, borrado de datos de Play Services). |

---

## 3. Especificación de interfaces y contratos

### 3.1 Convenciones comunes

Según el SDD §7.1: base `/api/v1`, JSON `camelCase`, fechas ISO 8601 UTC, errores `application/problem+json` con `code` y `correlationId`, `X-Correlation-Id` y `X-App-Version` en cada petición. Los contratos se publican como OpenAPI 3.1 generado por springdoc-openapi; los fragmentos siguientes son la especificación de referencia.

**Esquemas comunes**

```yaml
openapi: 3.1.0
info:
  title: SafeRoute API
  version: 1.0.0
servers:
  - url: https://api.saferoute.example/api/v1   # dominio pendiente (PT-06)
  - url: http://10.0.2.2:8080/api/v1            # local desde el emulador
components:
  securitySchemes:
    bearerAuth: { type: http, scheme: bearer, bearerFormat: JWT }
  schemas:
    Problem:
      type: object
      required: [type, title, status, code, correlationId]
      properties:
        type: { type: string, format: uri, example: "https://api.saferoute.example/errors/VAL-001" }
        title: { type: string, example: "Validation failed" }
        status: { type: integer, example: 400 }
        detail: { type: string }
        code: { type: string, pattern: "^[A-Z]{3}-\\d{3}$", example: "VAL-001" }
        correlationId: { type: string, format: uuid }
        errors:
          type: array
          items:
            type: object
            required: [field, message]
            properties:
              field: { type: string, example: "password" }
              message: { type: string, example: "Debe tener al menos 8 caracteres con letras y números" }
    TokenPair:
      type: object
      required: [tokenType, accessToken, accessTokenExpiresAt, refreshToken]
      properties:
        tokenType: { type: string, const: Bearer }
        accessToken: { type: string }
        accessTokenExpiresAt: { type: string, format: date-time }
        refreshToken: { type: string }
    Guardian:
      type: object
      required: [guardianId, email, displayName, isEmailVerified]
      properties:
        guardianId: { type: string, format: uuid }
        email: { type: string, format: email }
        displayName: { type: string, maxLength: 50 }
        isEmailVerified: { type: boolean }
```

### 3.2 Autenticación de tutor y vinculación del menor

El tutor tiene cuenta (registro e inicio de sesión). El menor **no** tiene cuenta ni contraseña: su "inicio de sesión" es la vinculación del dispositivo con un código generado por el tutor (RF-015), que emite credenciales del principal `device`.

#### 3.2.1 `POST /api/v1/auth/register` — registro del tutor

```yaml
paths:
  /auth/register:
    post:
      tags: [auth]
      summary: Register a guardian with email and password
      operationId: registerGuardian
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password, displayName, acceptedPrivacyNoticeVersion]
              properties:
                email: { type: string, format: email, maxLength: 500 }
                password: { type: string, minLength: 8, maxLength: 128 }
                displayName: { type: string, minLength: 1, maxLength: 50 }
                phoneNumber: { type: string, pattern: "^\\+[1-9]\\d{7,14}$" }
                acceptedPrivacyNoticeVersion: { type: string }
      responses:
        "201":
          description: Guardian created; email verification pending
          content:
            application/json:
              schema:
                type: object
                properties:
                  guardian: { $ref: "#/components/schemas/Guardian" }
                  familyGroupId: { type: string, format: uuid }
                  tokens: { $ref: "#/components/schemas/TokenPair" }
        "400": { description: Invalid input (VAL-001), content: { application/problem+json: { schema: { $ref: "#/components/schemas/Problem" } } } }
        "409": { description: Email already registered (AUT-004), content: { application/problem+json: { schema: { $ref: "#/components/schemas/Problem" } } } }
```

Petición:

```http
POST /api/v1/auth/register
Content-Type: application/json
X-App-Version: 1.0.0
```

```json
{
  "email": "laura.martinez@example.com",
  "password": "Escuela2026",
  "displayName": "Laura",
  "phoneNumber": "+5215512345678",
  "acceptedPrivacyNoticeVersion": "2026-10"
}
```

Respuesta `201 Created`:

```json
{
  "guardian": {
    "guardianId": "0192f3a0-6b1c-7d2e-9f40-1a2b3c4d5e6f",
    "email": "laura.martinez@example.com",
    "displayName": "Laura",
    "isEmailVerified": false
  },
  "familyGroupId": "0192f3a0-6b1d-7a11-8c22-3d4e5f6a7b8c",
  "tokens": {
    "tokenType": "Bearer",
    "accessToken": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMTAifQ...",
    "accessTokenExpiresAt": "2026-10-22T17:15:00Z",
    "refreshToken": "rt_7Qm9b2Xc...c41"
  }
}
```

Respuesta `400` (validación):

```json
{
  "type": "https://api.saferoute.example/errors/VAL-001",
  "title": "Validation failed",
  "status": 400,
  "code": "VAL-001",
  "correlationId": "c4d19e77-2b8f-4a51-b7e3-90d64c2f8a15",
  "errors": [
    { "field": "password", "message": "Debe tener al menos 8 caracteres con letras y números" }
  ]
}
```

#### 3.2.2 `POST /api/v1/auth/login` — inicio de sesión del tutor

```yaml
  /auth/login:
    post:
      tags: [auth]
      summary: Authenticate a guardian
      operationId: loginGuardian
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [email, password]
              properties:
                email: { type: string, format: email }
                password: { type: string }
      responses:
        "200":
          description: Authenticated
          content:
            application/json:
              schema:
                type: object
                properties:
                  guardian: { $ref: "#/components/schemas/Guardian" }
                  tokens: { $ref: "#/components/schemas/TokenPair" }
        "401": { description: Invalid credentials (AUT-001) }
        "429": { description: Too many attempts (AUT-005) }
```

Petición:

```json
{ "email": "laura.martinez@example.com", "password": "Escuela2026" }
```

Respuesta `200 OK`:

```json
{
  "guardian": {
    "guardianId": "0192f3a0-6b1c-7d2e-9f40-1a2b3c4d5e6f",
    "email": "laura.martinez@example.com",
    "displayName": "Laura",
    "isEmailVerified": true
  },
  "tokens": {
    "tokenType": "Bearer",
    "accessToken": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMTAifQ...",
    "accessTokenExpiresAt": "2026-10-22T17:30:00Z",
    "refreshToken": "rt_Pz81kLq...9ad"
  }
}
```

Respuesta `401 Unauthorized` (el mensaje no distingue si el correo existe):

```json
{
  "type": "https://api.saferoute.example/errors/AUT-001",
  "title": "Invalid credentials",
  "status": 401,
  "code": "AUT-001",
  "correlationId": "5e2c7d90-1f4a-4c6b-8d23-7a9b0e3f6c48"
}
```

El acceso con Google (`POST /api/v1/auth/google`, cuerpo `{ "idToken": "..." }`) devuelve la misma respuesta que el login.

#### 3.2.3 `POST /api/v1/device/pair` — vinculación ("login") del dispositivo del menor

```yaml
  /device/pair:
    post:
      tags: [device]
      summary: Redeem a pairing code and obtain device credentials
      operationId: pairDevice
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [pairingCode, device]
              properties:
                pairingCode: { type: string, pattern: "^\\d{6}$" }
                device:
                  type: object
                  required: [manufacturer, model, osVersion, appVersion, hasGooglePlayServices]
                  properties:
                    manufacturer: { type: string, maxLength: 50 }
                    model: { type: string, maxLength: 50 }
                    osVersion: { type: string, maxLength: 50 }
                    appVersion: { type: string, maxLength: 50 }
                    pushToken: { type: string, maxLength: 500 }
                    hasGooglePlayServices: { type: boolean }
      responses:
        "200":
          description: Device paired
          content:
            application/json:
              schema:
                type: object
                properties:
                  deviceId: { type: string, format: uuid }
                  child:
                    type: object
                    properties:
                      childId: { type: string, format: uuid }
                      displayName: { type: string }
                  tokens: { $ref: "#/components/schemas/TokenPair" }
                  geofenceConfigVersion: { type: integer }
        "400": { description: Invalid or expired code (DEV-001) }
        "422": { description: Device without Google Play Services (DEV-004) }
        "429": { description: Too many attempts (DEV-003) }
```

Petición (el código llega por QR o se escribe):

```json
{
  "pairingCode": "482913",
  "device": {
    "manufacturer": "motorola",
    "model": "moto g54",
    "osVersion": "14",
    "appVersion": "1.0.0",
    "pushToken": "fcm_dX3k...Q9",
    "hasGooglePlayServices": true
  }
}
```

Respuesta `200 OK`:

```json
{
  "deviceId": "0192f3b2-11aa-7bcd-8e01-23456789abcd",
  "child": {
    "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7",
    "displayName": "Sofía"
  },
  "tokens": {
    "tokenType": "Bearer",
    "accessToken": "eyJhbGciOiJFUzI1NiIsImtpZCI6IjIwMjYtMTAifQ...",
    "accessTokenExpiresAt": "2026-10-22T17:30:00Z",
    "refreshToken": "drt_Lm4Q...7e"
  },
  "geofenceConfigVersion": 3
}
```

Tras la vinculación, el dispositivo descarga su configuración con `GET /api/v1/device/geofences` y renueva sus tokens con `POST /api/v1/auth/refresh`.

### 3.3 `POST /api/v1/family-groups/{familyGroupId}/destinations` — creación de geocerca

La geocerca pertenece a un grupo familiar; por eso la ruta validada en el SDD incluye el grupo.

```yaml
  /family-groups/{familyGroupId}/destinations:
    post:
      tags: [destinations]
      summary: Create a destination (circular geofence) with schedules
      operationId: createDestination
      security: [{ bearerAuth: [] }]
      parameters:
        - { name: familyGroupId, in: path, required: true, schema: { type: string, format: uuid } }
        - { name: Idempotency-Key, in: header, required: false, schema: { type: string, format: uuid } }
      requestBody:
        required: true
        content:
          application/json:
            schema: { $ref: "#/components/schemas/DestinationRequest" }
      responses:
        "201":
          description: Destination created
          headers:
            Location: { schema: { type: string } }
          content:
            application/json:
              schema: { $ref: "#/components/schemas/Destination" }
        "400": { description: Invalid input (VAL-001) }
        "403": { description: Not a member of the family group (FAM-004) }
        "422": { description: Business rule violated (DST-002, DST-003) }
components:
  schemas:
    DestinationRequest:
      type: object
      required: [name, icon, latitude, longitude, radiusMeters, timeZoneId, childIds, schedules]
      properties:
        name: { type: string, minLength: 1, maxLength: 50 }
        icon: { type: string, enum: [school, home, sports, music, family, other] }
        address: { type: string, maxLength: 500 }
        latitude: { type: number, minimum: -90, maximum: 90 }
        longitude: { type: number, minimum: -180, maximum: 180 }
        radiusMeters: { type: integer, minimum: 100, maximum: 1000, default: 150 }
        timeZoneId: { type: string, example: America/Mexico_City }
        childIds: { type: array, minItems: 1, items: { type: string, format: uuid } }
        schedules:
          type: array
          minItems: 1
          items: { $ref: "#/components/schemas/Schedule" }
    Schedule:
      type: object
      required: [recurrenceType, windowStartTime, windowEndTime]
      properties:
        recurrenceType: { type: string, enum: [once, weekly] }
        specificDate: { type: string, format: date, description: "Required when recurrenceType = once" }
        daysOfWeek:
          type: array
          description: "Required when recurrenceType = weekly"
          items: { type: string, enum: [MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY, SATURDAY, SUNDAY] }
        windowStartTime: { type: string, pattern: "^([01]\\d|2[0-3]):[0-5]\\d$" }
        windowEndTime: { type: string, pattern: "^([01]\\d|2[0-3]):[0-5]\\d$" }
        expectedArrivalTime: { type: string, pattern: "^([01]\\d|2[0-3]):[0-5]\\d$" }
        toleranceMinutes: { type: integer, minimum: 5, maximum: 60, default: 15 }
    Destination:
      allOf:
        - $ref: "#/components/schemas/DestinationRequest"
        - type: object
          properties:
            destinationId: { type: string, format: uuid }
            familyGroupId: { type: string, format: uuid }
            status: { type: string, enum: [active, paused] }
            configVersion: { type: integer }
            syncStatus:
              type: array
              items:
                type: object
                properties:
                  childId: { type: string, format: uuid }
                  status: { type: string, enum: [pending, synced] }
            createdAt: { type: string, format: date-time }
```

Petición:

```http
POST /api/v1/family-groups/0192f3a0-6b1d-7a11-8c22-3d4e5f6a7b8c/destinations
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
Content-Type: application/json
Idempotency-Key: 0192f3c0-7d1e-7a8b-9c2d-1e3f4a5b6c7d
```

```json
{
  "name": "Escuela",
  "icon": "school",
  "address": "Av. Insurgentes Sur 1234, Ciudad de México",
  "latitude": 19.371234,
  "longitude": -99.178456,
  "radiusMeters": 150,
  "timeZoneId": "America/Mexico_City",
  "childIds": ["0192f3a8-0c4d-7e5f-a061-728394a5b6c7"],
  "schedules": [
    {
      "recurrenceType": "weekly",
      "daysOfWeek": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"],
      "windowStartTime": "07:00",
      "windowEndTime": "14:00",
      "expectedArrivalTime": "07:30",
      "toleranceMinutes": 15
    }
  ]
}
```

Respuesta `201 Created`:

```json
{
  "destinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c",
  "familyGroupId": "0192f3a0-6b1d-7a11-8c22-3d4e5f6a7b8c",
  "name": "Escuela",
  "icon": "school",
  "address": "Av. Insurgentes Sur 1234, Ciudad de México",
  "latitude": 19.371234,
  "longitude": -99.178456,
  "radiusMeters": 150,
  "timeZoneId": "America/Mexico_City",
  "childIds": ["0192f3a8-0c4d-7e5f-a061-728394a5b6c7"],
  "schedules": [
    {
      "recurrenceType": "weekly",
      "daysOfWeek": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"],
      "windowStartTime": "07:00",
      "windowEndTime": "14:00",
      "expectedArrivalTime": "07:30",
      "toleranceMinutes": 15
    }
  ],
  "status": "active",
  "configVersion": 1,
  "syncStatus": [
    { "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7", "status": "pending" }
  ],
  "createdAt": "2026-10-22T16:58:12Z"
}
```

Respuesta `422` (regla de negocio):

```json
{
  "type": "https://api.saferoute.example/errors/DST-003",
  "title": "Expected arrival time is outside the watch window",
  "status": 422,
  "code": "DST-003",
  "correlationId": "8f3a91c2-6d2e-4b3a-9c01-2f5d8e7a1b4c"
}
```

### 3.4 `GET /api/v1/locations/history/{childId}` — historial de ubicaciones puntuales

Devuelve las ubicaciones asociadas a **eventos** (llegada, salida, reingreso, no llegada) y a **consultas bajo demanda** de los últimos 30 días. No incluye las lecturas de verificación periódica ni reconstruye trayectos: SafeRoute no hace rastreo continuo (W-01, RNF-PRI-04). Cada consulta del historial queda registrada como acceso en el log de auditoría.

```yaml
  /locations/history/{childId}:
    get:
      tags: [locations]
      summary: List point locations linked to events and on-demand queries (max 30 days)
      operationId: getLocationHistory
      security: [{ bearerAuth: [] }]
      parameters:
        - { name: childId, in: path, required: true, schema: { type: string, format: uuid } }
        - { name: from, in: query, required: false, schema: { type: string, format: date-time }, description: "Default: now - 7 days. Cannot be earlier than now - 30 days" }
        - { name: to, in: query, required: false, schema: { type: string, format: date-time }, description: "Default: now" }
        - { name: source, in: query, required: false, schema: { type: string, enum: [event, on_demand] } }
        - { name: limit, in: query, required: false, schema: { type: integer, minimum: 1, maximum: 200, default: 50 } }
        - { name: cursor, in: query, required: false, schema: { type: string } }
      responses:
        "200":
          description: Location history page
          content:
            application/json:
              schema:
                type: object
                required: [childId, items]
                properties:
                  childId: { type: string, format: uuid }
                  from: { type: string, format: date-time }
                  to: { type: string, format: date-time }
                  items:
                    type: array
                    items: { $ref: "#/components/schemas/LocationPoint" }
                  nextCursor: { type: [string, "null"] }
        "400": { description: Range exceeds retention (VAL-001) }
        "403": { description: Not a guardian of the child (FAM-004) }
        "404": { description: Child not found (CHD-001) }
components:
  schemas:
    LocationPoint:
      type: object
      required: [locationSampleId, capturedAt, latitude, longitude, accuracyMeters, source]
      properties:
        locationSampleId: { type: string, format: uuid }
        capturedAt: { type: string, format: date-time }
        latitude: { type: number }
        longitude: { type: number }
        accuracyMeters: { type: number }
        source: { type: string, enum: [event, on_demand] }
        isMock: { type: boolean }
        eventType: { type: [string, "null"], enum: [ARRIVAL, LATE_ARRIVAL, DEPARTURE, RETURN, NO_ARRIVAL, LOCATION_QUERY, null] }
        safetyEventId: { type: [string, "null"], format: uuid }
        destinationId: { type: [string, "null"], format: uuid }
        destinationName: { type: [string, "null"] }
```

Petición:

```http
GET /api/v1/locations/history/0192f3a8-0c4d-7e5f-a061-728394a5b6c7?from=2026-10-21T00:00:00Z&to=2026-10-22T00:00:00Z&limit=50
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
```

Respuesta `200 OK`:

```json
{
  "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7",
  "from": "2026-10-21T00:00:00Z",
  "to": "2026-10-22T00:00:00Z",
  "items": [
    {
      "locationSampleId": "0192ee10-9a01-7b22-8c33-44d55e66f770",
      "capturedAt": "2026-10-21T19:04:31Z",
      "latitude": 19.374410,
      "longitude": -99.178012,
      "accuracyMeters": 14.0,
      "source": "event",
      "isMock": false,
      "eventType": "DEPARTURE",
      "safetyEventId": "0192ee10-9a7f-7c01-9d12-3e4f5a6b7c8d",
      "destinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c",
      "destinationName": "Escuela"
    },
    {
      "locationSampleId": "0192ec41-3b02-7d44-8e55-66f778899aab",
      "capturedAt": "2026-10-21T13:21:44Z",
      "latitude": 19.371301,
      "longitude": -99.178390,
      "accuracyMeters": 18.5,
      "source": "event",
      "isMock": false,
      "eventType": "ARRIVAL",
      "safetyEventId": "0192ec41-3b9e-7f00-a111-2b3c4d5e6f70",
      "destinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c",
      "destinationName": "Escuela"
    }
  ],
  "nextCursor": null
}
```

### 3.5 Protocolo WebSocket

#### 3.5.1 Alcance

- **Solo la app del tutor** se conecta, y solo mientras está en primer plano. Al pasar a segundo plano cierra la conexión con `1000` y los avisos siguen llegando por FCM.
- La app del menor **no** usa WebSocket (§1.3.1).
- El canal es de conveniencia: todo lo que se emite por WebSocket también queda en la bandeja y el historial.

#### 3.5.2 Conexión y autenticación

```http
GET /ws/v1 HTTP/1.1
Host: api.saferoute.example
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Version: 13
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Protocol: saferoute.v1
Authorization: Bearer eyJhbGciOiJFUzI1NiIs...
X-App-Version: 1.0.0
```

- El JWT de acceso del tutor se envía en el encabezado `Authorization` del handshake (OkHttp lo permite); nunca en la URL, para que no quede en logs del proxy.
- Un JWT inválido rechaza el handshake con `401`. Un token de dispositivo (`scope = device`) se rechaza con `403`.
- Antes de que caduque el token (15 min), la app envía `auth:refresh` con el nuevo token. Si caduca sin renovarse, el servidor cierra con `4401`.
- Implementación: `TextWebSocketHandler` de Spring WebSocket en `saferoute-api`, sin STOMP; Caddy hace proxy de la ruta `/ws/v1` con *keep-alive*.

#### 3.5.3 Sobre de mensaje

Todos los mensajes son texto JSON UTF-8 con esta estructura:

```json
{
  "v": 1,
  "type": "geofence:status",
  "id": "0192f400-1111-7aaa-8bbb-ccccdddd0001",
  "ts": "2026-10-22T13:21:46.120Z",
  "replyTo": null,
  "data": { }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `v` | integer | Versión del protocolo (`1`) |
| `type` | string | Tipo de evento `dominio:acción` |
| `id` | uuid | Identificador único del mensaje (UUID v7) |
| `ts` | date-time | Hora de emisión en UTC |
| `replyTo` | uuid \| null | `id` del mensaje del cliente al que responde |
| `data` | object | Carga útil según el tipo |

Esquema JSON del sobre:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://api.saferoute.example/schemas/ws/envelope.json",
  "type": "object",
  "required": ["v", "type", "id", "ts", "data"],
  "properties": {
    "v": { "const": 1 },
    "type": { "type": "string", "pattern": "^[a-z]+:[a-z_]+$" },
    "id": { "type": "string", "format": "uuid" },
    "ts": { "type": "string", "format": "date-time" },
    "replyTo": { "type": ["string", "null"], "format": "uuid" },
    "data": { "type": "object" }
  },
  "additionalProperties": false
}
```

#### 3.5.4 Eventos del cliente al servidor

| Tipo | Propósito | `data` | Respuesta |
|---|---|---|---|
| `connection:ping` | Latido de aplicación cada 25 s | `{}` | `connection:pong` |
| `child:subscribe` | Recibir eventos de uno o varios menores | `{ "childIds": ["uuid"] }` | `child:subscribed` |
| `child:unsubscribe` | Dejar de recibirlos | `{ "childIds": ["uuid"] }` | `child:unsubscribed` |
| `location:request` | Pedir la ubicación actual (equivale a `POST /children/{id}/location-requests`, RF-050) | `{ "childId": "uuid" }` | `location:requested`, y después `location:ping` |
| `sync:request` | Recuperar eventos perdidos tras una reconexión | `{ "since": "<cursor>" }` | `sync:events` |
| `auth:refresh` | Renovar el token de acceso de la conexión | `{ "accessToken": "jwt" }` | `auth:refreshed` |

Ejemplos:

```json
{ "v": 1, "type": "child:subscribe", "id": "0192f401-0001-7000-8000-000000000001", "ts": "2026-10-22T13:20:00.000Z",
  "data": { "childIds": ["0192f3a8-0c4d-7e5f-a061-728394a5b6c7"] } }
```

```json
{ "v": 1, "type": "location:request", "id": "0192f401-0002-7000-8000-000000000002", "ts": "2026-10-22T13:42:00.000Z",
  "data": { "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7" } }
```

```json
{ "v": 1, "type": "sync:request", "id": "0192f401-0003-7000-8000-000000000003", "ts": "2026-10-22T13:50:02.000Z",
  "data": { "since": "c_0192ec41-3b9e-7f00-a111-2b3c4d5e6f70" } }
```

#### 3.5.5 Eventos del servidor al cliente

| Tipo | Cuándo | Relación con el dominio |
|---|---|---|
| `connection:ready` | Tras el handshake | — |
| `connection:pong` | Respuesta al latido | — |
| `child:subscribed` / `child:unsubscribed` | Confirmación de suscripción | Valida la pertenencia del tutor al grupo de cada menor |
| `geofence:status` | Cambio de estado de una sesión de vigilancia | Transiciones de `watch_session` (SDD §4.5) |
| `location:ping` | Llega una ubicación del menor solicitada por un tutor o asociada a un evento | `location_sample` con `source` `on_demand` o `event` |
| `location:requested` | Solicitud de ubicación aceptada | `location_request` creada |
| `alert:arrival` | Llegada o llegada tardía | `safety_event` `ARRIVAL`, `LATE_ARRIVAL`, `RETURN` |
| `alert:movement_detected` | Salida del destino tras la llegada | `safety_event` `DEPARTURE` |
| `alert:no_arrival` | El menor no llegó a tiempo | `safety_event` `NO_ARRIVAL` |
| `alert:device` | Estado del dispositivo | `PERMISSION_REVOKED`, `DEVICE_OFFLINE`, `DEVICE_RECONNECTED`, `BATTERY_LOW` |
| `sync:events` | Respuesta a `sync:request` | Eventos de seguridad posteriores al cursor |
| `auth:refreshed` | Token renovado | — |
| `error` | Mensaje inválido o no autorizado | Códigos del catálogo |

Las lecturas de la verificación periódica **no** generan `location:ping`: solo actualizan el estado si provocan una transición (`geofence:status`). Es la misma regla de privacidad que en el historial (RF-046).

**`connection:ready`**

```json
{
  "v": 1, "type": "connection:ready", "id": "0192f400-0000-7aaa-8bbb-000000000001",
  "ts": "2026-10-22T13:20:00.050Z", "replyTo": null,
  "data": {
    "connectionId": "0192f400-0000-7aaa-8bbb-00000000c0de",
    "guardianId": "0192f3a0-6b1c-7d2e-9f40-1a2b3c4d5e6f",
    "heartbeatIntervalSeconds": 25,
    "idleTimeoutSeconds": 60,
    "serverTime": "2026-10-22T13:20:00.050Z"
  }
}
```

**`geofence:status`**

```json
{
  "v": 1, "type": "geofence:status", "id": "0192f400-1111-7aaa-8bbb-ccccdddd0001",
  "ts": "2026-10-22T13:21:46.120Z", "replyTo": null,
  "data": {
    "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7",
    "watchSessionId": "0192f3d0-5a6b-7c8d-9e0f-102132435465",
    "destinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c",
    "previousStatus": "awaiting",
    "status": "present",
    "changedAt": "2026-10-22T13:21:44Z",
    "cause": "geofence_dwell",
    "windowEndsAt": "2026-10-22T20:00:00Z",
    "cursor": "c_0192f3d0-77aa-7bbb-8ccc-ddddeeee0001"
  }
}
```

| Campo | Valores |
|---|---|
| `status`, `previousStatus` | `scheduled`, `awaiting`, `overdue`, `present`, `away`, `closed`, `cancelled` |
| `cause` | `window_opened`, `geofence_dwell`, `geofence_exit`, `verification`, `arrival_deadline`, `window_closed`, `destination_changed` |

**`location:ping`**

```json
{
  "v": 1, "type": "location:ping", "id": "0192f400-2222-7aaa-8bbb-ccccdddd0002",
  "ts": "2026-10-22T13:42:09.310Z", "replyTo": "0192f401-0002-7000-8000-000000000002",
  "data": {
    "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7",
    "locationRequestId": "0192f402-aaaa-7bbb-8ccc-dddd00000001",
    "locationSampleId": "0192f402-bbbb-7ccc-8ddd-eeee00000001",
    "source": "on_demand",
    "latitude": 19.371190,
    "longitude": -99.178501,
    "accuracyMeters": 12.0,
    "capturedAt": "2026-10-22T13:42:07Z",
    "isMock": false,
    "isFresh": true,
    "insideDestinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c"
  }
}
```

**`alert:movement_detected`**

```json
{
  "v": 1, "type": "alert:movement_detected", "id": "0192f400-3333-7aaa-8bbb-ccccdddd0003",
  "ts": "2026-10-22T17:04:33.020Z", "replyTo": null,
  "data": {
    "safetyEventId": "0192ee10-9a7f-7c01-9d12-3e4f5a6b7c8d",
    "eventType": "DEPARTURE",
    "priority": "high",
    "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7",
    "watchSessionId": "0192f3d0-5a6b-7c8d-9e0f-102132435465",
    "destinationId": "0192f3c1-2e3f-7a40-b152-6374859a0b1c",
    "occurredAt": "2026-10-22T17:04:31Z",
    "detectedBy": "geofence_exit",
    "location": {
      "latitude": 19.374410,
      "longitude": -99.178012,
      "accuracyMeters": 14.0,
      "isMock": false
    },
    "cursor": "c_0192ee10-9a7f-7c01-9d12-3e4f5a6b7c8d"
  }
}
```

`detectedBy` vale `geofence_exit` (geocerca del dispositivo) o `verification` (ciclo de 20 min, SDD §9).

**`alert:arrival`**, **`alert:no_arrival`** y **`alert:device`** usan la misma forma que `alert:movement_detected`, con `eventType` según la tabla anterior. En `alert:no_arrival`, `location` es la última ubicación conocida y se agrega `lastSeenAt`; en `alert:device` se agregan `batteryLevel` o `locationPermission` cuando aplican, y `location` es `null`.

**`sync:events`**

```json
{
  "v": 1, "type": "sync:events", "id": "0192f400-4444-7aaa-8bbb-ccccdddd0004",
  "ts": "2026-10-22T13:50:02.200Z", "replyTo": "0192f401-0003-7000-8000-000000000003",
  "data": {
    "events": [
      { "type": "alert:arrival", "data": { "safetyEventId": "0192f3d0-77aa-7bbb-8ccc-ddddeeee0001", "eventType": "ARRIVAL", "childId": "0192f3a8-0c4d-7e5f-a061-728394a5b6c7", "occurredAt": "2026-10-22T13:21:44Z", "cursor": "c_0192f3d0-77aa-7bbb-8ccc-ddddeeee0001" } }
    ],
    "nextCursor": "c_0192f3d0-77aa-7bbb-8ccc-ddddeeee0001",
    "hasMore": false
  }
}
```

**`error`**

```json
{
  "v": 1, "type": "error", "id": "0192f400-5555-7aaa-8bbb-ccccdddd0005",
  "ts": "2026-10-22T13:20:01.000Z", "replyTo": "0192f401-0001-7000-8000-000000000001",
  "data": {
    "code": "FAM-004",
    "title": "Not a guardian of the child",
    "correlationId": "1f2e3d4c-5b6a-4978-8a9b-0c1d2e3f4a5b"
  }
}
```

#### 3.5.6 Reconexión y entrega

- El cursor (`c_<safetyEventId>`) es el identificador UUID v7 del último evento recibido; por ser ordenable en el tiempo, `sync:request` devuelve los eventos con identificador mayor, en orden, en páginas de 100.
- Reconexión con espera exponencial y variación aleatoria: 1, 2, 4, 8, 16 y 30 s como máximo.
- Tras reconectar, la app envía `child:subscribe` y `sync:request` con su último cursor.
- La entrega es "al menos una vez": la app descarta duplicados por `safetyEventId`.
- Con más de 24 h sin conexión, la app no usa `sync:request` y recarga el historial por REST.

#### 3.5.7 Códigos de cierre

| Código | Significado |
|---|---|
| `1000` | Cierre normal (app a segundo plano, cierre de sesión) |
| `1001` | El servidor se apaga (despliegue); el cliente reconecta |
| `1012` | Reinicio del servicio; el cliente reconecta |
| `4400` | Mensaje inválido (no cumple el esquema) |
| `4401` | Token caducado o inválido; el cliente renueva y reconecta |
| `4403` | Acceso revocado (tutor eliminado del grupo) |
| `4408` | Sin latido en 60 s |
| `4429` | Límite de mensajes superado (más de 30 por minuto) |

#### 3.5.8 Distribución entre instancias

```mermaid
sequenceDiagram
    participant W as saferoute-worker
    participant DB as PostgreSQL
    participant A1 as saferoute-api #1
    participant A2 as saferoute-api #2
    participant T as App Tutor (conectada a #2)

    W->>DB: COMMIT safety_event DEPARTURE
    W->>DB: NOTIFY saferoute_realtime, '{"kind":"safety_event","familyGroupId":"…","childId":"…","refId":"…"}'
    DB-->>A1: Notificación (sin suscriptores de ese menor → ignora)
    DB-->>A2: Notificación
    A2->>DB: Carga safety_event por refId
    A2->>T: alert:movement_detected
```

El `NOTIFY` se emite dentro de la misma transacción que el evento, así que solo se entrega si la transacción se confirma. Si una instancia pierde la conexión `LISTEN`, la reabre y los clientes recuperan lo perdido con `sync:request`.

---

## 4. Modelo de datos detallado

### 4.1 Correspondencia con la solicitud

| Entidad solicitada | Implementación en PostgreSQL | Motivo |
|---|---|---|
| `User` con roles `TUTOR`, `CHILD` | `guardian` (rol TUTOR), `child` + `device` (rol CHILD) | El tutor tiene credenciales y el menor no; separar las tablas evita columnas nulas y aplicar por error políticas de un rol al otro. El rol viaja como `scope` del JWT (`guardian` o `device`). |
| `Destination` con índice `2dsphere` | `destination` + `destination_schedule` + `destination_child` | Sin índice espacial (§4.4). |
| `LocationLog` con índice TTL | `location_sample`, particionada por mes | Purga por partición (§4.5). |
| `Alert` | `safety_event` (+ `cat_event_type`, `notification`) | Llegadas, salidas ("movimiento indebido"), no llegadas y alertas de dispositivo. |

Convenciones (SDD §6.1): `snake_case` singular, `uuid` como clave, `created_at`, `updated_at`, `is_active` en toda tabla, restricciones con nombre. Los identificadores son **UUID v7 generados en la aplicación** (`java-uuid-generator`), con `uuidv7()` como valor por defecto de respaldo en la base.

### 4.2 DDL (Flyway)

`V1__identity_and_family.sql`

```sql
CREATE TABLE guardian (
    guardian_id            uuid         NOT NULL DEFAULT uuidv7(),
    email                  varchar(500) NOT NULL,
    password_hash          varchar(500),
    display_name           varchar(50)  NOT NULL,
    phone_number           varchar(50),
    photo_url              varchar(500),
    is_email_verified      boolean      NOT NULL DEFAULT false,
    deletion_requested_at  timestamptz,
    created_at             timestamptz  NOT NULL DEFAULT now(),
    updated_at             timestamptz  NOT NULL DEFAULT now(),
    is_active              boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_guardian PRIMARY KEY (guardian_id),
    CONSTRAINT uq_guardian_email UNIQUE (email),
    CONSTRAINT ck_guardian_email_lowercase CHECK (email = lower(email)),
    CONSTRAINT ck_guardian_phone_number_e164 CHECK (phone_number IS NULL OR phone_number ~ '^\+[1-9][0-9]{7,14}$')
);

CREATE TABLE family_group (
    family_group_id    uuid        NOT NULL DEFAULT uuidv7(),
    name               varchar(50) NOT NULL,
    owner_guardian_id  uuid        NOT NULL,
    created_at         timestamptz NOT NULL DEFAULT now(),
    updated_at         timestamptz NOT NULL DEFAULT now(),
    is_active          boolean     NOT NULL DEFAULT true,
    CONSTRAINT pk_family_group PRIMARY KEY (family_group_id),
    CONSTRAINT fk_family_group_guardian_owner_guardian_id
        FOREIGN KEY (owner_guardian_id) REFERENCES guardian (guardian_id)
);

CREATE TABLE family_membership (
    family_membership_id  uuid        NOT NULL DEFAULT uuidv7(),
    family_group_id       uuid        NOT NULL,
    guardian_id           uuid        NOT NULL,
    role                  varchar(50) NOT NULL,
    created_at            timestamptz NOT NULL DEFAULT now(),
    updated_at            timestamptz NOT NULL DEFAULT now(),
    is_active             boolean     NOT NULL DEFAULT true,
    CONSTRAINT pk_family_membership PRIMARY KEY (family_membership_id),
    CONSTRAINT fk_family_membership_family_group_family_group_id
        FOREIGN KEY (family_group_id) REFERENCES family_group (family_group_id) ON DELETE CASCADE,
    CONSTRAINT fk_family_membership_guardian_guardian_id
        FOREIGN KEY (guardian_id) REFERENCES guardian (guardian_id) ON DELETE CASCADE,
    CONSTRAINT uq_family_membership_family_group_id_guardian_id UNIQUE (family_group_id, guardian_id),
    CONSTRAINT ck_family_membership_role CHECK (role IN ('owner', 'member'))
);

CREATE TABLE child (
    child_id         uuid         NOT NULL DEFAULT uuidv7(),
    family_group_id  uuid         NOT NULL,
    display_name     varchar(50)  NOT NULL,
    birth_year       smallint,
    photo_url        varchar(500),
    created_at       timestamptz  NOT NULL DEFAULT now(),
    updated_at       timestamptz  NOT NULL DEFAULT now(),
    is_active        boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_child PRIMARY KEY (child_id),
    CONSTRAINT fk_child_family_group_family_group_id
        FOREIGN KEY (family_group_id) REFERENCES family_group (family_group_id) ON DELETE CASCADE,
    CONSTRAINT ck_child_birth_year CHECK (birth_year IS NULL OR birth_year BETWEEN 2000 AND 2100)
);

CREATE INDEX ix_child_family_group_id ON child (family_group_id);

CREATE TABLE device (
    device_id                       uuid         NOT NULL DEFAULT uuidv7(),
    child_id                        uuid         NOT NULL,
    manufacturer                    varchar(50)  NOT NULL,
    model                           varchar(50)  NOT NULL,
    os_version                      varchar(50)  NOT NULL,
    app_version                     varchar(50)  NOT NULL,
    push_token                      varchar(500),
    location_permission             varchar(50)  NOT NULL,
    is_precise_location             boolean      NOT NULL DEFAULT false,
    is_battery_optimization_exempt  boolean      NOT NULL DEFAULT false,
    pairing_status                  varchar(50)  NOT NULL,
    battery_level                   smallint,
    applied_config_version          integer      NOT NULL DEFAULT 0,
    last_seen_at                    timestamptz,
    paired_at                       timestamptz  NOT NULL DEFAULT now(),
    unpaired_at                     timestamptz,
    created_at                      timestamptz  NOT NULL DEFAULT now(),
    updated_at                      timestamptz  NOT NULL DEFAULT now(),
    is_active                       boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_device PRIMARY KEY (device_id),
    CONSTRAINT fk_device_child_child_id
        FOREIGN KEY (child_id) REFERENCES child (child_id) ON DELETE CASCADE,
    CONSTRAINT ck_device_location_permission CHECK (location_permission IN ('always', 'while_in_use', 'denied')),
    CONSTRAINT ck_device_pairing_status CHECK (pairing_status IN ('complete', 'incomplete')),
    CONSTRAINT ck_device_battery_level CHECK (battery_level IS NULL OR battery_level BETWEEN 0 AND 100)
);

-- RF-016: como máximo un dispositivo activo por menor.
CREATE UNIQUE INDEX uq_device_child_id_active ON device (child_id) WHERE is_active;
```

`V2__destinations.sql`

```sql
CREATE TABLE destination (
    destination_id   uuid          NOT NULL DEFAULT uuidv7(),
    family_group_id  uuid          NOT NULL,
    name             varchar(50)   NOT NULL,
    icon             varchar(50)   NOT NULL,
    address          varchar(500),
    latitude         numeric(9,6)  NOT NULL,
    longitude        numeric(9,6)  NOT NULL,
    radius_meters    integer       NOT NULL DEFAULT 150,
    time_zone_id     varchar(50)   NOT NULL,
    status           varchar(50)   NOT NULL DEFAULT 'active',
    config_version   integer       NOT NULL DEFAULT 1,
    created_at       timestamptz   NOT NULL DEFAULT now(),
    updated_at       timestamptz   NOT NULL DEFAULT now(),
    is_active        boolean       NOT NULL DEFAULT true,
    version          bigint        NOT NULL DEFAULT 0,
    CONSTRAINT pk_destination PRIMARY KEY (destination_id),
    CONSTRAINT fk_destination_family_group_family_group_id
        FOREIGN KEY (family_group_id) REFERENCES family_group (family_group_id) ON DELETE CASCADE,
    CONSTRAINT ck_destination_latitude CHECK (latitude BETWEEN -90 AND 90),
    CONSTRAINT ck_destination_longitude CHECK (longitude BETWEEN -180 AND 180),
    CONSTRAINT ck_destination_radius_meters CHECK (radius_meters BETWEEN 100 AND 1000),
    CONSTRAINT ck_destination_status CHECK (status IN ('active', 'paused'))
);

CREATE INDEX ix_destination_family_group_id ON destination (family_group_id) WHERE is_active;

CREATE TABLE destination_child (
    destination_child_id  uuid        NOT NULL DEFAULT uuidv7(),
    destination_id        uuid        NOT NULL,
    child_id              uuid        NOT NULL,
    created_at            timestamptz NOT NULL DEFAULT now(),
    updated_at            timestamptz NOT NULL DEFAULT now(),
    is_active             boolean     NOT NULL DEFAULT true,
    CONSTRAINT pk_destination_child PRIMARY KEY (destination_child_id),
    CONSTRAINT fk_destination_child_destination_destination_id
        FOREIGN KEY (destination_id) REFERENCES destination (destination_id) ON DELETE CASCADE,
    CONSTRAINT fk_destination_child_child_child_id
        FOREIGN KEY (child_id) REFERENCES child (child_id) ON DELETE CASCADE,
    CONSTRAINT uq_destination_child_destination_id_child_id UNIQUE (destination_id, child_id)
);

CREATE INDEX ix_destination_child_child_id ON destination_child (child_id) WHERE is_active;

CREATE TABLE destination_schedule (
    destination_schedule_id  uuid         NOT NULL DEFAULT uuidv7(),
    destination_id           uuid         NOT NULL,
    recurrence_type          varchar(50)  NOT NULL,
    specific_date            date,
    days_of_week             smallint[],
    window_start_time        time         NOT NULL,
    window_end_time          time         NOT NULL,
    expected_arrival_time    time,
    tolerance_minutes        smallint     NOT NULL DEFAULT 15,
    created_at               timestamptz  NOT NULL DEFAULT now(),
    updated_at               timestamptz  NOT NULL DEFAULT now(),
    is_active                boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_destination_schedule PRIMARY KEY (destination_schedule_id),
    CONSTRAINT fk_destination_schedule_destination_destination_id
        FOREIGN KEY (destination_id) REFERENCES destination (destination_id) ON DELETE CASCADE,
    CONSTRAINT ck_destination_schedule_recurrence CHECK (
        (recurrence_type = 'once'   AND specific_date IS NOT NULL AND days_of_week IS NULL) OR
        (recurrence_type = 'weekly' AND specific_date IS NULL AND cardinality(days_of_week) BETWEEN 1 AND 7
             AND days_of_week <@ ARRAY[1,2,3,4,5,6,7]::smallint[])
    ),
    CONSTRAINT ck_destination_schedule_window_order CHECK (window_end_time > window_start_time),
    CONSTRAINT ck_destination_schedule_expected_arrival_in_window CHECK (
        expected_arrival_time IS NULL OR expected_arrival_time BETWEEN window_start_time AND window_end_time
    ),
    CONSTRAINT ck_destination_schedule_tolerance_minutes CHECK (tolerance_minutes BETWEEN 5 AND 60)
);
```

`V3__events_and_locations.sql`

```sql
CREATE TABLE cat_event_type (
    event_type_id           uuid         NOT NULL DEFAULT uuidv7(),
    code                    varchar(50)  NOT NULL,
    description             varchar(500) NOT NULL,
    is_priority_by_default  boolean      NOT NULL DEFAULT false,
    created_at              timestamptz  NOT NULL DEFAULT now(),
    updated_at              timestamptz  NOT NULL DEFAULT now(),
    is_active               boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_cat_event_type PRIMARY KEY (event_type_id),
    CONSTRAINT uq_cat_event_type_code UNIQUE (code)
);

-- Ubicaciones puntuales. Particionada por mes: la purga de 30 días elimina particiones completas.
CREATE TABLE location_sample (
    location_sample_id   uuid          NOT NULL DEFAULT uuidv7(),
    device_id            uuid          NOT NULL,
    child_id             uuid          NOT NULL,
    location_request_id  uuid,
    source               varchar(50)   NOT NULL,
    captured_at          timestamptz   NOT NULL,
    latitude             numeric(9,6)  NOT NULL,
    longitude            numeric(9,6)  NOT NULL,
    accuracy_meters      numeric(7,1)  NOT NULL,
    is_mock              boolean       NOT NULL DEFAULT false,
    created_at           timestamptz   NOT NULL DEFAULT now(),
    updated_at           timestamptz   NOT NULL DEFAULT now(),
    is_active            boolean       NOT NULL DEFAULT true,
    -- La clave de partición debe formar parte de la clave primaria.
    CONSTRAINT pk_location_sample PRIMARY KEY (location_sample_id, captured_at),
    CONSTRAINT fk_location_sample_device_device_id
        FOREIGN KEY (device_id) REFERENCES device (device_id) ON DELETE CASCADE,
    CONSTRAINT fk_location_sample_child_child_id
        FOREIGN KEY (child_id) REFERENCES child (child_id) ON DELETE CASCADE,
    CONSTRAINT ck_location_sample_source CHECK (source IN ('on_demand', 'verification', 'event', 'heartbeat')),
    CONSTRAINT ck_location_sample_latitude CHECK (latitude BETWEEN -90 AND 90),
    CONSTRAINT ck_location_sample_longitude CHECK (longitude BETWEEN -180 AND 180),
    CONSTRAINT ck_location_sample_accuracy_meters CHECK (accuracy_meters >= 0)
) PARTITION BY RANGE (captured_at);

CREATE INDEX ix_location_sample_child_id_captured_at ON location_sample (child_id, captured_at DESC);
CREATE INDEX ix_location_sample_device_id_captured_at ON location_sample (device_id, captured_at DESC);

-- Particiones iniciales; la tarea partition-maintenance crea las siguientes con dos meses de antelación.
CREATE TABLE location_sample_2026_10 PARTITION OF location_sample
    FOR VALUES FROM ('2026-10-01 00:00:00+00') TO ('2026-11-01 00:00:00+00');
CREATE TABLE location_sample_2026_11 PARTITION OF location_sample
    FOR VALUES FROM ('2026-11-01 00:00:00+00') TO ('2026-12-01 00:00:00+00');
CREATE TABLE location_sample_2026_12 PARTITION OF location_sample
    FOR VALUES FROM ('2026-12-01 00:00:00+00') TO ('2027-01-01 00:00:00+00');

-- Eventos de seguridad ("alertas"): llegadas, salidas, no llegadas y estado del dispositivo.
CREATE TABLE safety_event (
    safety_event_id         uuid          NOT NULL DEFAULT uuidv7(),
    child_id                uuid          NOT NULL,
    event_type_id           uuid          NOT NULL,
    watch_session_id        uuid,
    destination_id          uuid,
    location_sample_id      uuid,
    occurred_at             timestamptz   NOT NULL,
    latitude                numeric(9,6),
    longitude               numeric(9,6),
    accuracy_meters         numeric(7,1),
    is_location_unverified  boolean       NOT NULL DEFAULT false,
    detected_by             varchar(50),
    actor_guardian_id       uuid,
    dedup_key               varchar(500)  NOT NULL,
    created_at              timestamptz   NOT NULL DEFAULT now(),
    updated_at              timestamptz   NOT NULL DEFAULT now(),
    is_active               boolean       NOT NULL DEFAULT true,
    CONSTRAINT pk_safety_event PRIMARY KEY (safety_event_id),
    CONSTRAINT fk_safety_event_child_child_id
        FOREIGN KEY (child_id) REFERENCES child (child_id) ON DELETE CASCADE,
    CONSTRAINT fk_safety_event_cat_event_type_event_type_id
        FOREIGN KEY (event_type_id) REFERENCES cat_event_type (event_type_id),
    CONSTRAINT fk_safety_event_destination_destination_id
        FOREIGN KEY (destination_id) REFERENCES destination (destination_id) ON DELETE SET NULL,
    CONSTRAINT fk_safety_event_guardian_actor_guardian_id
        FOREIGN KEY (actor_guardian_id) REFERENCES guardian (guardian_id) ON DELETE SET NULL,
    -- watch_session_id referencia watch_session (V4). location_sample_id no lleva clave foránea:
    -- la muestra se elimina a los 30 días con su partición, y el evento se conserva sin coordenadas.
    CONSTRAINT uq_safety_event_dedup_key UNIQUE (dedup_key),
    CONSTRAINT ck_safety_event_detected_by CHECK (
        detected_by IS NULL OR detected_by IN ('geofence_dwell', 'geofence_exit', 'verification', 'arrival_deadline', 'device_report', 'on_demand')
    ),
    CONSTRAINT ck_safety_event_coordinates_together CHECK (
        (latitude IS NULL AND longitude IS NULL AND accuracy_meters IS NULL) OR
        (latitude IS NOT NULL AND longitude IS NOT NULL AND accuracy_meters IS NOT NULL)
    )
);

CREATE INDEX ix_safety_event_child_id_occurred_at ON safety_event (child_id, occurred_at DESC)
    INCLUDE (event_type_id, destination_id);

CREATE TABLE notification (
    notification_id      uuid         NOT NULL DEFAULT uuidv7(),
    safety_event_id      uuid         NOT NULL,
    guardian_id          uuid         NOT NULL,
    priority             varchar(50)  NOT NULL,
    status               varchar(50)  NOT NULL DEFAULT 'pending',
    attempts             smallint     NOT NULL DEFAULT 0,
    provider_message_id  varchar(500),
    last_error_code      varchar(50),
    sent_at              timestamptz,
    read_at              timestamptz,
    created_at           timestamptz  NOT NULL DEFAULT now(),
    updated_at           timestamptz  NOT NULL DEFAULT now(),
    is_active            boolean      NOT NULL DEFAULT true,
    CONSTRAINT pk_notification PRIMARY KEY (notification_id),
    CONSTRAINT fk_notification_safety_event_safety_event_id
        FOREIGN KEY (safety_event_id) REFERENCES safety_event (safety_event_id) ON DELETE CASCADE,
    CONSTRAINT fk_notification_guardian_guardian_id
        FOREIGN KEY (guardian_id) REFERENCES guardian (guardian_id) ON DELETE CASCADE,
    CONSTRAINT uq_notification_safety_event_id_guardian_id UNIQUE (safety_event_id, guardian_id),
    CONSTRAINT ck_notification_priority CHECK (priority IN ('normal', 'high')),
    CONSTRAINT ck_notification_status CHECK (status IN ('pending', 'sent', 'failed', 'skipped'))
);

CREATE INDEX ix_notification_guardian_id_created_at ON notification (guardian_id, created_at DESC);
```

`R__cat_event_type.sql` (migración repetible; identifica cada fila por `code`, nunca por identificador)

```sql
INSERT INTO cat_event_type (code, description, is_priority_by_default) VALUES
    ('ARRIVAL',            'Llegada al destino',                          false),
    ('LATE_ARRIVAL',       'Llegada posterior a la alerta de no llegada', false),
    ('DEPARTURE',          'Salida del destino tras la llegada',          true),
    ('RETURN',             'Regreso al destino tras una salida',          false),
    ('NO_ARRIVAL',         'Sin llegada a la hora esperada',              true),
    ('PERMISSION_REVOKED', 'Permiso de ubicación revocado o reducido',    true),
    ('DEVICE_OFFLINE',     'Dispositivo sin respuesta',                   false),
    ('DEVICE_RECONNECTED', 'Dispositivo reconectado',                     false),
    ('BATTERY_LOW',        'Batería baja',                                false),
    ('LOCATION_QUERY',     'Consulta de ubicación por un tutor',          false)
ON CONFLICT (code) DO UPDATE
    SET description = EXCLUDED.description,
        is_priority_by_default = EXCLUDED.is_priority_by_default,
        updated_at = now();
```

Las tablas `watch_session`, `device_event`, `location_request`, `refresh_token`, `consent` y demás del SDD §6.3 siguen el mismo patrón en `V4` y siguientes.

### 4.3 Entidades JPA

Las entidades JPA viven en `saferoute-infrastructure/.../persistence/entity` y se mapean a las entidades de dominio con MapStruct (SDD §4.2). Los valores enumerados se guardan en minúsculas mediante un convertidor común.

`persistence/converter/PersistableEnum.java` y su convertidor base

```java
package mx.saferoute.infrastructure.persistence.converter;

import jakarta.persistence.AttributeConverter;
import java.util.Arrays;

/** Enumeración con un valor estable en base de datos, independiente del nombre Java. */
public interface PersistableEnum {
  String dbValue();

  /** Convierte entre la enumeración y su valor en base de datos. */
  abstract class Converter<E extends Enum<E> & PersistableEnum>
      implements AttributeConverter<E, String> {

    private final Class<E> type;

    protected Converter(Class<E> type) {
      this.type = type;
    }

    @Override
    public String convertToDatabaseColumn(E value) {
      return value == null ? null : value.dbValue();
    }

    @Override
    public E convertToEntityAttribute(String dbValue) {
      if (dbValue == null) {
        return null;
      }
      return Arrays.stream(type.getEnumConstants())
          .filter(constant -> constant.dbValue().equals(dbValue))
          .findFirst()
          .orElseThrow(() -> new IllegalStateException("Valor desconocido para " + type + ": " + dbValue));
    }
  }
}
```

`persistence/entity/AuditedEntity.java`

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.MappedSuperclass;
import jakarta.persistence.PrePersist;
import jakarta.persistence.PreUpdate;
import java.time.Instant;

/** Campos de control comunes a toda tabla. */
@MappedSuperclass
public abstract class AuditedEntity {

  @Column(name = "created_at", nullable = false, updatable = false)
  private Instant createdAt;

  @Column(name = "updated_at", nullable = false)
  private Instant updatedAt;

  @Column(name = "is_active", nullable = false)
  private boolean isActive = true;

  @PrePersist
  void onCreate() {
    Instant now = Instant.now();
    createdAt = now;
    updatedAt = now;
  }

  @PreUpdate
  void onUpdate() {
    updatedAt = Instant.now();
  }

  public Instant getCreatedAt() { return createdAt; }
  public Instant getUpdatedAt() { return updatedAt; }
  public boolean isActive() { return isActive; }

  /** Baja lógica: actualiza la fecha junto con el indicador. */
  public void deactivate() {
    isActive = false;
  }
}
```

Las marcas de tiempo de auditoría usan el reloj del servidor en el ciclo de vida de JPA; las horas de negocio (`occurredAt`, `capturedAt`, ventanas) vienen siempre del `Clock` inyectado o del dispositivo.

`persistence/entity/GuardianEntity.java` (rol TUTOR)

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "guardian")
public class GuardianEntity extends AuditedEntity {

  @Id
  @Column(name = "guardian_id", nullable = false, updatable = false)
  private UUID guardianId;

  @Column(name = "email", nullable = false, length = 500)
  private String email;

  @Column(name = "password_hash", length = 500)
  private String passwordHash;

  @Column(name = "display_name", nullable = false, length = 50)
  private String displayName;

  @Column(name = "phone_number", length = 50)
  private String phoneNumber;

  @Column(name = "photo_url", length = 500)
  private String photoUrl;

  @Column(name = "is_email_verified", nullable = false)
  private boolean isEmailVerified;

  @Column(name = "deletion_requested_at")
  private Instant deletionRequestedAt;

  protected GuardianEntity() {}

  public GuardianEntity(UUID guardianId, String email, String passwordHash, String displayName) {
    this.guardianId = guardianId;
    this.email = email;
    this.passwordHash = passwordHash;
    this.displayName = displayName;
  }

  public UUID getGuardianId() { return guardianId; }
  public String getEmail() { return email; }
  public String getPasswordHash() { return passwordHash; }
  public String getDisplayName() { return displayName; }
  public String getPhoneNumber() { return phoneNumber; }
  public String getPhotoUrl() { return photoUrl; }
  public boolean isEmailVerified() { return isEmailVerified; }
  public Instant getDeletionRequestedAt() { return deletionRequestedAt; }

  public void markEmailVerified() { isEmailVerified = true; }
  public void changePasswordHash(String newPasswordHash) { passwordHash = newPasswordHash; }
  public void updateProfile(String newDisplayName, String newPhoneNumber, String newPhotoUrl) {
    displayName = newDisplayName;
    phoneNumber = newPhoneNumber;
    photoUrl = newPhotoUrl;
  }
  public void requestDeletion(Instant requestedAt) { deletionRequestedAt = requestedAt; }
}
```

`persistence/entity/ChildEntity.java` y `DeviceEntity.java` (rol CHILD)

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.util.UUID;

@Entity
@Table(name = "child")
public class ChildEntity extends AuditedEntity {

  @Id
  @Column(name = "child_id", nullable = false, updatable = false)
  private UUID childId;

  @Column(name = "family_group_id", nullable = false, updatable = false)
  private UUID familyGroupId;

  @Column(name = "display_name", nullable = false, length = 50)
  private String displayName;

  @Column(name = "birth_year")
  private Short birthYear;

  @Column(name = "photo_url", length = 500)
  private String photoUrl;

  protected ChildEntity() {}

  public ChildEntity(UUID childId, UUID familyGroupId, String displayName, Short birthYear) {
    this.childId = childId;
    this.familyGroupId = familyGroupId;
    this.displayName = displayName;
    this.birthYear = birthYear;
  }

  public UUID getChildId() { return childId; }
  public UUID getFamilyGroupId() { return familyGroupId; }
  public String getDisplayName() { return displayName; }
  public Short getBirthYear() { return birthYear; }
  public String getPhotoUrl() { return photoUrl; }

  public void updateProfile(String newDisplayName, Short newBirthYear, String newPhotoUrl) {
    displayName = newDisplayName;
    birthYear = newBirthYear;
    photoUrl = newPhotoUrl;
  }
}
```

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.time.Instant;
import java.util.UUID;
import mx.saferoute.infrastructure.persistence.converter.PersistableEnum;

@Entity
@Table(name = "device")
public class DeviceEntity extends AuditedEntity {

  public enum LocationPermission implements PersistableEnum {
    ALWAYS("always"), WHILE_IN_USE("while_in_use"), DENIED("denied");
    private final String dbValue;
    LocationPermission(String dbValue) { this.dbValue = dbValue; }
    @Override public String dbValue() { return dbValue; }
    @jakarta.persistence.Converter(autoApply = true)
    public static class Converter extends PersistableEnum.Converter<LocationPermission> {
      public Converter() { super(LocationPermission.class); }
    }
  }

  public enum PairingStatus implements PersistableEnum {
    COMPLETE("complete"), INCOMPLETE("incomplete");
    private final String dbValue;
    PairingStatus(String dbValue) { this.dbValue = dbValue; }
    @Override public String dbValue() { return dbValue; }
    @jakarta.persistence.Converter(autoApply = true)
    public static class Converter extends PersistableEnum.Converter<PairingStatus> {
      public Converter() { super(PairingStatus.class); }
    }
  }

  @Id
  @Column(name = "device_id", nullable = false, updatable = false)
  private UUID deviceId;

  @Column(name = "child_id", nullable = false, updatable = false)
  private UUID childId;

  @Column(name = "manufacturer", nullable = false, length = 50)
  private String manufacturer;

  @Column(name = "model", nullable = false, length = 50)
  private String model;

  @Column(name = "os_version", nullable = false, length = 50)
  private String osVersion;

  @Column(name = "app_version", nullable = false, length = 50)
  private String appVersion;

  @Column(name = "push_token", length = 500)
  private String pushToken;

  @Convert(converter = LocationPermission.Converter.class)
  @Column(name = "location_permission", nullable = false, length = 50)
  private LocationPermission locationPermission;

  @Column(name = "is_precise_location", nullable = false)
  private boolean isPreciseLocation;

  @Column(name = "is_battery_optimization_exempt", nullable = false)
  private boolean isBatteryOptimizationExempt;

  @Convert(converter = PairingStatus.Converter.class)
  @Column(name = "pairing_status", nullable = false, length = 50)
  private PairingStatus pairingStatus;

  @Column(name = "battery_level")
  private Short batteryLevel;

  @Column(name = "applied_config_version", nullable = false)
  private int appliedConfigVersion;

  @Column(name = "last_seen_at")
  private Instant lastSeenAt;

  @Column(name = "paired_at", nullable = false, updatable = false)
  private Instant pairedAt;

  @Column(name = "unpaired_at")
  private Instant unpairedAt;

  protected DeviceEntity() {}

  public DeviceEntity(
      UUID deviceId, UUID childId, String manufacturer, String model, String osVersion,
      String appVersion, String pushToken, Instant pairedAt) {
    this.deviceId = deviceId;
    this.childId = childId;
    this.manufacturer = manufacturer;
    this.model = model;
    this.osVersion = osVersion;
    this.appVersion = appVersion;
    this.pushToken = pushToken;
    this.locationPermission = LocationPermission.DENIED;
    this.pairingStatus = PairingStatus.INCOMPLETE;
    this.pairedAt = pairedAt;
  }

  public UUID getDeviceId() { return deviceId; }
  public UUID getChildId() { return childId; }
  public String getManufacturer() { return manufacturer; }
  public String getModel() { return model; }
  public String getOsVersion() { return osVersion; }
  public String getAppVersion() { return appVersion; }
  public String getPushToken() { return pushToken; }
  public LocationPermission getLocationPermission() { return locationPermission; }
  public boolean isPreciseLocation() { return isPreciseLocation; }
  public boolean isBatteryOptimizationExempt() { return isBatteryOptimizationExempt; }
  public PairingStatus getPairingStatus() { return pairingStatus; }
  public Short getBatteryLevel() { return batteryLevel; }
  public int getAppliedConfigVersion() { return appliedConfigVersion; }
  public Instant getLastSeenAt() { return lastSeenAt; }
  public Instant getPairedAt() { return pairedAt; }
  public Instant getUnpairedAt() { return unpairedAt; }

  /** Aplica el estado reportado en un latido o un cambio de permisos. */
  public void applyStatus(
      LocationPermission permission, boolean isPrecise, boolean isExempt, Short battery,
      int configVersion, Instant seenAt) {
    locationPermission = permission;
    isPreciseLocation = isPrecise;
    isBatteryOptimizationExempt = isExempt;
    batteryLevel = battery;
    appliedConfigVersion = configVersion;
    lastSeenAt = seenAt;
    pairingStatus =
        permission == LocationPermission.ALWAYS && isPrecise ? PairingStatus.COMPLETE : PairingStatus.INCOMPLETE;
  }

  public void updatePushToken(String newPushToken) { pushToken = newPushToken; }

  public void unpair(Instant at) {
    unpairedAt = at;
    deactivate();
  }
}
```

`persistence/entity/DestinationEntity.java` y `DestinationScheduleEntity.java`

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.CascadeType;
import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.OneToMany;
import jakarta.persistence.Table;
import jakarta.persistence.Version;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.UUID;
import mx.saferoute.infrastructure.persistence.converter.PersistableEnum;

@Entity
@Table(name = "destination")
public class DestinationEntity extends AuditedEntity {

  public enum Status implements PersistableEnum {
    ACTIVE("active"), PAUSED("paused");
    private final String dbValue;
    Status(String dbValue) { this.dbValue = dbValue; }
    @Override public String dbValue() { return dbValue; }
    @jakarta.persistence.Converter(autoApply = true)
    public static class Converter extends PersistableEnum.Converter<Status> {
      public Converter() { super(Status.class); }
    }
  }

  @Id
  @Column(name = "destination_id", nullable = false, updatable = false)
  private UUID destinationId;

  @Column(name = "family_group_id", nullable = false, updatable = false)
  private UUID familyGroupId;

  @Column(name = "name", nullable = false, length = 50)
  private String name;

  @Column(name = "icon", nullable = false, length = 50)
  private String icon;

  @Column(name = "address", length = 500)
  private String address;

  @Column(name = "latitude", nullable = false, precision = 9, scale = 6)
  private BigDecimal latitude;

  @Column(name = "longitude", nullable = false, precision = 9, scale = 6)
  private BigDecimal longitude;

  @Column(name = "radius_meters", nullable = false)
  private int radiusMeters;

  @Column(name = "time_zone_id", nullable = false, length = 50)
  private String timeZoneId;

  @Convert(converter = Status.Converter.class)
  @Column(name = "status", nullable = false, length = 50)
  private Status status = Status.ACTIVE;

  @Column(name = "config_version", nullable = false)
  private int configVersion = 1;

  @Version
  @Column(name = "version", nullable = false)
  private long version;

  @OneToMany(mappedBy = "destination", cascade = CascadeType.ALL, orphanRemoval = true)
  private List<DestinationScheduleEntity> schedules = new ArrayList<>();

  protected DestinationEntity() {}

  public DestinationEntity(
      UUID destinationId, UUID familyGroupId, String name, String icon, String address,
      BigDecimal latitude, BigDecimal longitude, int radiusMeters, String timeZoneId) {
    this.destinationId = destinationId;
    this.familyGroupId = familyGroupId;
    this.name = name;
    this.icon = icon;
    this.address = address;
    this.latitude = latitude;
    this.longitude = longitude;
    this.radiusMeters = radiusMeters;
    this.timeZoneId = timeZoneId;
  }

  public UUID getDestinationId() { return destinationId; }
  public UUID getFamilyGroupId() { return familyGroupId; }
  public String getName() { return name; }
  public String getIcon() { return icon; }
  public String getAddress() { return address; }
  public BigDecimal getLatitude() { return latitude; }
  public BigDecimal getLongitude() { return longitude; }
  public int getRadiusMeters() { return radiusMeters; }
  public String getTimeZoneId() { return timeZoneId; }
  public Status getStatus() { return status; }
  public int getConfigVersion() { return configVersion; }
  public List<DestinationScheduleEntity> getSchedules() { return Collections.unmodifiableList(schedules); }

  /** Reemplaza los datos y horarios del destino e incrementa la versión de configuración. */
  public void replace(
      String newName, String newIcon, String newAddress, BigDecimal newLatitude,
      BigDecimal newLongitude, int newRadiusMeters, String newTimeZoneId,
      List<DestinationScheduleEntity> newSchedules) {
    name = newName;
    icon = newIcon;
    address = newAddress;
    latitude = newLatitude;
    longitude = newLongitude;
    radiusMeters = newRadiusMeters;
    timeZoneId = newTimeZoneId;
    schedules.clear();
    newSchedules.forEach(this::addSchedule);
    configVersion++;
  }

  public void addSchedule(DestinationScheduleEntity schedule) {
    schedule.attachTo(this);
    schedules.add(schedule);
  }

  public void changeStatus(Status newStatus) {
    status = newStatus;
    configVersion++;
  }
}
```

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;
import java.time.LocalDate;
import java.time.LocalTime;
import java.util.UUID;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;

@Entity
@Table(name = "destination_schedule")
public class DestinationScheduleEntity extends AuditedEntity {

  @Id
  @Column(name = "destination_schedule_id", nullable = false, updatable = false)
  private UUID destinationScheduleId;

  @ManyToOne(fetch = FetchType.LAZY, optional = false)
  @JoinColumn(name = "destination_id", nullable = false, updatable = false)
  private DestinationEntity destination;

  @Column(name = "recurrence_type", nullable = false, length = 50)
  private String recurrenceType;

  @Column(name = "specific_date")
  private LocalDate specificDate;

  /** Días ISO 8601: 1 = lunes … 7 = domingo. */
  @JdbcTypeCode(SqlTypes.ARRAY)
  @Column(name = "days_of_week", columnDefinition = "smallint[]")
  private Short[] daysOfWeek;

  @Column(name = "window_start_time", nullable = false)
  private LocalTime windowStartTime;

  @Column(name = "window_end_time", nullable = false)
  private LocalTime windowEndTime;

  @Column(name = "expected_arrival_time")
  private LocalTime expectedArrivalTime;

  @Column(name = "tolerance_minutes", nullable = false)
  private short toleranceMinutes;

  protected DestinationScheduleEntity() {}

  public DestinationScheduleEntity(
      UUID destinationScheduleId, String recurrenceType, LocalDate specificDate, Short[] daysOfWeek,
      LocalTime windowStartTime, LocalTime windowEndTime, LocalTime expectedArrivalTime,
      short toleranceMinutes) {
    this.destinationScheduleId = destinationScheduleId;
    this.recurrenceType = recurrenceType;
    this.specificDate = specificDate;
    this.daysOfWeek = daysOfWeek == null ? null : daysOfWeek.clone();
    this.windowStartTime = windowStartTime;
    this.windowEndTime = windowEndTime;
    this.expectedArrivalTime = expectedArrivalTime;
    this.toleranceMinutes = toleranceMinutes;
  }

  void attachTo(DestinationEntity owner) { destination = owner; }

  public UUID getDestinationScheduleId() { return destinationScheduleId; }
  public String getRecurrenceType() { return recurrenceType; }
  public LocalDate getSpecificDate() { return specificDate; }
  public Short[] getDaysOfWeek() { return daysOfWeek == null ? null : daysOfWeek.clone(); }
  public LocalTime getWindowStartTime() { return windowStartTime; }
  public LocalTime getWindowEndTime() { return windowEndTime; }
  public LocalTime getExpectedArrivalTime() { return expectedArrivalTime; }
  public short getToleranceMinutes() { return toleranceMinutes; }
}
```

`persistence/entity/LocationSampleEntity.java` (historial de ubicaciones)

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Convert;
import jakarta.persistence.Embeddable;
import jakarta.persistence.EmbeddedId;
import jakarta.persistence.Entity;
import jakarta.persistence.Table;
import java.io.Serializable;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.Objects;
import java.util.UUID;
import mx.saferoute.infrastructure.persistence.converter.PersistableEnum;

@Entity
@Table(name = "location_sample")
public class LocationSampleEntity extends AuditedEntity {

  /** Clave compuesta: la tabla está particionada por captured_at. */
  @Embeddable
  public static class Key implements Serializable {
    @Column(name = "location_sample_id", nullable = false, updatable = false)
    private UUID locationSampleId;

    @Column(name = "captured_at", nullable = false, updatable = false)
    private Instant capturedAt;

    protected Key() {}

    public Key(UUID locationSampleId, Instant capturedAt) {
      this.locationSampleId = locationSampleId;
      this.capturedAt = capturedAt;
    }

    public UUID getLocationSampleId() { return locationSampleId; }
    public Instant getCapturedAt() { return capturedAt; }

    @Override
    public boolean equals(Object other) {
      return other instanceof Key key
          && locationSampleId.equals(key.locationSampleId)
          && capturedAt.equals(key.capturedAt);
    }

    @Override
    public int hashCode() { return Objects.hash(locationSampleId, capturedAt); }
  }

  public enum Source implements PersistableEnum {
    ON_DEMAND("on_demand"), VERIFICATION("verification"), EVENT("event"), HEARTBEAT("heartbeat");
    private final String dbValue;
    Source(String dbValue) { this.dbValue = dbValue; }
    @Override public String dbValue() { return dbValue; }
    @jakarta.persistence.Converter(autoApply = true)
    public static class Converter extends PersistableEnum.Converter<Source> {
      public Converter() { super(Source.class); }
    }
  }

  @EmbeddedId
  private Key key;

  @Column(name = "device_id", nullable = false, updatable = false)
  private UUID deviceId;

  @Column(name = "child_id", nullable = false, updatable = false)
  private UUID childId;

  @Column(name = "location_request_id", updatable = false)
  private UUID locationRequestId;

  @Convert(converter = Source.Converter.class)
  @Column(name = "source", nullable = false, updatable = false, length = 50)
  private Source source;

  @Column(name = "latitude", nullable = false, updatable = false, precision = 9, scale = 6)
  private BigDecimal latitude;

  @Column(name = "longitude", nullable = false, updatable = false, precision = 9, scale = 6)
  private BigDecimal longitude;

  @Column(name = "accuracy_meters", nullable = false, updatable = false, precision = 7, scale = 1)
  private BigDecimal accuracyMeters;

  @Column(name = "is_mock", nullable = false, updatable = false)
  private boolean isMock;

  protected LocationSampleEntity() {}

  public LocationSampleEntity(
      Key key, UUID deviceId, UUID childId, UUID locationRequestId, Source source,
      BigDecimal latitude, BigDecimal longitude, BigDecimal accuracyMeters, boolean isMock) {
    this.key = key;
    this.deviceId = deviceId;
    this.childId = childId;
    this.locationRequestId = locationRequestId;
    this.source = source;
    this.latitude = latitude;
    this.longitude = longitude;
    this.accuracyMeters = accuracyMeters;
    this.isMock = isMock;
  }

  public Key getKey() { return key; }
  public UUID getDeviceId() { return deviceId; }
  public UUID getChildId() { return childId; }
  public UUID getLocationRequestId() { return locationRequestId; }
  public Source getSource() { return source; }
  public BigDecimal getLatitude() { return latitude; }
  public BigDecimal getLongitude() { return longitude; }
  public BigDecimal getAccuracyMeters() { return accuracyMeters; }
  public boolean isMock() { return isMock; }
}
```

Las muestras son inmutables: todas sus columnas son `updatable = false`.

`persistence/entity/SafetyEventEntity.java` (alertas y eventos)

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.FetchType;
import jakarta.persistence.Id;
import jakarta.persistence.JoinColumn;
import jakarta.persistence.ManyToOne;
import jakarta.persistence.Table;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Entity
@Table(name = "safety_event")
public class SafetyEventEntity extends AuditedEntity {

  @Id
  @Column(name = "safety_event_id", nullable = false, updatable = false)
  private UUID safetyEventId;

  @Column(name = "child_id", nullable = false, updatable = false)
  private UUID childId;

  @ManyToOne(fetch = FetchType.EAGER, optional = false)
  @JoinColumn(name = "event_type_id", nullable = false, updatable = false)
  private CatEventTypeEntity eventType;

  @Column(name = "watch_session_id", updatable = false)
  private UUID watchSessionId;

  @Column(name = "destination_id", updatable = false)
  private UUID destinationId;

  @Column(name = "location_sample_id", updatable = false)
  private UUID locationSampleId;

  @Column(name = "occurred_at", nullable = false, updatable = false)
  private Instant occurredAt;

  @Column(name = "latitude", precision = 9, scale = 6)
  private BigDecimal latitude;

  @Column(name = "longitude", precision = 9, scale = 6)
  private BigDecimal longitude;

  @Column(name = "accuracy_meters", precision = 7, scale = 1)
  private BigDecimal accuracyMeters;

  @Column(name = "is_location_unverified", nullable = false, updatable = false)
  private boolean isLocationUnverified;

  @Column(name = "detected_by", updatable = false, length = 50)
  private String detectedBy;

  @Column(name = "actor_guardian_id", updatable = false)
  private UUID actorGuardianId;

  @Column(name = "dedup_key", nullable = false, updatable = false, length = 500)
  private String dedupKey;

  protected SafetyEventEntity() {}

  public SafetyEventEntity(
      UUID safetyEventId, UUID childId, CatEventTypeEntity eventType, UUID watchSessionId,
      UUID destinationId, UUID locationSampleId, Instant occurredAt, BigDecimal latitude,
      BigDecimal longitude, BigDecimal accuracyMeters, boolean isLocationUnverified,
      String detectedBy, UUID actorGuardianId, String dedupKey) {
    this.safetyEventId = safetyEventId;
    this.childId = childId;
    this.eventType = eventType;
    this.watchSessionId = watchSessionId;
    this.destinationId = destinationId;
    this.locationSampleId = locationSampleId;
    this.occurredAt = occurredAt;
    this.latitude = latitude;
    this.longitude = longitude;
    this.accuracyMeters = accuracyMeters;
    this.isLocationUnverified = isLocationUnverified;
    this.detectedBy = detectedBy;
    this.actorGuardianId = actorGuardianId;
    this.dedupKey = dedupKey;
  }

  public UUID getSafetyEventId() { return safetyEventId; }
  public UUID getChildId() { return childId; }
  public CatEventTypeEntity getEventType() { return eventType; }
  public UUID getWatchSessionId() { return watchSessionId; }
  public UUID getDestinationId() { return destinationId; }
  public UUID getLocationSampleId() { return locationSampleId; }
  public Instant getOccurredAt() { return occurredAt; }
  public BigDecimal getLatitude() { return latitude; }
  public BigDecimal getLongitude() { return longitude; }
  public BigDecimal getAccuracyMeters() { return accuracyMeters; }
  public boolean isLocationUnverified() { return isLocationUnverified; }
  public String getDetectedBy() { return detectedBy; }
  public UUID getActorGuardianId() { return actorGuardianId; }
  public String getDedupKey() { return dedupKey; }
}
```

`persistence/entity/CatEventTypeEntity.java`

```java
package mx.saferoute.infrastructure.persistence.entity;

import jakarta.persistence.Column;
import jakarta.persistence.Entity;
import jakarta.persistence.Id;
import jakarta.persistence.Table;
import java.util.UUID;
import org.hibernate.annotations.Immutable;

/** Catálogo de tipos de evento; se llena con la migración repetible R__cat_event_type.sql. */
@Entity
@Immutable
@Table(name = "cat_event_type")
public class CatEventTypeEntity extends AuditedEntity {

  @Id
  @Column(name = "event_type_id", nullable = false, updatable = false)
  private UUID eventTypeId;

  @Column(name = "code", nullable = false, length = 50)
  private String code;

  @Column(name = "description", nullable = false, length = 500)
  private String description;

  @Column(name = "is_priority_by_default", nullable = false)
  private boolean isPriorityByDefault;

  protected CatEventTypeEntity() {}

  public UUID getEventTypeId() { return eventTypeId; }
  public String getCode() { return code; }
  public String getDescription() { return description; }
  public boolean isPriorityByDefault() { return isPriorityByDefault; }
}
```

### 4.4 Índice espacial: por qué no hace falta

El índice `2dsphere` de MongoDB sirve para consultas de cercanía ("destinos a menos de X metros"). SafeRoute no las necesita:

| Operación | Cómo se resuelve | Índice |
|---|---|---|
| ¿El menor está en su destino? | Se carga el destino por `destination_id` (≤ 10 por menor) y se aplica Haversine en Java (§1.4) | `pk_destination`, `ix_destination_child_child_id` |
| Geocercas de un dispositivo | Destinos activos del menor | `ix_destination_child_child_id` |
| Historial de ubicaciones de un menor | Rango de tiempo por menor | `ix_location_sample_child_id_captured_at` |

Si en el futuro se necesitara una consulta espacial (p. ej., sugerir destinos ya registrados cerca de un punto), se agregaría la extensión PostGIS (GPL-2.0, usada como servicio sin modificar, permitida por el SDD §14.1) con una columna `geography(Point, 4326)` y un índice GiST:

```sql
CREATE EXTENSION IF NOT EXISTS postgis;
ALTER TABLE destination ADD COLUMN location geography(Point, 4326)
    GENERATED ALWAYS AS (ST_SetSRID(ST_MakePoint(longitude::float8, latitude::float8), 4326)::geography) STORED;
CREATE INDEX ix_destination_location ON destination USING gist (location);
```

No se incluye en el MVP (KISS).

### 4.5 Purga automática (equivalente al índice TTL)

| Mecanismo MongoDB | Equivalente SafeRoute |
|---|---|
| Índice TTL (`expireAfterSeconds`) que borra documento a documento cada 60 s | Tarea `retention-purge` (diaria, 03:00 UTC) que hace `DETACH PARTITION ... CONCURRENTLY` y `DROP TABLE` de las particiones mensuales completamente vencidas |
| — | Tarea `partition-maintenance` (diaria) que crea las particiones de los dos meses siguientes |
| — | Para los días de la partición en curso que superan los 30 días, `DELETE` por lotes de 5 000 filas con `captured_at < now() - interval '30 days'` |

Ventajas sobre un TTL documento a documento: el borrado de una partición es instantáneo, no fragmenta los índices ni genera carga de escritura, y la retención se garantiza a nivel de día.

```sql
-- Ejecutado por retention-purge para cada partición cuyo límite superior es anterior a now() - 30 días.
ALTER TABLE location_sample DETACH PARTITION location_sample_2026_08 CONCURRENTLY;
DROP TABLE location_sample_2026_08;

-- Recorte de la partición más antigua que sigue adjunta.
DELETE FROM location_sample
 WHERE ctid IN (
     SELECT ctid FROM location_sample
      WHERE captured_at < now() - interval '30 days'
      LIMIT 5000);

-- Anulación de coordenadas en eventos de más de 30 días (el evento se conserva).
UPDATE safety_event
   SET latitude = NULL, longitude = NULL, accuracy_meters = NULL, updated_at = now()
 WHERE occurred_at < now() - interval '30 days'
   AND latitude IS NOT NULL;
```

### 4.6 Diagrama del modelo

```mermaid
erDiagram
    guardian ||--o{ family_membership : "pertenece"
    family_group ||--o{ family_membership : "tiene"
    family_group ||--o{ child : "agrupa"
    family_group ||--o{ destination : "define"
    child ||--o| device : "usa (rol CHILD)"
    destination ||--o{ destination_schedule : "horarios"
    destination ||--o{ destination_child : "asignado a"
    child ||--o{ destination_child : ""
    device ||--o{ location_sample : "captura"
    child ||--o{ location_sample : ""
    child ||--o{ safety_event : "alertas"
    cat_event_type ||--o{ safety_event : "tipo"
    destination ||--o{ safety_event : ""
    safety_event ||--o{ notification : "notifica"
    guardian ||--o{ notification : "recibe"
```

### 4.7 Reglas de integridad que garantiza la base

| Regla | Restricción |
|---|---|
| Radio entre 100 y 1 000 m | `ck_destination_radius_meters` |
| Coordenadas válidas | `ck_destination_latitude`, `ck_destination_longitude`, `ck_location_sample_*` |
| Un dispositivo activo por menor | `uq_device_child_id_active` |
| Una llegada por sesión | `uq_safety_event_dedup_key` |
| Una notificación por evento y tutor | `uq_notification_safety_event_id_guardian_id` |
| Coherencia de horarios | `ck_destination_schedule_*` |
| Coordenadas completas o ausentes en eventos | `ck_safety_event_coordinates_together` |

### 4.8 Estimación de almacenamiento

Tamaño estimado por fila (datos + índices) y volumen a 30 días:

| Tabla | Bytes por fila | Piloto (20 familias) | 5 000 familias | 50 000 familias |
|---|---|---|---|---|
| `location_sample` (retenida 30 días) | ~200 | ~1 MB | ~1,4 GB | ~14 GB |
| `device_event` (retenida 30 días) | ~250 | ~1 MB | ~0,8 GB | ~7,5 GB |
| `safety_event` (por año) | ~180 | ~2 MB | ~3 GB | ~30 GB |
| `watch_session` (por año) | ~160 | ~2 MB | ~2,9 GB | ~29 GB |
| Resto | — | < 10 MB | < 1 GB | ~5 GB |

Con PostgreSQL autoalojado no hay un límite de 512 MB: el piloto completo ocupa menos de 50 MB y el crecimiento lo absorbe el disco del servidor (dimensionamiento en §2.3). Las tablas sin coordenadas que crecen indefinidamente (`watch_session`, `safety_event`) se archivan por año si fuera necesario.

---

## 5. Casos de prueba críticos y plan de aseguramiento de calidad

### 5.1 Convenciones de la matriz

| Campo | Contenido |
|---|---|
| ID | `QA-<área>-<nn>`: `GPS` precisión, `NET` red, `CYC` ciclo de 20 minutos, `WS` WebSocket |
| Nivel | `U` unitaria, `I` integración (Testcontainers), `E` emulador (escenarios del SDD §15.4), `C` campo (dispositivo físico) |
| Trazabilidad | Requisitos que verifica |

Constantes usadas en los casos: radio 150 m, histéresis 50 m, margen máximo 50 m, precisión máxima 150 m, permanencia de llegada 60 s, confirmación de salida 3 min, ciclo 20 min, espera de respuesta 5 min, sin conexión a los 30 min.

### 5.2 Precisión GPS fluctuante o baja

| ID | Precondición | Pasos | Resultado esperado | Nivel | Trazabilidad |
|---|---|---|---|---|---|
| QA-GPS-01 | Sesión `awaiting`; destino de 150 m. | 1. El dispositivo envía `GEOFENCE_DWELL` con `accuracyMeters = 300`. | No se registra llegada. Se crea `location_request` (`arrival_confirmation`) y se envía `LOCATION_REQUEST`. | U, I | RF-030, §1.4.3 |
| QA-GPS-02 | Como QA-GPS-01. | 1. `DWELL` con precisión 300 m. 2. La ubicación fresca llega con precisión 15 m a 80 m del centro. | Se registra `ARRIVAL` con el `occurredAt` del `DWELL` original. | I | RF-030, RF-035 |
| QA-GPS-03 | Sesión `present`. | 1. Enviar 30 lecturas cada 10 s que oscilan entre 120 y 190 m del centro con precisión 20–40 m. | Ninguna lectura clasifica como `OUTSIDE`; no hay `DEPARTURE` ni `pending_departure_since`. | U, E | RF-032, RNF-FIA-03 |
| QA-GPS-04 | Sesión `present`. | 1. Verificación con una lectura atípica a 1,2 km con precisión 25 m. 2. A los 3 min, lectura a 60 m del centro. | Se fija `pending_departure_since` y luego se limpia. Sin `DEPARTURE`. | I | RF-032, RF-044 |
| QA-GPS-05 | Sesión `present`. | 1. Lectura a 334 m con precisión 140 m. | Clasificación `AMBIGUOUS` (334 − 140 = 194 ≤ 200); sin cambio de estado. | U | §1.4.3 |
| QA-GPS-06 | Sesión `present`. | 1. Lectura a 400 m con precisión 15 m. 2. A los 3 min, lectura a 420 m con precisión 12 m. | `DEPARTURE` con `detectedBy = verification`; alerta prioritaria y `alert:movement_detected` por WebSocket. | I, E | RF-032, RF-044, RF-045 |
| QA-GPS-07 | Sesión `awaiting`. | 1. `DWELL` con `isMock = true` dentro del destino. | `ARRIVAL` con `is_location_unverified = true`; la notificación indica "ubicación no verificada". | I, E | RT-04 |
| QA-GPS-08 | Sesión `present`; menor en interior sin señal GPS. | 1. Verificación: `getCurrentLocation` agota los 25 s sin lectura. | El ciclo se registra como `no_response`; **no** como salida. Se programa reintento a +10 min. | E, C | RNF-FIA-07e |
| QA-GPS-09 | Menor en el borde del destino (140–160 m) durante 2 h. | 1. Prueba de campo con el dispositivo estático junto a la reja de la escuela. | Cero alertas de salida. | C | RNF-FIA-03, CA-04 |
| QA-GPS-10 | Vectores de `testing/geo-test-vectors.json`. | 1. Ejecutar las pruebas del backend y de la app con los mismos vectores. | Distancias con diferencia ≤ 0,01 m y clasificaciones idénticas en ambas implementaciones. | U | §1.4.5 |

### 5.3 Pérdida y recuperación de señal celular

| ID | Precondición | Pasos | Resultado esperado | Nivel | Trazabilidad |
|---|---|---|---|---|---|
| QA-NET-01 | Sesión `awaiting`, hora esperada 07:30, tolerancia 15 min. | 1. Modo avión a las 07:10. 2. Entrar al destino a las 07:20. 3. Desactivar modo avión a las 07:40. | A las 07:47, como mucho, `NO_ARRIVAL` (sin respuesta). Al reconectar, `LATE_ARRIVAL` con `occurredAt` ≈ 07:21 y aviso a los tutores. | E | RF-034, RF-036 |
| QA-NET-02 | Como QA-NET-01, sin hora esperada. | 1. Modo avión antes de entrar. 2. Entrar. 3. Reconectar a los 10 min. | `ARRIVAL` con la hora real de entrada; sin `NO_ARRIVAL`. | E | RF-036, RNF-FIA-07b |
| QA-NET-03 | Sesión `present`. | 1. Modo avión durante 35 min. | `DEVICE_OFFLINE` una sola vez, a los 30–32 min del último contacto. Ninguna `DEPARTURE`. | I, E | RF-041, RNF-FIA-07e |
| QA-NET-04 | Tras QA-NET-03. | 1. Desactivar modo avión. | `DEVICE_RECONNECTED`; los ciclos siguientes vuelven a `inside`. | E | RF-041 |
| QA-NET-05 | Dispositivo con conexión intermitente (proxy que corta la respuesta del API). | 1. Enviar un lote de eventos; el servidor lo procesa pero la respuesta se pierde. 2. El dispositivo reintenta. | El segundo envío devuelve los `eventId` en `duplicates`; no hay filas ni notificaciones duplicadas. | I | RF-037 |
| QA-NET-06 | Dispositivo sin red. | 1. Generar 200 eventos (salidas y entradas simuladas). 2. Reconectar. | Los 200 se envían en lotes de 100, en orden de `occurredAt`; ninguno se pierde. | E | RNF-FIA-07a |
| QA-NET-07 | Ciclo de verificación en curso. | 1. El dispositivo recibe `LOCATION_REQUEST` sin red. 2. Reconecta a los 8 min. | La lectura se guarda en el historial interno; el ciclo vencido queda `no_response` y la lectura se reutiliza en el ciclo siguiente si tiene < 5 min. | I | §9.3, RNF-FIA-07 |
| QA-NET-08 | Ventana 07:00–14:00. | 1. Modo avión a las 13:50. 2. Salir del destino a las 13:55. 3. Reconectar a las 14:20. | La salida (13:58) está dentro de la ventana: `DEPARTURE` diferida con su hora real. Un evento posterior a las 14:00 solo queda en el historial. | E | RF-032, RF-036 |
| QA-NET-09 | App del tutor abierta con WebSocket. | 1. Cortar la red del emulador 2 min. 2. Durante el corte se registra un `ARRIVAL`. 3. Restaurar la red. | El cliente reconecta con espera exponencial, envía `sync:request` y recibe el `ARRIVAL` una sola vez. El push también llega; la app no duplica el aviso. | E | RF-045, RNF-FIA-07d |
| QA-NET-10 | Cola local con eventos de más de 7 días. | 1. Mantener el dispositivo sin red 8 días (reloj simulado). | Los eventos de más de 7 días se descartan localmente con un registro; el resto se envía. | U | RNF-FIA-07a |

### 5.4 Activación y desactivación del ciclo de 20 minutos

| ID | Precondición | Pasos | Resultado esperado | Nivel | Trazabilidad |
|---|---|---|---|---|---|
| QA-CYC-01 | Destino con ventana 07:00–14:00 materializado; reloj controlado. | 1. Avanzar el reloj a 07:00. | `window-opened` pone la sesión en `awaiting` y programa `verification-due:{id}:1` a las 07:20. | I | RF-044 |
| QA-CYC-02 | Como QA-CYC-01. | 1. Ejecutar el ciclo 1 con 4 min de retraso (07:24). | El ciclo 2 se programa a las 07:40 (hora planificada + 20), no a las 07:44. | U, I | RF-044, SDD §9.2 |
| QA-CYC-03 | Ventana que termina a las 14:00. | 1. Ejecutar el ciclo de las 13:40. | No se programa el ciclo de las 14:00 ni posteriores. | U | RF-044 |
| QA-CYC-04 | Sesión `present` con ciclos programados. | 1. El tutor pausa el destino a las 10:05. 2. Vence la tarea de las 10:20. | Sesión `cancelled`; la tarea de las 10:20 se descarta como obsoleta, sin `LOCATION_REQUEST`. El dispositivo recibe `CONFIG_CHANGED` y retira la geocerca. | I, E | RF-025 |
| QA-CYC-05 | Sesión en curso. | 1. El tutor cambia el radio de 150 a 300 m a las 09:10. | Se conserva la sesión en curso; los ciclos usan el radio nuevo; las sesiones futuras se rematerializan; `configVersion` aumenta y el dispositivo la confirma. | I | RF-025, RF-027 |
| QA-CYC-06 | Sesión `present`. | 1. Reiniciar el dispositivo. 2. Esperar al siguiente ciclo. | `BootReceiver` vuelve a registrar las geocercas; el ciclo recibe respuesta normal. | E, C | SDD §5.5 |
| QA-CYC-07 | Dos instancias del Worker. | 1. Programar 1 000 tareas `verification-due` a la misma hora. | Cada tarea se ejecuta exactamente una vez (`SKIP LOCKED`); ninguna sesión recibe dos `LOCATION_REQUEST` por el mismo ciclo. | I | SDD §4.7 |
| QA-CYC-08 | Ubicación de calidad recibida hace 3 min por una consulta del tutor. | 1. Vence el ciclo. | Se reutiliza la lectura; no se envía `LOCATION_REQUEST`. | U, I | SDD §9.3 |
| QA-CYC-09 | Destino en `America/Tijuana` (con horario de verano); ventana 07:00–14:00. | 1. Materializar las sesiones del día del cambio de horario. | Las sesiones abren a las 07:00 hora local antes y después del cambio; los ciclos se programan en UTC sin huecos ni duplicados. | U | RF-028 |
| QA-CYC-10 | Destino desasignado del menor durante la ventana. | 1. Quitar al menor del destino. | La sesión del menor pasa a `cancelled`; no se emiten más ciclos ni alertas para ese menor. | I | RF-022, RF-025 |
| QA-CYC-11 | Sesión cerrada (`closed`). | 1. Llega tarde una tarea `verification-due` de un ciclo anterior. | La tarea se descarta; sin efectos. | U | SDD §4.7 |
| QA-CYC-12 | Día escolar completo, dos ventanas. | 1. Prueba de campo de 5 días por fabricante. | Consumo atribuido a SafeRoute ≤ 5 % diario; ≥ 95 % de los ciclos con respuesta en Android sin capa agresiva de fabricante. | C | RNF-PER-03, RF-044 |
| QA-CYC-13 | Perfil `development` con `saferoute.verification.interval=PT2M`. | 1. Repetir QA-CYC-01 a QA-CYC-03 con el intervalo reducido. | Mismo comportamiento con ciclos de 2 min. En cualquier otro perfil, la propiedad se rechaza al arrancar. | I | RNF-OPE-03 |

### 5.5 Canal WebSocket

| ID | Precondición | Pasos | Resultado esperado | Nivel | Trazabilidad |
|---|---|---|---|---|---|
| QA-WS-01 | Tutor autenticado. | 1. Handshake con JWT válido. | `connection:ready` en ≤ 1 s. | I | RNF-PER-06a |
| QA-WS-02 | — | 1. Handshake con JWT de dispositivo. | `403`; sin conexión. | I | RNF-SEG-04 |
| QA-WS-03 | Tutor conectado. | 1. `child:subscribe` con un menor de otro grupo. | `error` con `FAM-004`; la suscripción no se crea. | I | RNF-SEG-04 |
| QA-WS-04 | Dos instancias del API; tutor conectado a la 2. | 1. El Worker registra un `DEPARTURE`. | `alert:movement_detected` llega al tutor en ≤ 2 s (p95 en 1 000 repeticiones). | I | RF-045, RNF-PER-06 |
| QA-WS-05 | Tutor conectado. | 1. Revocar al tutor del grupo. | El servidor cierra con `4403` en ≤ 2 s. | I | RF-012 |
| QA-WS-06 | Tutor conectado sin enviar latidos. | 1. Esperar 60 s. | Cierre con `4408`. | I | RNF-PER-06b |
| QA-WS-07 | Ciclo de verificación con el menor dentro. | 1. Ejecutar el ciclo. | **No** se emite `location:ping` (las lecturas de verificación no se muestran al tutor). | I | RF-046, RNF-PRI-04 |

### 5.6 Plan de aseguramiento de calidad

#### 5.6.1 Objetivo y alcance

Verificar que el MVP cumple los requisitos Must y Should de §1.2, los RNF del PRD §8 y los criterios de liberación CA-10, con énfasis en los tres riesgos con más impacto en la confianza del producto: falsas alertas, eventos perdidos y consumo de batería.

#### 5.6.2 Niveles de prueba

| Nivel | Responsable | Herramientas | Cuándo | Criterio de salida |
|---|---|---|---|---|
| Unitarias backend | Desarrollo | JUnit 5, Mockito, AssertJ | Cada commit (CI) | 100 % en verde; cobertura ≥ 80 % en `domain/rule` |
| Arquitectura | Desarrollo | ArchUnit | Cada commit | Sin violaciones de capas |
| Integración backend | Desarrollo | Testcontainers (PostgreSQL 18) | Cada PR | 100 % en verde |
| Unitarias Android | Desarrollo | JUnit 4, Mockito, Robolectric | Cada PR | 100 % en verde |
| Instrumentadas | QA | Espresso en emulador con Google Play | Cada PR a la rama principal | 100 % en verde |
| Extremo a extremo | QA | Maestro CLI contra el ambiente local o QA | Diario y antes de cada liberación | Escenarios críticos en verde |
| Escenarios de ubicación | QA | Emulador + `adb emu geo fix` + GPX (SDD §15.4) | Antes de cada liberación | QA-GPS, QA-NET y QA-CYC de nivel `E` en verde |
| Carga | QA + Desarrollo | Gatling (Apache 2.0) o k6 (AGPL-3.0, herramienta sin distribuir) | Antes del piloto y de cada aumento de escala | RNF-PER-01, RNF-PER-02, RNF-PER-06 y RNF-ESC-02 con 10× el promedio |
| Seguridad | Seguridad | OWASP ZAP (Apache 2.0), MobSF (GPL-3.0), revisión MASVS L1 | Antes del lanzamiento | Sin hallazgos críticos ni altos abiertos |
| Accesibilidad | QA | Accessibility Scanner, comprobaciones de Espresso | Antes del lanzamiento | WCAG 2.1 AA en pantallas del tutor |
| Campo (piloto) | Producto + QA | Dispositivos reales, Battery Historian, métricas del backend | 2 semanas, 20 familias (CA-10) | RNF-FIA-02, RNF-FIA-03, RNF-PER-03 |

#### 5.6.3 Ambientes

| Ambiente | Uso | Datos |
|---|---|---|
| Local | Desarrollo y escenarios rápidos (intervalos reducidos) | Semilla `seed.sh` |
| QA | Integración continua, Maestro, carga | Datos sintéticos; nunca datos reales de menores |
| UAT | Validación de producto | Familias de prueba internas con consentimiento |
| Producción (piloto) | Prueba de campo | Familias reales con consentimiento expreso |

#### 5.6.4 Matriz de dispositivos

| Fabricante | Modelos de referencia | Versiones de Android | Motivo |
|---|---|---|---|
| Google | Pixel (emulador y físico) | 10, 14, 16 | Comportamiento de referencia de Android |
| Samsung | Gama A (A1x, A3x) | 12, 14 | Mayor cuota en México; *Device Care* |
| Motorola | Serie g | 12, 14 | Alta cuota en México; Android cercano al de referencia |
| Xiaomi / Redmi | Redmi Note | 12, 14 | Capa agresiva de ahorro de energía (MIUI/HyperOS) |
| Oppo / Realme | Gama media | 12, 14 | Restricción de inicio automático |
| Huawei (sin Google Play Services) | Modelo reciente | — | Verificar el rechazo con `DEV-004` (R-15) |

#### 5.6.5 Criterios de entrada y salida de la liberación

**Entrada a QA:** build verde en CI (incluidas la verificación de licencias y de vulnerabilidades), migraciones aplicadas en QA y notas de cambio con los requisitos afectados.

**Salida (liberación del MVP):**

- Todos los casos de §5.2 a §5.5 en verde en su nivel.
- Ningún defecto abierto de severidad crítica o alta.
- Piloto: detección de llegadas ≥ 98 %, falsas alertas de salida ≤ 2 %, latencia de notificación p95 ≤ 60 s, batería ≤ 5 % diario.
- Inventario de licencias sin incumplimientos (RNF-LIC-01 a RNF-LIC-03).
- Revisión MASVS L1 y aviso de privacidad aprobados.

#### 5.6.6 Severidad de defectos

| Severidad | Definición | Ejemplos |
|---|---|---|
| Crítica | Pérdida de una alerta, alerta falsa sistemática o exposición de datos de un menor | `DEPARTURE` no emitida; tutor de otro grupo ve la ubicación |
| Alta | Funcionalidad Must degradada sin alternativa | Vinculación imposible en un fabricante de la matriz |
| Media | Funcionalidad Should degradada o Must con alternativa | WebSocket no reconecta (el push sí llega) |
| Baja | Defecto visual o de texto | Hora mal formateada |

#### 5.6.7 Métricas de calidad

| Métrica | Fuente | Meta |
|---|---|---|
| Tasa de detección de llegadas | Piloto: llegadas registradas vs. llegadas reportadas por las familias | ≥ 98 % |
| Falsas alertas de salida | Piloto: salidas marcadas como falsas por los tutores | ≤ 2 % |
| Respuesta a la verificación | `saferoute.verification.outcome` | ≥ 95 % `inside`/`outside` por fabricante sin capa agresiva |
| Latencia de notificación | `saferoute.notification.latency` | p95 ≤ 60 s |
| Latencia WebSocket | Métrica `saferoute.realtime.delivery_latency` | p95 ≤ 2 s |
| Sesiones sin fallos | ACRA | ≥ 99,5 % |
| Defectos escapados a producción | Seguimiento de incidencias | 0 críticos en el piloto |

#### 5.6.8 Trazabilidad requisito ↔ prueba

Cada prueba automatizada lleva la etiqueta del requisito que verifica (`@Tag("RF-044")` en JUnit 5; anotación `@Requirement("RF-044")` propia en Android), de modo que la suite puede filtrarse por requisito y el reporte de CI muestra la cobertura por requisito. Un requisito Must sin al menos una prueba etiquetada bloquea la liberación.
