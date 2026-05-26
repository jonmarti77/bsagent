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
| --- | --- |
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

## Calendarios usados por BSAgent

### Personal

- Calendario operativo principal.
- Se usa para las pruebas actuales de WF-00.
- Se usa por defecto para consultas de agenda (`QUERY_EVENTS`).
- Puede tener pocos eventos; una respuesta vacía no implica error.
- Será el destino de `CREATE_EVENT` cuando esa capacidad se active, siempre con confirmación.

### Gastos

- Calendario secundario.
- Útil para consultas futuras de cargos mensuales aproximados.
- No entra en WF-00 por defecto.
- No debe mezclarse con la agenda Personal salvo que el usuario pregunte explícitamente por gastos, cargos o pagos.
- No se debe escribir en este calendario en MVP 1.

### Nota de validación

Las pruebas con pin data no validan la conexión real. La validación real requiere:

- Credencial asignada y autorizada en n8n.
- Pin data desactivado — confirmar que `pinData: {}` en la ejecución.
- Ejecución contra API (el nodo Calendar mostrará un tiempo de ejecución > 100 ms; pin data es instantáneo).

Personal puede devolver 0 eventos sin que sea error. Si today y tomorrow están vacíos, usar `range = week` para forzar un rango más amplio. Si todos los rangos devuelven vacío, la validación sigue siendo válida si la API responde sin errores.

Para validación manual fuerte: crear un evento temporal en Personal, ejecutar el workflow con `range = today`, verificar que aparece en la respuesta, y borrar el evento después.

**Validación completada el 2026-05-26** contra Google Calendar API real:

- today: 0 eventos — API respondió correctamente en 341 ms ✅
- tomorrow: 0 eventos — API respondió correctamente en 176 ms ✅
- week: 1 evento real devuelto ("Lorea - Cumpleaños", todo el día, 31/05) en 177 ms ✅
- No se creó, modificó ni eliminó ningún evento ✅

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
| --- | --- |
| `summary` | `params.prefix + " " + params.title` (o solo `params.title` si no hay prefijo) |
| `start.dateTime` | `params.date + "T" + params.start_time + ":00"` |
| `start.timeZone` | `"Europe/Madrid"` |
| `end.dateTime` | `start.dateTime + params.duration_minutes` |
| `end.timeZone` | `"Europe/Madrid"` |
| `description` | `params.description` (puede estar vacío) |

## Prueba Fase 1A — Calendar Query Test

Esta sección cubre la validación del acceso a Google Calendar usando el workflow WF-00
(`BSAgent - Fase 1A - Calendar Query Test`).

### Pasos para ejecutar la prueba

1. Importar `workflows/bsagent-calendar-query-test.template.json` en n8n.
2. Abrir el nodo "Consultar Google Calendar" y asignar la credencial `GOOGLE_CALENDAR_CREDENTIAL`.
3. En n8n → Settings → Variables, crear `PRIMARY_CALENDAR_ID` con el ID del calendario objetivo.
4. En el nodo "Entrada de prueba", verificar que el campo `range` tiene el valor a probar.
5. Hacer clic en "Test workflow" o "Execute workflow".
6. Revisar la salida del nodo "Output final" — debe contener el campo `text`.

### Prueba 1 — range = today

- Cambiar `range` a `"today"` en el nodo "Entrada de prueba".
- Ejecutar el workflow.
- Resultado esperado con eventos: `"Tienes N eventos hoy:\n\n• HH:MM–HH:MM Título..."`
- Resultado esperado sin eventos: `"No tienes nada en la agenda de hoy."`

### Prueba 2 — range = tomorrow

- Cambiar `range` a `"tomorrow"`.
- Resultado esperado con eventos: `"Tienes N eventos mañana:\n\n• ..."`
- Resultado esperado sin eventos: `"No tienes nada en la agenda de mañana."`

### Prueba 3 — range = week

- Cambiar `range` a `"week"`.
- Resultado esperado: lista de todos los eventos del lunes al domingo de la semana actual.
- Verificar que el rango empieza en el lunes de la semana en curso (aunque hoy sea miércoles).

### Qué revisar en la salida

| Verificación | Cómo comprobarlo |
| --- | --- |
| Los eventos aparecen en hora local Spain | Las horas deben reflejar Europe/Madrid, no UTC |
| Los eventos de todo el día muestran "Todo el día" | Eventos sin `start.dateTime` se formatean distinto |
| Los títulos con prefijo aparecen completos | `[KOBO] Revisar Resend`, no truncados |
| No se ha creado ningún evento nuevo | Abrir Google Calendar y confirmar visualmente |
| No se ha modificado ningún evento | Los eventos existentes no han cambiado |

### Validación de zona horaria

Para verificar que Europe/Madrid es correcta:

1. Crear un evento de prueba manual en Google Calendar para hoy a las 10:00 (hora local Spain).
2. Ejecutar el workflow con `range = today`.
3. Verificar que el evento aparece como `10:00`, no como `08:00` (UTC) ni `09:00` (UTC+1 incorrecto).
4. En verano (CEST, UTC+2): un evento a las 10:00 local debe aparecer como `10:00`.
5. En invierno (CET, UTC+1): un evento a las 10:00 local debe aparecer como `10:00`.

### Errores esperables en esta fase

| Error | Causa probable | Solución |
| --- | --- | --- |
| "No se pudo autenticar" o 401 | Credencial no conectada o token caducado | Reconectar en n8n Credentials |
| "Calendar not found" o 404 | `PRIMARY_CALENDAR_ID` incorrecto o vacío | Verificar el ID del calendario en Google Calendar settings |
| Eventos en UTC (horas incorrectas) | `timeZone` no configurado en el nodo Calendar | Revisar que el nodo tiene `timeZone: Europe/Madrid` en opciones |
| Lista vacía aunque haya eventos | Rango de fechas calculado incorrectamente | Revisar el Code node "Calcular rango de fechas" con `console.log` |
| "Rango no válido" | Valor de `range` no es `today`, `tomorrow` o `week` | Revisar el nodo "Entrada de prueba" |

### Confirmación de solo lectura

Después de ejecutar las pruebas:

- Abrir [calendar.google.com](https://calendar.google.com).
- Confirmar que no hay eventos nuevos creados por BSAgent.
- Confirmar que los eventos existentes no han sido modificados.
- El workflow WF-00 no tiene ningún nodo de creación, modificación ni borrado.

---

## Errores comunes

| Error | Causa probable | Solución |
| --- | --- | --- |
| 401 Unauthorized | Token OAuth2 caducado | Reconectar la credencial en n8n |
| 403 Forbidden | Scope insuficiente | Revisar permisos de la app en Google Cloud Console |
| 404 Calendar not found | ID de calendario incorrecto | Verificar el ID del calendario objetivo |
| Invalid time zone | Zona horaria no reconocida | Usar exactamente `"Europe/Madrid"` |
| Event end time before start | Cálculo de duración incorrecto | Revisar el cálculo de `end.dateTime` |
