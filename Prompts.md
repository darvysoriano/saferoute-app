

# 🤖 Registro de Prompts — SafeRoute

Este archivo contiene el historial de prompts utilizados durante el desarrollo del proyecto SafeRoute, siguiendo los lineamientos del curso AI4Devs.

## Prompt 1

Contexto: Proyecto SafeRoute en mi repositorio local. Necesito crear la estructura de carpetas para la documentación técnica del proyecto.
Prompt: Crea dentro del proyecto las carpetas /docs, /docs/prd, /docs/sdd y /docs/ai. No generes contenido aún, solo la estructura de directorios.

## Prompt 2

Contexto: Aplicación móvil para validar la ubicación de mi hijo mediante geolocalización, notificaciones de llegada y alertas de movimiento. La documentación debe seguir los estándares de un PRD profesional.
Prompt: Genera el archivo /docs/prd/PRD-SafeRoute.md con un PRD completo que incluya: visión del producto, objetivos, alcance, funcionalidades detalladas, casos de uso, requerimientos funcionales, requerimientos no funcionales, criterios de aceptación, métricas de éxito y riesgos del proyecto. Usa lenguaje técnico claro y estructura profesional.

## Prompt 3

Contexto: Arquitectura móvil + backend + base de datos. La app debe validar llegada a destino y detectar movimiento cada 20 minutos. Necesito un documento técnico para desarrolladores.
Prompt: Genera el archivo /docs/sdd/SDD-SafeRoute.md con un SDD completo que incluya: arquitectura general del sistema, diagramas de componentes, diseño del backend, diseño del frontend móvil, modelo de datos, APIs y endpoints, flujo de validación de llegada, flujo de verificación cada 20 minutos, manejo de notificaciones, decisiones técnicas, dependencias y estándares de desarrollo. Usa formato técnico profesional.

## prompt 4

Contexto: Validación de la arquitectura propuesta y validación para poder poner en ejecución el proyecto con herramientas sin licenciamiento.
Prompt: La arquitectura general debe ser con Java y servicios gratuitos sin licencia para poder llevar acabo su ejecución tanto en frontend movil, backend, base de datos, notificaciones, mapas, realiza estas modificaciones en la documentación y el analisis para realizar la ejecución sin problemas de licenciamiento al momento de ser ejecutados y poder llevar a pruebas en un ambiente local.  continua usando un formato técnico profesional

## Prompt 5

Contexto: Necesito llevar un registro de todos los prompts usados para generar documentación técnica.
Prompt: Crea el archivo /docs/ai/prompts-history.md y registra los prompts utilizados hasta ahora, incluyendo fecha, descripción y el prompt exacto. Este archivo debe actualizarse cada vez que se ejecute un nuevo prompt.

## Prompt 6

Contexto: Necesito un archivo donde Cloud Code registre automáticamente los resultados de cada generación de documentación.
Prompt: Crea el archivo /docs/ai/generation-logs.md y agrega una sección para registrar logs de cada ejecución, incluyendo: fecha, archivo generado, tiempo de ejecución y observaciones.


## Prompt 7

Contexto: Valide la documentación necesaria para este proyecto ocupando la IA para aprovechar el conocimiento y resolviendo algunas dudas de lo que debemos siendo Arquitectos de SW, para lo cual se preparo el siguiente Prompt.

Prompt:
Actúa como un Arquitecto de Software Principal y Technical Lead especializado en sistemas distribuidos, IoT y aplicaciones móviles. Tu tarea es generar la documentación técnica exhaustiva para el proyecto "SafeRoute", siguiendo las mejores prácticas de la industria (IEEE 829, IEEE 29148, arquitectura limpia y Clean Code).

