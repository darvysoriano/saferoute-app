# PRD — SafeRoute

| Campo | Valor |
|---|---|
| Producto | SafeRoute (aplicación móvil) |
| Documento | Product Requirements Document |
| Versión | 0.2 |
| Estado | Borrador para revisión |
| Fecha | 2026-10-20 |
| Responsable | Darvy |

### Historial de cambios

| Versión | Fecha | Cambio |
|---|---|---|
| 0.1 | 2026-10-20 | Versión inicial. |
| 0.2 | 2026-10-20 | Restricción de arquitectura: Java y componentes gratuitos sin costo de licencia. El MVP se limita a Android (iOS pasa a fuera de alcance); se retira el inicio de sesión con Apple; se agregan RF-044 (verificación periódica), los requerimientos de licenciamiento y ejecución local (§8.8) y los riesgos R-13 a R-15. |

---

## Índice

1. [Visión del producto](#1-visión-del-producto)
2. [Objetivos](#2-objetivos)
3. [Alcance](#3-alcance)
4. [Usuarios y actores](#4-usuarios-y-actores)
5. [Funcionalidades detalladas](#5-funcionalidades-detalladas)
6. [Casos de uso](#6-casos-de-uso)
7. [Requerimientos funcionales](#7-requerimientos-funcionales)
8. [Requerimientos no funcionales](#8-requerimientos-no-funcionales)
9. [Criterios de aceptación](#9-criterios-de-aceptación)
10. [Métricas de éxito](#10-métricas-de-éxito)
11. [Riesgos del proyecto](#11-riesgos-del-proyecto)
12. [Supuestos y preguntas abiertas](#12-supuestos-y-preguntas-abiertas)
13. [Glosario](#13-glosario)

---

## 1. Visión del producto

### 1.1 Declaración de visión

SafeRoute da a padres y tutores la certeza de que sus hijos llegaron a donde debían llegar, sin tener que llamarles ni vigilar un mapa. El tutor registra un destino; SafeRoute avisa cuando el menor llega y alerta si después se aleja de ese lugar.

### 1.2 Problema

- Los tutores no tienen una confirmación confiable y automática de que el menor llegó a la escuela, a una actividad o a casa.
- Las alternativas actuales (llamadas, mensajes, apps de rastreo continuo) dependen de que el menor responda, o bien exponen su ubicación en todo momento, lo que genera fricción con el menor y preocupaciones de privacidad.
- Una vez que el menor llegó, el tutor tampoco se entera si se retira del lugar antes de lo esperado.

### 1.3 Propuesta de valor

| Para | Valor |
|---|---|
| Tutor | Notificación automática de llegada y alerta de salida, sin intervención del menor. |
| Menor | Sin acciones manuales; la ubicación se usa para eventos concretos, no como vigilancia constante. |
| Familia | Varios tutores pueden recibir los mismos avisos sobre los mismos menores. |

### 1.4 Principios de producto

1. **Eventos, no vigilancia.** El valor central son los avisos de llegada y salida; la ubicación en vivo es secundaria y queda registrada como consulta.
2. **Cero fricción para el menor.** Tras la vinculación inicial, el dispositivo del menor no requiere interacción.
3. **Pocas alertas, pero confiables.** Una falsa alarma cuesta más confianza que un aviso con un minuto de retraso.
4. **Privacidad por diseño.** Mínimo de datos, retención limitada y consentimiento explícito.

---

## 2. Objetivos

### 2.1 Objetivos de negocio

| ID | Objetivo | Indicador asociado |
|---|---|---|
| OB-01 | Lanzar un MVP funcional en Android construido íntegramente con componentes sin costo de licencia. | Publicación en Google Play e inventario de licencias aprobado. |
| OB-02 | Lograr la adopción inicial en familias con hijos en edad escolar. | Familias activas a 90 días del lanzamiento (ver §10). |
| OB-03 | Construir confianza en la marca por la fiabilidad de los avisos. | Tasa de falsas alertas y calificación en tiendas. |

### 2.2 Objetivos de producto

| ID | Objetivo |
|---|---|
| OP-01 | Notificar al tutor la llegada del menor a un destino registrado en menos de 2 minutos (p95). |
| OP-02 | Alertar al tutor cuando el menor sale de un destino al que ya había llegado, dentro de su ventana de vigilancia. |
| OP-03 | Alertar al tutor cuando el menor no llega a un destino a la hora esperada. |
| OP-04 | Mantener el consumo de batería en el dispositivo del menor dentro de un umbral aceptable (ver RNF-PER-03). |
| OP-05 | Cumplir con la normativa de protección de datos aplicable a menores. |

---

## 3. Alcance

### 3.1 Dentro del alcance (MVP)

- Registro e inicio de sesión de tutores.
- Grupo familiar con uno o varios tutores y uno o varios menores.
- Vinculación del dispositivo del menor mediante código o QR.
- Registro de destinos con ubicación, radio y horario (único o recurrente).
- Notificación de llegada.
- Alerta de salida posterior a la llegada.
- Alerta de no llegada a la hora esperada.
- Alertas de estado del dispositivo del menor (permisos de ubicación revocados, sin conexión, batería baja).
- Consulta de ubicación actual bajo demanda.
- Historial de eventos.
- Configuración de preferencias de notificación.
- Gestión de consentimiento y eliminación de cuenta y datos.

### 3.2 Fuera del alcance (MVP)

- Versión para iOS (requiere el Apple Developer Program, de pago, para notificaciones push y distribución).
- Dispositivos Android sin Google Play Services.
- Rastreo continuo de rutas y reproducción de trayectos.
- Botón de pánico / SOS del menor.
- Chat o mensajería entre tutor y menor.
- Soporte para relojes inteligentes y dispositivos GPS dedicados.
- Versión web para tutores.
- Integraciones con escuelas o transporte escolar.
- Planes de pago y monetización.

Estos elementos se evaluarán para versiones posteriores (ver §12).

### 3.3 Plataformas

| Plataforma | Versión mínima | Requisito adicional |
|---|---|---|
| Android | 10 (API 29) | Google Play Services (geocercas y notificaciones push) |

Tanto la app del tutor como la del menor son Android en el MVP. La versión para iOS se evaluará después (ver PA-07).

### 3.4 Restricción de arquitectura

Todo el sistema (app móvil, backend, base de datos, notificaciones y mapas) se construye con **Java** y con componentes **de código abierto o gratuitos, sin costo de licencia**, de modo que pueda ejecutarse completo en un ambiente local para pruebas. El detalle de componentes y licencias está en el SDD.

---

## 4. Usuarios y actores

| Actor | Descripción |
|---|---|
| **Tutor principal** | Adulto que crea el grupo familiar, registra menores y destinos, e invita a otros tutores. Otorga el consentimiento sobre el tratamiento de datos del menor. |
| **Tutor secundario** | Adulto invitado al grupo. Recibe avisos y consulta información; no puede eliminar el grupo ni a otros tutores. |
| **Menor** | Usuario del dispositivo vinculado. Su app opera en segundo plano y muestra únicamente el estado de vinculación y un aviso de transparencia. |
| **Sistema SafeRoute** | Backend que procesa eventos, evalúa reglas y envía notificaciones. |
| **Servicio de notificación** | Firebase Cloud Messaging (FCM), gratuito. |

### 4.1 Personas

- **Laura, 38 años, madre trabajadora.** Su hijo de 11 años va solo a la escuela en transporte público. Quiere saber que llegó sin interrumpir su trabajo ni hacer que él le escriba.
- **Jorge, 45 años, padre separado.** Comparte la custodia y quiere recibir los mismos avisos que la madre sin depender de ella para enterarse.
- **Sofía, 13 años, hija.** Acepta que sus padres sepan si llegó a la escuela, pero no quiere sentirse vigilada todo el día.

---

## 5. Funcionalidades detalladas

Prioridad: **P0** = imprescindible para el MVP, **P1** = deseable en el MVP, **P2** = posterior.

### F-01 Gestión de cuenta del tutor — P0

- Registro con correo electrónico y contraseña, o con cuenta de Google.
- Verificación de correo.
- Recuperación de contraseña.
- Edición de perfil (nombre, foto, teléfono opcional).
- Eliminación de cuenta con borrado de datos asociados.

### F-02 Grupo familiar — P0

- Al registrarse, el tutor principal crea su grupo familiar.
- El tutor principal invita a tutores secundarios mediante enlace o correo.
- El tutor principal puede revocar el acceso de un tutor secundario.
- Un tutor puede pertenecer a más de un grupo familiar (p. ej., familias con custodia compartida).

### F-03 Perfil y vinculación del menor — P0

- El tutor crea el perfil del menor (nombre o alias, foto opcional, edad).
- El sistema genera un código de vinculación de un solo uso, con vigencia de 15 minutos, mostrable como texto y como QR.
- En el dispositivo del menor se instala SafeRoute, se elige el modo "Menor" y se introduce o escanea el código.
- La app del menor guía el otorgamiento de permisos: ubicación "Permitir todo el tiempo", notificaciones y exclusión de optimización de batería (Android).
- La app del menor muestra un aviso permanente y visible de que su ubicación se usa para avisos de llegada y salida.
- Un tutor puede desvincular el dispositivo en cualquier momento.

### F-04 Registro de destinos — P0

- El tutor busca una dirección o selecciona un punto en el mapa.
- Define un nombre (p. ej., "Escuela"), un radio de geocerca entre 100 m y 1 000 m (por defecto 150 m) y un icono.
- Asigna el destino a uno o varios menores.
- Define el horario de vigilancia:
  - **Hora esperada de llegada** (opcional): activa la alerta de no llegada.
  - **Ventana de vigilancia**: intervalo durante el cual se evalúan llegada y salida.
  - **Recurrencia**: única fecha, o días de la semana.
- Puede pausar, editar o eliminar un destino.
- Límite en el MVP: 10 destinos activos por menor.

### F-05 Notificación de llegada — P0

- Cuando el menor entra a la geocerca de un destino dentro de su ventana de vigilancia, el sistema notifica a todos los tutores del grupo.
- Contenido: nombre del menor, nombre del destino, hora de llegada.
- Para evitar avisos espurios, la llegada se confirma cuando el menor permanece dentro de la geocerca al menos 60 s (tiempo de permanencia configurable a nivel de sistema).
- Solo se emite una notificación de llegada por destino y ventana de vigilancia.

### F-06 Alerta de salida — P0

- Una vez confirmada la llegada, si el menor sale de la geocerca antes de que termine la ventana de vigilancia, el sistema emite una alerta de salida.
- Para reducir falsas alertas por imprecisión del GPS, la salida se confirma cuando el menor está fuera del radio más un margen de histéresis (por defecto 50 m) durante al menos 3 minutos.
- Contenido: nombre del menor, destino, hora de salida y acceso directo a la ubicación actual.
- Si el menor regresa al destino, se notifica el reingreso.
- Las salidas posteriores al fin de la ventana no generan alerta.

### F-07 Alerta de no llegada — P0

- Si el destino tiene hora esperada de llegada y el menor no ha llegado transcurrido un margen de tolerancia (por defecto 15 minutos, configurable por destino entre 5 y 60), el sistema alerta a los tutores.
- Contenido: nombre del menor, destino, hora esperada y última ubicación conocida con su antigüedad.

### F-08 Estado del dispositivo del menor — P0

El sistema alerta a los tutores cuando:

- Se revoca o reduce el permiso de ubicación.
- El dispositivo no reporta durante más de 30 minutos dentro de una ventana de vigilancia activa.
- La batería baja del 15 % dentro de una ventana de vigilancia activa.
- Se desinstala la app o se cierra la sesión del menor (detectado por ausencia de reportes y por invalidación del token de notificaciones).

### F-09 Ubicación bajo demanda — P1

- El tutor puede solicitar la ubicación actual del menor.
- El sistema solicita una lectura fresca al dispositivo; si no se obtiene en 30 s, muestra la última ubicación conocida con su antigüedad.
- Cada consulta queda registrada en el historial y es visible para el resto de tutores del grupo.

### F-10 Historial de eventos — P0

- Lista cronológica de eventos por menor: llegadas, salidas, reingresos, no llegadas, alertas de dispositivo y consultas de ubicación.
- Filtros por menor, destino, tipo de evento y rango de fechas.
- Retención de 30 días.

### F-11 Preferencias de notificación — P1

- Activar o desactivar cada tipo de aviso por destino y por tutor.
- Las alertas críticas (salida, no llegada, permisos revocados) se pueden marcar como prioritarias para atravesar el modo "No molestar" donde el sistema operativo lo permita.

### F-12 Privacidad y consentimiento — P0

- Aviso de privacidad y términos aceptados por el tutor al registrarse.
- Consentimiento explícito del tutor para el tratamiento de datos de ubicación del menor, registrado con fecha y versión del aviso.
- Pantalla de transparencia en la app del menor.
- Exportación de los datos del grupo familiar a solicitud del tutor principal.
- Eliminación de cuenta, perfiles de menores y datos de ubicación.

---

## 6. Casos de uso

### CU-01 Registrar cuenta y crear grupo familiar

| Campo | Detalle |
|---|---|
| Actor principal | Tutor principal |
| Precondiciones | La app está instalada en el dispositivo del tutor. |
| Postcondiciones | Existe una cuenta verificada y un grupo familiar del que el tutor es principal. |

**Flujo principal**

1. El tutor abre la app y selecciona "Soy tutor".
2. Elige registrarse con correo o con proveedor de identidad.
3. Acepta el aviso de privacidad y los términos.
4. El sistema crea la cuenta y envía un correo de verificación (solo registro por correo).
5. El tutor verifica su correo.
6. El sistema crea el grupo familiar y lleva al tutor al asistente de alta del primer menor.

**Flujos alternos**

- 2a. El correo ya está registrado: el sistema ofrece iniciar sesión o recuperar la contraseña.
- 5a. El tutor no verifica en 24 h: el enlace expira y puede solicitar uno nuevo.

### CU-02 Vincular el dispositivo del menor

| Campo | Detalle |
|---|---|
| Actores | Tutor (principal o secundario), Menor |
| Precondiciones | El tutor tiene sesión iniciada y existe el perfil del menor. |
| Postcondiciones | El dispositivo del menor está vinculado y con permisos suficientes para reportar ubicación en segundo plano. |

**Flujo principal**

1. El tutor selecciona el menor y elige "Vincular dispositivo".
2. El sistema genera un código de un solo uso y lo muestra como texto y QR.
3. En el dispositivo del menor se instala la app y se selecciona "Soy menor".
4. Se escanea el QR o se introduce el código.
5. La app guía el otorgamiento de permisos de ubicación "Permitir todo el tiempo", notificaciones y exclusión de optimización de batería.
6. El sistema confirma la vinculación en ambos dispositivos.

**Flujos alternos**

- 4a. El código expiró o ya fue usado: se informa el error y el tutor genera uno nuevo.
- 5a. No se otorga ubicación "Permitir todo el tiempo": la vinculación queda en estado "Incompleta", la app del menor muestra cómo corregirlo y el tutor recibe un aviso.
- 6a. El menor ya tenía un dispositivo vinculado: el sistema pide confirmación para reemplazarlo y desvincula el anterior.

### CU-03 Registrar un destino

| Campo | Detalle |
|---|---|
| Actor principal | Tutor |
| Precondiciones | Existe al menos un menor en el grupo. |
| Postcondiciones | El destino está activo y sincronizado con el dispositivo de cada menor asignado. |

**Flujo principal**

1. El tutor selecciona "Nuevo destino".
2. Busca una dirección o coloca un marcador en el mapa.
3. Ajusta el radio de la geocerca, visible sobre el mapa.
4. Asigna nombre e icono.
5. Selecciona los menores a los que aplica.
6. Define ventana de vigilancia, hora esperada de llegada (opcional) y recurrencia.
7. Guarda. El sistema valida y sincroniza la geocerca con los dispositivos.

**Flujos alternos**

- 6a. La hora esperada queda fuera de la ventana de vigilancia: el sistema muestra un error de validación.
- 7a. El menor alcanzó el límite de destinos activos: el sistema lo informa y sugiere pausar otro destino.
- 7b. El dispositivo del menor está sin conexión: el destino se guarda y se sincroniza cuando el dispositivo se reconecte; el tutor ve el estado "Pendiente de sincronizar".

### CU-04 Recibir notificación de llegada

| Campo | Detalle |
|---|---|
| Actor principal | Sistema; receptor: tutores del grupo |
| Precondiciones | Destino activo, ventana de vigilancia en curso, dispositivo vinculado con permisos completos. |
| Postcondiciones | Evento de llegada registrado y notificado. |

**Flujo principal**

1. El dispositivo del menor detecta la entrada a la geocerca.
2. Tras el tiempo de permanencia mínimo, el dispositivo envía el evento al backend.
3. El backend valida que el evento corresponde a una ventana activa y que no hay una llegada previa en esa ventana.
4. El backend registra el evento y envía la notificación a todos los tutores con avisos de llegada activados.
5. El tutor toca la notificación y ve el detalle del evento.

**Flujos alternos**

- 2a. El dispositivo no tiene conexión: el evento se encola localmente con su marca de tiempo original y se envía al reconectar. La notificación indica la hora real de llegada.
- 3a. El evento llega fuera de la ventana: se registra en el historial sin notificar.

### CU-05 Recibir alerta de salida

| Campo | Detalle |
|---|---|
| Actor principal | Sistema; receptor: tutores del grupo |
| Precondiciones | Llegada confirmada en la ventana activa. |
| Postcondiciones | Evento de salida registrado y alertado. |

**Flujo principal**

1. El dispositivo detecta que el menor está fuera del radio más el margen de histéresis.
2. Tras 3 minutos fuera, el dispositivo envía el evento de salida.
3. El backend registra el evento y envía una alerta prioritaria a los tutores.
4. El tutor abre la alerta y ve la ubicación actual del menor.

**Flujos alternos**

- 2a. El menor regresa antes de los 3 minutos: no se emite evento.
- 4a. El menor vuelve después a la geocerca: se notifica el reingreso.

### CU-06 Recibir alerta de no llegada

| Campo | Detalle |
|---|---|
| Actor principal | Sistema; receptor: tutores del grupo |
| Precondiciones | Destino con hora esperada de llegada, ventana activa. |
| Postcondiciones | Evento de no llegada registrado y alertado. |

**Flujo principal**

1. El backend evalúa en la hora esperada más la tolerancia si existe llegada confirmada.
2. Si no existe, solicita una ubicación fresca al dispositivo.
3. Envía la alerta con la última ubicación disponible y su antigüedad.

**Flujos alternos**

- 2a. La ubicación fresca muestra al menor dentro de la geocerca: se registra la llegada en lugar de la alerta (corrige eventos perdidos).
- 2b. El dispositivo no responde: la alerta indica que no fue posible obtener la ubicación actual.

### CU-07 Consultar ubicación bajo demanda

| Campo | Detalle |
|---|---|
| Actor principal | Tutor |
| Precondiciones | Dispositivo del menor vinculado. |
| Postcondiciones | Consulta registrada en el historial. |

**Flujo principal**

1. El tutor selecciona al menor y elige "Ver ubicación actual".
2. El backend envía una solicitud silenciosa al dispositivo.
3. El dispositivo responde con su ubicación y precisión.
4. La app muestra el punto en el mapa con su precisión y hora.

**Flujos alternos**

- 3a. Sin respuesta en 30 s: se muestra la última ubicación conocida con su antigüedad.

### CU-08 Invitar a un tutor secundario

| Campo | Detalle |
|---|---|
| Actor principal | Tutor principal |
| Precondiciones | Grupo familiar creado. |
| Postcondiciones | El invitado pertenece al grupo con rol de tutor secundario. |

**Flujo principal**

1. El tutor principal elige "Invitar tutor" y comparte un enlace o introduce un correo.
2. El invitado abre el enlace, inicia sesión o se registra y acepta la invitación.
3. El sistema lo agrega al grupo y notifica al tutor principal.

**Flujos alternos**

- 2a. La invitación expiró (7 días): el invitado ve un mensaje y debe pedir una nueva.

### CU-09 Eliminar cuenta y datos

| Campo | Detalle |
|---|---|
| Actor principal | Tutor principal |
| Precondiciones | Sesión iniciada. |
| Postcondiciones | Cuenta, grupo, perfiles de menores y datos de ubicación eliminados. |

**Flujo principal**

1. El tutor solicita eliminar su cuenta desde la configuración.
2. El sistema advierte las consecuencias y pide reautenticación.
3. El sistema desvincula los dispositivos, notifica a los tutores secundarios y elimina los datos.

**Flujos alternos**

- 3a. Si el grupo tiene tutores secundarios, el sistema ofrece transferir el rol de tutor principal antes de eliminar.

---

## 7. Requerimientos funcionales

### 7.1 Cuenta y autenticación

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-001 | El sistema debe permitir el registro de tutores con correo y contraseña. | P0 | F-01 |
| RF-002 | El sistema debe permitir el registro e inicio de sesión con cuenta de Google. | P0 | F-01 |
| RF-003 | El sistema debe verificar el correo electrónico antes de permitir el alta de menores. | P0 | F-01 |
| RF-004 | El sistema debe permitir recuperar la contraseña mediante enlace enviado al correo. | P0 | F-01 |
| RF-005 | El sistema debe exigir contraseñas de al menos 8 caracteres con letras y números. | P0 | F-01 |
| RF-006 | El sistema debe permitir eliminar la cuenta tras reautenticación. | P0 | F-01, F-12 |

### 7.2 Grupo familiar y menores

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-010 | El sistema debe crear un grupo familiar al registrar un tutor principal. | P0 | F-02 |
| RF-011 | El tutor principal debe poder invitar tutores secundarios mediante enlace con vigencia de 7 días. | P0 | F-02 |
| RF-012 | El tutor principal debe poder revocar el acceso de un tutor secundario. | P0 | F-02 |
| RF-013 | Un tutor debe poder pertenecer a varios grupos familiares. | P1 | F-02 |
| RF-014 | El sistema debe permitir crear, editar y eliminar perfiles de menores. | P0 | F-03 |
| RF-015 | El sistema debe generar códigos de vinculación de un solo uso, con vigencia de 15 minutos, como texto y QR. | P0 | F-03 |
| RF-016 | Un menor debe tener como máximo un dispositivo vinculado a la vez. | P0 | F-03 |
| RF-017 | La app del menor debe verificar y guiar el otorgamiento de los permisos de ubicación en segundo plano, notificaciones y exclusión de optimización de batería. | P0 | F-03 |
| RF-018 | La app del menor debe mostrar permanentemente que su ubicación se usa para avisos de llegada y salida. | P0 | F-03, F-12 |
| RF-019 | Cualquier tutor del grupo debe poder desvincular el dispositivo de un menor. | P0 | F-03 |

### 7.3 Destinos

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-020 | El tutor debe poder crear un destino por búsqueda de dirección o selección en el mapa. | P0 | F-04 |
| RF-021 | El radio de la geocerca debe ser configurable entre 100 m y 1 000 m, con 150 m por defecto. | P0 | F-04 |
| RF-022 | Un destino debe poder asignarse a uno o varios menores. | P0 | F-04 |
| RF-023 | Un destino debe tener una ventana de vigilancia y, opcionalmente, una hora esperada de llegada contenida en ella. | P0 | F-04 |
| RF-024 | Un destino debe admitir recurrencia por fecha única o por días de la semana. | P0 | F-04 |
| RF-025 | El tutor debe poder pausar, reanudar, editar y eliminar destinos. | P0 | F-04 |
| RF-026 | El sistema debe limitar a 10 los destinos activos por menor. | P0 | F-04 |
| RF-027 | Los cambios en destinos deben sincronizarse con los dispositivos de los menores asignados; si no hay conexión, deben quedar pendientes y mostrarse como tales. | P0 | F-04 |
| RF-028 | Los horarios deben evaluarse en la zona horaria del destino. | P0 | F-04 |

### 7.4 Eventos y notificaciones

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-030 | El sistema debe confirmar una llegada cuando el menor permanece dentro de la geocerca al menos 60 s durante la ventana de vigilancia. | P0 | F-05 |
| RF-031 | El sistema debe emitir como máximo una notificación de llegada por destino, menor y ventana. | P0 | F-05 |
| RF-032 | El sistema debe confirmar una salida cuando, tras una llegada, el menor permanece fuera del radio más 50 m de histéresis durante al menos 3 minutos dentro de la ventana. | P0 | F-06 |
| RF-033 | El sistema debe notificar el reingreso al destino tras una salida. | P1 | F-06 |
| RF-034 | El sistema debe emitir una alerta de no llegada si no hay llegada confirmada al cumplirse la hora esperada más la tolerancia configurada (5–60 min, 15 por defecto). | P0 | F-07 |
| RF-035 | Antes de emitir una alerta de no llegada, el sistema debe solicitar una ubicación fresca y registrar la llegada en lugar de alertar si el menor está dentro de la geocerca. | P0 | F-07 |
| RF-036 | El dispositivo del menor debe encolar eventos sin conexión con su marca de tiempo original y enviarlos al reconectar. | P0 | F-05, F-06 |
| RF-037 | El sistema debe procesar los eventos de forma idempotente, de modo que los reenvíos no produzcan notificaciones duplicadas. | P0 | F-05, F-06 |
| RF-038 | Las notificaciones deben incluir nombre del menor, nombre del destino, tipo de evento y hora real del evento. | P0 | F-05, F-06, F-07 |
| RF-039 | Las notificaciones deben enviarse a todos los tutores del grupo que tengan ese tipo de aviso activado. | P0 | F-05, F-06, F-07, F-11 |

### 7.5 Estado del dispositivo

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-040 | El sistema debe alertar a los tutores si se revoca o reduce el permiso de ubicación del dispositivo del menor. | P0 | F-08 |
| RF-041 | El sistema debe alertar si el dispositivo no reporta durante más de 30 minutos dentro de una ventana activa. | P0 | F-08 |
| RF-042 | El sistema debe alertar si la batería del dispositivo baja del 15 % dentro de una ventana activa, como máximo una vez por ventana. | P1 | F-08 |
| RF-043 | La app del tutor debe mostrar el estado de cada dispositivo vinculado: permisos, última conexión y nivel de batería. | P0 | F-08 |
| RF-044 | Durante una ventana de vigilancia activa, el sistema debe verificar cada 20 minutos la posición del menor respecto del destino, para detectar llegadas, salidas y regresos que la geocerca no haya reportado, y dispositivos sin respuesta. | P0 | F-05, F-06, F-08 |

### 7.6 Ubicación, historial y preferencias

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-050 | El tutor debe poder solicitar la ubicación actual del menor; si no se obtiene en 30 s, se muestra la última conocida con su antigüedad. | P1 | F-09 |
| RF-051 | Cada consulta de ubicación debe registrarse en el historial con el tutor que la realizó. | P1 | F-09 |
| RF-052 | El sistema debe mostrar un historial cronológico de eventos por menor con filtros por menor, destino, tipo y fechas. | P0 | F-10 |
| RF-053 | El tutor debe poder activar o desactivar cada tipo de aviso por destino. | P1 | F-11 |
| RF-054 | Las alertas de salida, no llegada y permisos revocados deben enviarse con prioridad alta y, donde el sistema operativo lo permita y el tutor lo autorice, como alertas críticas. | P1 | F-11 |

### 7.7 Privacidad

| ID | Requerimiento | Prioridad | Funcionalidad |
|---|---|---|---|
| RF-060 | El sistema debe registrar el consentimiento del tutor con fecha, hora y versión del aviso de privacidad. | P0 | F-12 |
| RF-061 | El sistema debe solicitar nuevo consentimiento cuando cambie la versión del aviso de privacidad. | P0 | F-12 |
| RF-062 | El tutor principal debe poder exportar los datos del grupo en formato legible por máquina (JSON). | P1 | F-12 |
| RF-063 | Al eliminar una cuenta o un menor, el sistema debe eliminar sus datos personales y de ubicación en un plazo máximo de 30 días, incluidas las copias de respaldo. | P0 | F-12 |

---

## 8. Requerimientos no funcionales

### 8.1 Rendimiento

| ID | Requerimiento |
|---|---|
| RNF-PER-01 | Latencia del evento confirmado en el dispositivo hasta la entrega de la notificación al tutor: p95 ≤ 60 s y p99 ≤ 120 s con conexión disponible. |
| RNF-PER-02 | Tiempo de respuesta de la API para operaciones de lectura: p95 ≤ 300 ms; escritura: p95 ≤ 500 ms. |
| RNF-PER-03 | Consumo adicional de batería en el dispositivo del menor: ≤ 5 % diario en un día escolar típico (2 destinos, 2 ventanas de vigilancia). |
| RNF-PER-04 | Arranque en frío de la app: ≤ 2,5 s en dispositivos de gama media. |
| RNF-PER-05 | Consumo de datos móviles del dispositivo del menor: ≤ 10 MB mensuales en uso típico. |

### 8.2 Fiabilidad y disponibilidad

| ID | Requerimiento |
|---|---|
| RNF-FIA-01 | Disponibilidad mensual del backend: ≥ 99,5 %. |
| RNF-FIA-02 | Tasa de detección de llegadas en condiciones normales (dispositivo con permisos, batería y conexión): ≥ 98 %. |
| RNF-FIA-03 | Tasa de falsas alertas de salida: ≤ 2 % de las alertas emitidas. |
| RNF-FIA-04 | Ningún evento aceptado por el backend debe perderse; el procesamiento debe ser "al menos una vez" con deduplicación. |
| RNF-FIA-05 | Sesiones sin fallos (crash-free): ≥ 99,5 % en Android. |
| RNF-FIA-06 | RPO ≤ 1 h y RTO ≤ 4 h ante desastre. |

### 8.3 Seguridad

| ID | Requerimiento |
|---|---|
| RNF-SEG-01 | Toda comunicación debe usar TLS 1.2 o superior; la app debe aplicar certificate pinning hacia el backend. |
| RNF-SEG-02 | Los datos de ubicación y personales deben cifrarse en reposo (AES-256 o equivalente del proveedor). |
| RNF-SEG-03 | La autenticación debe usar tokens de acceso de corta duración (≤ 15 min) con tokens de refresco rotativos. |
| RNF-SEG-04 | La autorización debe verificar en cada solicitud que el tutor pertenece al grupo del menor consultado. |
| RNF-SEG-05 | Las contraseñas deben almacenarse con un algoritmo de derivación robusto (Argon2id o bcrypt). |
| RNF-SEG-06 | Los tokens y credenciales en el dispositivo deben cifrarse con claves del Android Keystore. |
| RNF-SEG-07 | Límite de intentos de inicio de sesión y de canje de códigos de vinculación para prevenir fuerza bruta. |
| RNF-SEG-08 | Los registros de la aplicación no deben contener coordenadas ni datos personales en claro. |
| RNF-SEG-09 | El producto debe pasar una revisión de seguridad (OWASP MASVS L1) antes del lanzamiento. |

### 8.4 Privacidad y cumplimiento

| ID | Requerimiento |
|---|---|
| RNF-PRI-01 | Cumplir con la Ley Federal de Protección de Datos Personales en Posesión de los Particulares (México), incluidos aviso de privacidad y derechos ARCO. |
| RNF-PRI-02 | Tratar la ubicación del menor como dato personal sensible, con consentimiento expreso del tutor. |
| RNF-PRI-03 | Retener ubicaciones puntuales un máximo de 30 días; conservar solo eventos agregados más allá de ese plazo. |
| RNF-PRI-04 | Recolectar ubicación del menor únicamente para evaluar geocercas activas, atender consultas bajo demanda y ventanas de vigilancia; no rastrear de forma continua fuera de ellas. |
| RNF-PRI-05 | Cumplir las políticas de Google Play sobre ubicación en segundo plano y datos de menores, incluida la justificación del permiso de ubicación "Todo el tiempo". |
| RNF-PRI-06 | No compartir datos con terceros con fines publicitarios ni incluir SDK de publicidad. |
| RNF-PRI-07 | Las notificaciones push no deben transportar datos personales a través del proveedor de push: el contenido visible se construye en el dispositivo. |

### 8.5 Escalabilidad

| ID | Requerimiento |
|---|---|
| RNF-ESC-01 | La arquitectura debe soportar 50 000 familias activas sin rediseño, con escalado horizontal. |
| RNF-ESC-02 | El backend debe absorber picos de eventos en horarios de entrada escolar (10× el promedio) sin degradar RNF-PER-01. |

### 8.6 Usabilidad y accesibilidad

| ID | Requerimiento |
|---|---|
| RNF-USA-01 | Un tutor nuevo debe completar registro, alta de menor, vinculación y primer destino en ≤ 10 minutos. |
| RNF-USA-02 | Cumplir WCAG 2.1 nivel AA en las pantallas del tutor: contraste, tamaños de toque, soporte de lectores de pantalla y texto dinámico. |
| RNF-USA-03 | Idioma inicial: español (México); la interfaz debe estar preparada para internacionalización. |
| RNF-USA-04 | Las notificaciones deben ser comprensibles sin abrir la app. |

### 8.7 Mantenibilidad y observabilidad

| ID | Requerimiento |
|---|---|
| RNF-MAN-01 | Cobertura de pruebas unitarias ≥ 80 % en la lógica de evaluación de geocercas y reglas de eventos. |
| RNF-MAN-02 | Métricas, trazas y registros centralizados con alertas operativas sobre latencia de notificaciones y tasa de error. |
| RNF-MAN-03 | Telemetría anónima del dispositivo (estado de permisos, precisión, frecuencia de reportes) para diagnosticar fallos de detección. |
| RNF-MAN-04 | Despliegues automatizados con capacidad de reversión y feature flags para activar funcionalidades de forma gradual. |

### 8.8 Licenciamiento y ejecución local

| ID | Requerimiento |
|---|---|
| RNF-LIC-01 | Todo componente de software del sistema debe tener licencia de código abierto aprobada por OSI o ser un servicio gratuito sin costo de licencia ni de uso en el volumen del MVP. |
| RNF-LIC-02 | El proyecto debe mantener un inventario de componentes con su licencia y sus obligaciones (atribución, avisos), revisado en cada cambio de dependencias. |
| RNF-LIC-03 | No se admiten componentes con licencias de pago por uso comercial, de código fuente disponible no abierto (BSL, FSL, SSPL) ni distribuciones del JDK sujetas a licencia comercial. |
| RNF-LIC-04 | La app debe mostrar las atribuciones exigidas por las licencias, incluidos "© OpenStreetMap contributors" en el mapa y la pantalla de licencias de código abierto. |
| RNF-OPE-01 | El sistema completo (backend, base de datos, mapas, búsqueda de direcciones y correo) debe poder ejecutarse en un equipo de desarrollo con un solo comando de contenedores, y la app en un emulador Android, sin cuentas de pago. |
| RNF-OPE-02 | La preparación del ambiente local desde cero debe completarse en ≤ 30 minutos, sin contar descargas. |
| RNF-OPE-03 | El ambiente local debe incluir datos de prueba y escenarios reproducibles de llegada, salida y no llegada mediante ubicación simulada en el emulador. |

---

## 9. Criterios de aceptación

Los criterios usan el formato *Dado / Cuando / Entonces*.

### CA-01 Vinculación del dispositivo (F-03)

- **Dado** un código de vinculación vigente, **cuando** se introduce en la app del menor, **entonces** el dispositivo queda vinculado y ambos dispositivos muestran la confirmación en menos de 5 s.
- **Dado** un código usado o con más de 15 minutos, **cuando** se introduce, **entonces** se rechaza con un mensaje que indica generar uno nuevo.
- **Dado** que el menor no otorga ubicación "Permitir todo el tiempo", **cuando** termina el asistente, **entonces** la vinculación queda como "Incompleta" y el tutor recibe un aviso.

### CA-02 Registro de destino (F-04)

- **Dado** un tutor con un menor vinculado, **cuando** guarda un destino válido, **entonces** el destino aparece activo y la geocerca se registra en el dispositivo del menor en menos de 60 s si este tiene conexión.
- **Dado** un radio menor a 100 m o mayor a 1 000 m, **cuando** el tutor intenta guardar, **entonces** el sistema muestra un error de validación y no guarda.
- **Dado** un menor con 10 destinos activos, **cuando** el tutor intenta activar otro, **entonces** el sistema lo impide e informa el límite.

### CA-03 Notificación de llegada (F-05)

- **Dado** un destino con ventana activa, **cuando** el menor permanece dentro de la geocerca 60 s, **entonces** todos los tutores con el aviso activo reciben la notificación de llegada con p95 ≤ 60 s.
- **Dado** que ya se notificó la llegada en la ventana actual, **cuando** el menor sale y vuelve a entrar, **entonces** no se emite una segunda notificación de llegada (sí de reingreso, si aplica).
- **Dado** que el menor entra a la geocerca fuera de la ventana, **entonces** el evento se registra en el historial sin notificar.
- **Dado** que el dispositivo estuvo sin conexión al llegar, **cuando** se reconecta, **entonces** la notificación muestra la hora real de llegada.

### CA-04 Alerta de salida (F-06)

- **Dado** una llegada confirmada, **cuando** el menor permanece fuera del radio más 50 m durante 3 minutos dentro de la ventana, **entonces** los tutores reciben una alerta de salida de prioridad alta con acceso a la ubicación actual.
- **Dado** una llegada confirmada, **cuando** el menor sale y regresa antes de 3 minutos, **entonces** no se emite alerta.
- **Dado** que la ventana terminó, **cuando** el menor sale, **entonces** no se emite alerta.
- **Dado** un escenario de prueba de campo con el menor estático dentro del destino durante 6 h, **entonces** no se emite ninguna alerta de salida.

### CA-05 Alerta de no llegada (F-07)

- **Dado** un destino con hora esperada 07:30 y tolerancia de 15 min, **cuando** a las 07:45 no hay llegada confirmada y la ubicación fresca está fuera de la geocerca, **entonces** los tutores reciben la alerta con la última ubicación y su antigüedad.
- **Dado** el mismo escenario, **cuando** la ubicación fresca está dentro de la geocerca, **entonces** se registra la llegada y no se emite la alerta de no llegada.

### CA-06 Estado del dispositivo (F-08)

- **Dado** un dispositivo vinculado, **cuando** el menor revoca el permiso de ubicación, **entonces** los tutores reciben una alerta en menos de 15 minutos.
- **Dado** una ventana activa, **cuando** el dispositivo no reporta durante 30 minutos, **entonces** los tutores reciben una alerta de "sin conexión".

### CA-07 Ubicación bajo demanda (F-09)

- **Dado** un dispositivo con conexión, **cuando** el tutor solicita la ubicación, **entonces** se muestra una ubicación con menos de 60 s de antigüedad en ≤ 30 s.
- **Dado** un dispositivo sin respuesta, **cuando** pasan 30 s, **entonces** se muestra la última ubicación conocida con su antigüedad.
- **Dado** cualquier consulta, **entonces** queda registrada en el historial con el nombre del tutor que la hizo.

### CA-08 Historial (F-10)

- **Dado** eventos de los últimos 30 días, **cuando** el tutor filtra por menor y tipo, **entonces** solo se muestran los eventos que coinciden, en orden cronológico descendente.
- **Dado** un evento con más de 30 días, **entonces** su ubicación puntual ya no está disponible.

### CA-09 Privacidad (F-12)

- **Dado** un tutor nuevo, **cuando** intenta registrar un menor sin haber aceptado el aviso de privacidad vigente, **entonces** el sistema lo impide.
- **Dado** que el tutor principal elimina su cuenta, **entonces** los dispositivos se desvinculan de inmediato y los datos quedan eliminados en ≤ 30 días, verificable en una auditoría de base de datos y respaldos.

### CA-10 Criterios generales de liberación del MVP

- Todos los requerimientos P0 implementados y con sus criterios de aceptación aprobados.
- Prueba de campo con al menos 20 familias durante 2 semanas, cumpliendo RNF-FIA-02, RNF-FIA-03 y RNF-PER-03.
- Revisión de seguridad OWASP MASVS L1 sin hallazgos críticos ni altos abiertos.
- Aviso de privacidad revisado por el área legal.
- Inventario de licencias completo y sin componentes que incumplan RNF-LIC-01 o RNF-LIC-03.
- Aprobación en Google Play.

### CA-11 Verificación periódica (RF-044)

- **Dado** una sesión con el menor en el destino y la geocerca de salida deshabilitada en pruebas, **cuando** el menor sale del destino, **entonces** la alerta de salida se emite en ≤ 23 minutos (un ciclo más la confirmación de 3 minutos).
- **Dado** una sesión sin llegada registrada y la geocerca de llegada deshabilitada en pruebas, **cuando** el menor está dentro del destino, **entonces** la llegada se registra en el siguiente ciclo de verificación.

### CA-12 Ejecución local (RNF-OPE-01 a RNF-OPE-03)

- **Dado** un equipo con Docker Engine o Podman, JDK y Android Studio, **cuando** se sigue la guía de ejecución local, **entonces** el sistema completo arranca con un solo comando y la app del emulador se conecta al backend local.
- **Dado** el ambiente local, **cuando** se ejecuta el escenario de llegada con ubicación simulada, **entonces** el tutor de prueba recibe el aviso de llegada.

---

## 10. Métricas de éxito

### 10.1 Métrica norte

**Familias con al menos un aviso de llegada entregado por semana.** Refleja que el producto está configurado, funciona y entrega su valor central.

### 10.2 Adopción y retención

| Métrica | Definición | Meta a 90 días del lanzamiento |
|---|---|---|
| Activación | % de tutores registrados que vinculan un menor y crean un destino en las primeras 48 h. | ≥ 60 % |
| Tiempo a la primera llegada | Mediana entre el registro y la primera notificación de llegada. | ≤ 24 h |
| Retención D30 | % de familias activadas que reciben avisos en el día 30. | ≥ 50 % |
| Tutores por familia | Promedio de tutores en grupos activos. | ≥ 1,5 |
| Calificación en tienda | Promedio en Google Play. | ≥ 4,3 |

### 10.3 Calidad del servicio

| Métrica | Meta |
|---|---|
| Latencia de notificación p95 | ≤ 60 s |
| Tasa de detección de llegadas | ≥ 98 % |
| Tasa de falsas alertas de salida (reportadas por tutores o detectadas en telemetría) | ≤ 2 % |
| Consumo de batería adicional en el dispositivo del menor | ≤ 5 % diario |
| Sesiones sin fallos | ≥ 99,5 % |
| Disponibilidad del backend | ≥ 99,5 % |

### 10.4 Señales de alerta

- Más del 20 % de los dispositivos vinculados en estado "Incompleta" por permisos.
- Tasa de desinstalación de la app del menor mayor al 10 % mensual.
- Más del 30 % de los tutores desactivando las alertas de salida (posible fatiga por falsas alertas).

---

## 11. Riesgos del proyecto

Probabilidad e impacto: **A** = alto, **M** = medio, **B** = bajo.

| ID | Riesgo | Prob. | Impacto | Mitigación |
|---|---|---|---|---|
| R-01 | Las restricciones de ejecución en segundo plano de Android (y las capas de ahorro de energía de fabricantes como Xiaomi, Oppo o Samsung) retrasan o impiden la detección de eventos. | A | A | Usar las API nativas de geocercas del sistema operativo; guía de configuración por fabricante; detección de reportes ausentes (RF-041); verificación con ubicación fresca antes de alertas de no llegada (RF-035); matriz de pruebas por fabricante. |
| R-02 | La imprecisión del GPS en interiores o zonas urbanas densas genera falsas alertas de salida. | A | A | Radio mínimo de 100 m, histéresis de 50 m, permanencia de 3 minutos, descarte de lecturas con precisión baja, ajuste de parámetros con datos de la prueba de campo. |
| R-03 | El consumo de batería provoca que el menor o el tutor desactiven la app. | M | A | Priorizar geocercas del sistema sobre GPS continuo; muestreo solo durante ventanas; medir el consumo en la prueba de campo frente a RNF-PER-03. |
| R-04 | Rechazo en Google Play por el uso de ubicación en segundo plano o el tratamiento de datos de menores. | M | A | Revisión temprana de las políticas; justificación clara del permiso; video demostrativo; app sin publicidad ni SDK de terceros innecesarios. |
| R-05 | Incumplimiento normativo en protección de datos de menores. | M | A | Revisión legal del aviso de privacidad y del flujo de consentimiento antes del desarrollo; minimización y retención limitada; registro de consentimientos. |
| R-06 | Brecha de seguridad que exponga ubicaciones de menores. | B | A | Controles de §8.3; pruebas de penetración; autorización por grupo en cada solicitud; cifrado en reposo; monitoreo de accesos anómalos. |
| R-07 | El menor desactiva permisos, apaga la ubicación o desinstala la app. | A | M | Alertas de estado del dispositivo (F-08); aviso de transparencia para fomentar la aceptación; contenidos de acompañamiento para las familias. |
| R-08 | Retrasos o pérdida de notificaciones push por dependencia de FCM. | M | M | Monitoreo de tasa de entrega; reintentos; historial en la app como fuente de verdad; evaluación de canal de respaldo (SMS) para alertas críticas en versiones futuras. |
| R-09 | Uso indebido: un adulto vincula el dispositivo de otra persona sin su conocimiento. | B | A | Aviso permanente y visible en la app del menor; vinculación que requiere acceso físico al dispositivo; mecanismo de reporte de abuso; términos de uso que lo prohíben. |
| R-10 | Fatiga por alertas que reduce la confianza y lleva a desactivar avisos. | M | M | Una sola llegada por ventana; histéresis; preferencias por destino; seguimiento de la señal de alerta de §10.4. |
| R-11 | Picos de carga en horarios escolares que degradan la latencia. | M | M | Procesamiento asíncrono con cola persistente; escalado horizontal de instancias; pruebas de carga con 10× el promedio (RNF-ESC-02). |
| R-12 | Alcance creciente durante el MVP (rutas, SOS, chat). | M | M | Alcance cerrado en §3.2; cambios evaluados contra los objetivos de §2 y diferidos a versiones posteriores. |
| R-13 | Excluir iOS deja fuera a las familias en las que el tutor o el menor usan iPhone. | A | A | Medir la demanda durante el piloto; definir la estrategia de iOS (PA-07) antes de escalar la adquisición de usuarios. |
| R-14 | Los servicios de mapas y búsqueda de direcciones autoalojados (teselas de OpenStreetMap y geocodificador) no escalan o quedan desactualizados. | M | M | Extracto regional (México); actualización mensual de datos; caché en el backend; capacidad medida en pruebas de carga. |
| R-15 | Dispositivos Android sin Google Play Services (p. ej., modelos recientes de Huawei) no pueden usar geocercas ni push. | M | M | Detección en la vinculación con mensaje claro de dispositivo no compatible; requisito publicado en la ficha de la tienda. |

---

## 12. Supuestos y preguntas abiertas

### 12.1 Supuestos

- El tutor y el menor disponen de smartphones Android 10 o superior con Google Play Services, y el menor tiene plan de datos o acceso frecuente a Wi-Fi.
- El público objetivo inicial son familias en México con hijos de 8 a 17 años.
- El MVP se ofrece sin costo; la monetización se definirá después de validar la retención.
- Las notificaciones push son el canal principal; no se contempla SMS en el MVP.

### 12.2 Preguntas abiertas

| ID | Pregunta | Responsable |
|---|---|---|
| PA-01 | ¿Se requiere el consentimiento del propio menor a partir de cierta edad, además del consentimiento del tutor? | Legal |
| PA-02 | ¿Cuál es el modelo de monetización (freemium, suscripción) y qué funcionalidades quedarían en el plan de pago? | Producto |
| PA-03 | ~~¿Tecnología multiplataforma o desarrollo nativo?~~ Resuelta en el SDD v0.2: Android nativo en Java. | Arquitectura |
| PA-04 | ¿Deben las alertas de salida admitir excepciones planificadas (p. ej., salida autorizada temprano)? | Producto |
| PA-05 | ¿Se necesita un canal de respaldo (SMS, llamada) para alertas críticas cuando el tutor no tiene conexión? | Producto |
| PA-06 | Prioridad del roadmap posterior: botón SOS, soporte para relojes inteligentes, versión web, integración con escuelas. | Producto |
| PA-07 | Estrategia para iOS: ¿se asume el costo del Apple Developer Program en una versión posterior o se ofrece una app web para tutores con iPhone? | Producto |

---

## 13. Glosario

| Término | Definición |
|---|---|
| **Destino** | Lugar registrado por el tutor, definido por una ubicación, un radio y un horario. |
| **Geocerca** | Perímetro virtual circular alrededor de un destino. |
| **Ventana de vigilancia** | Intervalo horario en el que se evalúan llegada y salida para un destino. |
| **Hora esperada de llegada** | Hora a la que se espera que el menor esté en el destino; activa la alerta de no llegada. |
| **Tolerancia** | Minutos de margen tras la hora esperada antes de emitir una alerta de no llegada. |
| **Tiempo de permanencia** | Tiempo mínimo dentro o fuera de la geocerca para confirmar un evento. |
| **Histéresis** | Margen adicional al radio que debe superarse para confirmar una salida y así evitar oscilaciones. |
| **Grupo familiar** | Conjunto de tutores y menores que comparten destinos y avisos. |
| **Vinculación** | Proceso que asocia el dispositivo del menor a su perfil en el grupo familiar. |
| **p95 / p99** | Percentil 95 / 99: el valor por debajo del cual se encuentra el 95 % / 99 % de las mediciones. |
| **Derechos ARCO** | Derechos de Acceso, Rectificación, Cancelación y Oposición sobre datos personales. |
