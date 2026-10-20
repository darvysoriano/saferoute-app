# saferoute-app
 Aplicación móvil que permite a padres o tutores registrar un destino para sus hijos y recibir notificaciones cuando llegan al lugar, así como alertas si se mueven de esa ubicación posteriormente.
# 📱 SafeRoute — Aplicación móvil de monitoreo de ubicación en tiempo real

## 📌 Ficha del Proyecto
**Nombre del proyecto:** SafeRoute  
**Descripción:** Aplicación móvil que permite a padres o tutores registrar un destino para sus hijos y recibir notificaciones cuando llegan al lugar, así como alertas si se mueven posteriormente.  
**Repositorio:** https://github.com/darvysoriano/saferoute-app  

---

## 🎯 Descripción General del Producto

### Objetivo
Brindar a los padres una herramienta confiable para monitorear la ubicación de sus hijos mediante validaciones automáticas basadas en geolocalización.

### Características principales
- Registro de destino al que el hijo debe llegar.
- Validación automática cuando el hijo llega al destino.
- Verificación cada 20 minutos para detectar si permanece en el mismo lugar.
- Notificación inmediata si el hijo se mueve del sitio registrado.
- Visualización de ubicación en tiempo real.
- Historial básico de movimientos.

### Funcionalidades
1. **Registro de destino:** El padre define una ubicación objetivo (latitud/longitud).
2. **Monitoreo en tiempo real:** La app del hijo envía su ubicación periódicamente.
3. **Alerta de llegada:** Cuando el hijo entra al radio del destino, se envía una notificación.
4. **Verificación periódica:** Cada 20 minutos se compara la ubicación actual con la última registrada.
5. **Alerta de movimiento:** Si la ubicación cambia, se notifica al padre y se muestra el nuevo punto en el mapa.

---

## 🏗️ Arquitectura del Sistema

### Arquitectura General
**Frontend móvil (Flutter o React Native)**  
- App Padre  
- App Hijo  

**Backend (Node.js + Express)**  
- API REST para registro de destinos, usuarios y notificaciones.  
- WebSockets para actualizaciones en tiempo real.  

**Base de datos (Firebase Firestore o MongoDB Atlas)**  
- Colección de usuarios  
- Colección de destinos  
- Colección de ubicaciones  
- Colección de alertas  

### Flujo de Datos
1. El padre registra un destino → API lo guarda.
2. El hijo envía su ubicación cada X minutos → API la procesa.
3. El backend compara ubicación actual vs destino.
4. Si coincide → envía notificación de llegada.
5. Cada 20 minutos → backend compara ubicación actual vs última ubicación.
6. Si cambia → envía alerta de movimiento.

---

## 🗂️ Modelo de Datos (Ejemplo en JSON)

### Usuario
```json
{
  "id": "string",
  "nombre": "string",
  "rol": "padre | hijo",
  "telefono": "string"
}

## Destino

{
  "id": "string",
  "usuarioId": "string",
  "lat": "number",
  "lng": "number",
  "radio": 50
}

## Ubicación

{
  "id": "string",
  "usuarioId": "string",
  "lat": "number",
  "lng": "number",
  "timestamp": "date"
}


##Alerta

{
  "id": "string",
  "usuarioId": "string",
  "tipo": "llegada | movimiento",
  "mensaje": "string",
  "timestamp": "date"
}