### RESTRICCIONES CRÍTICAS Y LÍMITES DEL PROYECTO:
1. Plataforma Móvil: EXCLUSIVAMENTE Android (arquitectura compatible con Flutter o React Native). Queda terminantemente FUERA DE ALCANCE el desarrollo o soporte para iOS.
2. Presupuesto e Infraestructura: Se debe diseñar y documentar utilizando ÚNICAMENTE herramientas, librerías, servicios y niveles gratuitos (Free Tier / Open Source bajo licencias permisivas tipo MIT/Apache 2.0). No utilizar servicios que requieran tarjeta de crédito con riesgo de facturación imprevista (por ejemplo, priorizar OpenStreetMap / MapLibre / Leaflet frente a Google Maps SDK comercial, Firebase Free Spark Plan, MongoDB Atlas Free M0, y hosting gratuito como Render, Fly.io o similar).
3. Monitoreo en segundo plano: Abordar a fondo las restricciones de Android para servicios en segundo plano (Doze Mode, Foreground Service con notificación permanente y permisos ACCESS_BACKGROUND_LOCATION).

### ESTRUCTURA DE LA DOCUMENTACIÓN REQUERIDA:
Genera un documento técnico formal, estructurado en Markdown profesional, que contenga las siguientes secciones completas y detalladas:

1. ESPECIFICACIÓN DE REQUISITOS DE SOFTWARE (SRS)
   - Tabla de Requisitos Funcionales (RF-01 a RF-XX) con ID, Nombre, Descripción, Prioridad (MoSCoW) y Criterio de Aceptación.
   - Requisitos No Funcionales (RNF): Rendimiento de batería, seguridad de datos (menores de edad), latencia de WebSockets y tolerancia a pérdida de red móvil.
   - Definición formal del algoritmo de cálculo de geocercas (especificar la fórmula matemática de Haversine y su implementación en código backend para calcular distancias entre coordenadas).

2. ARQUITECTURA DEL SISTEMA Y DISEÑO DE INFRAESTRUCTURA GRATUITA
   - Diagrama de componentes textual (estilo C4 Model - Nivel Contenedores) que ilustre la relación entre: App Padre (Android), App Hijo (Android), Backend API (Node.js/Express), Servidor WebSocket, Base de Datos (MongoDB Atlas Free M0) y Servicio de Notificaciones (Firebase Cloud Messaging - Plan Gratuito).
   - Manejo de persistencia y tareas en background en Android (explicación técnica de la implementación de Foreground Service y WorkManager para el tracking cada 20 min sin que el sistema operativo elimine el proceso).

3. ESPECIFICACIÓN DE INTERFACES Y CONTRATOS (APIs & WEBSOCKETS)
   - Contratos RESTful en formato OpenAPI/JSON con ejemplos completos de Request y Response para:
     * POST /api/v1/auth/register y login (padre e hijo).
     * POST /api/v1/destinations (creación de geovalla con radio).
     * GET /api/v1/locations/history/:childId (historial de rutas).
   - Protocolo de WebSockets:
     * Especificación de eventos entrantes y salientes (ej. `location:ping`, `geofence:status`, `alert:movement_detected`).
     * Estructura exacta de los JSON payloads.

4. MODELO DE DATOS DETALLADO (MONGODB / MONGOOSE SCHEMAS)
   - Código completo de esquemas en Mongoose para:
     * User (roles: TUTOR, CHILD).
     * Destination (con índice geoespacial `2dsphere`).
     * LocationLog (historial con TTL index para purgar datos automáticamente y no sobrepasar el límite de 512 MB del tier gratuito de MongoDB Atlas).
     * Alert (registro de eventos de llegada y de movimiento indebido).

5. CASOS DE PRUEBA CRÍTICOS Y PLAN DE ASEGURAMIENTO (QA)
   - Matriz de Casos de Prueba (ID, Precondición, Pasos, Resultado Esperado) cubriendo:
     * Precisión GPS fluctuante o baja precisión (evitar falsos positivos de movimiento).
     * Pérdida y recuperación de señal celular en el trayecto.
     * Activación y desactivación del ciclo de 20 minutos.


    Valida los doucmentos ya documentados que son el PRD-SafeRoute.md y SDD-SafeRoute.md que todo la documentación este relacionada con lo ya documentado y validado.