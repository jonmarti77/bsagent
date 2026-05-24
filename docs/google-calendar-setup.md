# Configuración — Google Calendar

## Requisitos previos

- Cuenta de Google con el calendario que Jon quiere gestionar
- Acceso a n8n con capacidad para configurar credenciales OAuth2
- n8n accesible desde internet para el callback OAuth2

## Crear la credencial en n8n

BSAgent usa una credencial de tipo **Google Calendar OAuth2** en n8n.
La credencial se llama: `GOOGLE_CALENDAR_CREDENTIAL`

### Opción A — OAuth2 app propia (recomendada para producción)

1. Ir a [console.cloud.google.com](https://console.cloud.google.com)
2. Crear un proyecto o usar uno existente
3. Activar la API: **Google Calendar API**
4. Ir a "Credenciales" → "Crear credenciales" → "ID de cliente OAuth 2.0"
5. Tipo de aplicación: **Aplicación web**
6. URI de redirección autorizado: `https://{tu-n8n}/rest/oauth2-credential/callback`
7. Descargar el JSON con `client_id` y `client_secret`
8. En n8n → Credentials → New → Google Calendar OAuth2 API
9. Introducir `client_id` y `client_secret`
10. Hacer clic en "Connect my account" y autorizar con la cuenta de Jon

### Opción B — OAuth2 de n8n (más rápida para desarrollo)

n8n proporciona OAuth2 propio para Google Calendar en algunas instalaciones.
Seguir el flujo de autorización integrado en n8n si está disponible.

## Permisos mínimos necesarios

| Scope | Motivo |
|---|---|
| `https://www.googleapis.com/auth/calendar.readonly` | Leer eventos (QUERY_EVENTS) |
| `https://www.googleapis.com/auth/calendar.events` | Crear eventos (CREATE_EVENT) |

No solicitar permisos adicionales (acceso a otros calendarios, gestión de usuarios, etc.).

## Identificar el calendario objetivo

1. Abrir [calendar.google.com](https://calendar.google.com)
2. En el panel izquierdo, hacer clic en los tres puntos del calendario objetivo
3. "Configuración y uso compartido"
4. Copiar el **ID del calendario** (tiene formato: `nombreusuario@gmail.com` para el principal, o algo como `abc123@group.calendar.google.com` para secundarios)
5. Guardar este ID en n8n como variable de entorno o en la configuración del nodo

## Zona horaria

- Configurar el calendario en Google Calendar con zona horaria `Europe/Madrid`
- Todas las peticiones a la API incluyen `timeZone: "Europe/Madrid"` explícitamente
- n8n también debe tener configurada la zona horaria `Europe/Madrid` (Settings → General → Timezone)

## Verificar el acceso

Antes de construir el flujo completo de creación, validar el acceso con una consulta:

1. En n8n, crear un workflow de prueba
2. Añadir nodo Google Calendar → List Events
3. Configurar:
   - Calendar: el ID del calendario objetivo
   - Time Min: hoy a las 00:00
   - Time Max: hoy a las 23:59
   - Time Zone: Europe/Madrid
4. Ejecutar y verificar que devuelve eventos reales del calendario de Jon

## Prueba de creación de evento

Solo realizar cuando la consulta funcione correctamente:

1. Añadir nodo Google Calendar → Create Event en un workflow de prueba
2. Configurar:
   - Calendar: el ID del calendario objetivo
   - Title: "BSAgent prueba"
   - Start: fecha/hora de prueba
   - End: fecha/hora + 30 min
   - Time Zone: Europe/Madrid
3. Ejecutar
4. Verificar en Google Calendar que el evento aparece
5. Borrar el evento de prueba manualmente

## Campos usados en CREATE_EVENT

| Campo n8n | Valor |
|---|---|
| `summary` | `params.prefix + " " + params.title` (o solo `params.title` si no hay prefijo) |
| `start.dateTime` | `params.date + "T" + params.start_time + ":00"` |
| `start.timeZone` | `"Europe/Madrid"` |
| `end.dateTime` | `start.dateTime + params.duration_minutes` |
| `end.timeZone` | `"Europe/Madrid"` |
| `description` | `params.description` (puede estar vacío) |

## Errores comunes

| Error | Causa probable | Solución |
|---|---|---|
| 401 Unauthorized | Token OAuth2 caducado | Reconectar la credencial en n8n |
| 403 Forbidden | Scope insuficiente | Revisar permisos de la app en Google Cloud Console |
| 404 Calendar not found | ID de calendario incorrecto | Verificar el ID del calendario objetivo |
| Invalid time zone | Zona horaria no reconocida | Usar exactamente `"Europe/Madrid"` |
| Event end time before start | Cálculo de duración incorrecto | Revisar el cálculo de `end.dateTime` |
