# N8N_WORKFLOWS.md — Diseño de workflows de BSAgent

## Visión general

BSAgent se implementa con tres workflows en n8n. Cada uno tiene una responsabilidad única y bien delimitada.

| ID | Nombre | Responsabilidad |
|---|---|---|
| WF-01 | bsagent-whatsapp-router | Entrada, deduplicación, contexto, enrutamiento |
| WF-02 | bsagent-calendar-actions | Consulta y creación de eventos en Google Calendar |
| WF-03 | bsagent-logs | Registro de interacciones |

---

## WF-01 — bsagent-whatsapp-router

### Responsabilidad

Recibir todos los mensajes entrantes de WhatsApp y decidir qué hacer con ellos.

### Nodos (secuencia)

```
1. WhatsApp Trigger (webhook)
2. Normalize Message
   - Extraer: message_id, phone_number, text, timestamp
3. Deduplicate Message
   - Leer Data Store: processed_{message_id}
   - Si existe → Stop (respuesta vacía 200)
   - Si no existe → escribir en store y continuar
4. Get Pending Action
   - Leer Data Store: pending_action_{phone_number}
   - Guardar en contexto si existe y está en estado pending/no caducada
5. Classify Intent
   - Llamar a IA con el mensaje + contexto de acción pendiente (si existe)
   - Obtener JSON de respuesta: intent, confidence, params, missing_fields...
6. Route by Intent
   - QUERY_EVENTS → WF-02 (subworkflow o nodo Calendar)
   - CREATE_EVENT + campos completos → WF-02 (crear pending_action)
   - CREATE_EVENT + missing_fields → responder con clarification_question
   - CONFIRM → WF-02 (ejecutar pending_action)
   - CANCEL → cancelar pending_action + responder
   - AMBIGUOUS → responder con clarification_question
7. Send WhatsApp Response
   - Enviar respuesta al phone_number
8. Trigger WF-03 (logs)
```

### Nodos n8n relevantes

- `n8n-nodes-base.webhook` — trigger de entrada
- `n8n-nodes-base.set` — normalización de datos
- `n8n-nodes-base.kv` — Data Store (processed_messages, pending_actions)
- `@n8n/n8n-nodes-langchain.lmChatOpenAi` o `lmChatAnthropic` — clasificador de intención
- `n8n-nodes-base.switch` — enrutamiento por intent
- `n8n-nodes-base.httpRequest` — llamada a WhatsApp Cloud API para respuesta

### Configuración de credenciales (referencias)

- `WHATSAPP_CLOUD_API_CREDENTIAL` — token de acceso de WhatsApp Business Cloud API
- `OPENAI_OR_ANTHROPIC_CREDENTIAL` — clave de API del clasificador

---

## WF-02 — bsagent-calendar-actions

### Responsabilidad

Ejecutar las acciones sobre Google Calendar: consulta y creación de eventos.

### Nodos para QUERY_EVENTS

```
1. Recibir parámetros de WF-01 (range, from, to)
2. Calcular rango de fechas según range (today/tomorrow/week/custom)
3. Google Calendar — List Events
   - calendarId configurado
   - timeMin / timeMax según rango
   - timeZone: Europe/Madrid
4. Format Events Response
   - Construir texto legible con la lista de eventos
   - Si no hay eventos: "No tienes nada [periodo]."
5. Retornar respuesta a WF-01
```

### Nodos para CREATE_EVENT (guardar pending_action)

```
1. Recibir params de WF-01 (title, date, start_time, duration_minutes, etc.)
2. Generar action_id (UUID)
3. Calcular expires_at (created_at + 10 minutos)
4. Guardar en Data Store: pending_action_{phone_number}
5. Construir mensaje de confirmación con resumen del evento
6. Retornar mensaje de confirmación a WF-01
```

### Nodos para CREATE_EVENT (ejecutar tras CONFIRM)

```
1. Recibir phone_number de WF-01
2. Leer pending_action_{phone_number} del Data Store
3. Validar:
   - Existe
   - status == pending
   - expires_at > ahora
4. Cambiar status a executing
5. Google Calendar — Create Event
   - summary: params.prefix + params.title
   - start: params.date + params.start_time
   - end: calculado con duration_minutes
   - timeZone: Europe/Madrid
6. Cambiar status a executed
7. Guardar result_event_id y executed_at
8. Construir mensaje de confirmación de creación
9. Retornar mensaje a WF-01
```

### Nodos n8n relevantes

- `n8n-nodes-base.googleCalendar` — List Events, Create Event
- `n8n-nodes-base.kv` — Data Store (pending_actions)
- `n8n-nodes-base.set` — transformaciones de datos
- `n8n-nodes-base.dateTime` — cálculos de fecha/hora

### Configuración de credenciales (referencias)

- `GOOGLE_CALENDAR_CREDENTIAL` — OAuth2 de Google Calendar
- `N8N_DATA_STORE` — referencia al Data Store configurado

---

## WF-03 — bsagent-logs

### Responsabilidad

Registrar cada interacción en el Data Store de logs para trazabilidad.

### Nodos

```
1. Recibir datos del log desde WF-01 o WF-02
   - phone_number, message_id, user_message
   - intent, confidence
   - params
   - result (QUERIED, PENDING_CONFIRMATION, EVENT_CREATED, ERROR, etc.)
   - calendar_event_id (si aplica)
   - error (si aplica)
2. Generar clave: log_{timestamp}_{uuid}
3. Escribir en Data Store: logs
```

### Activación

- WF-03 se llama al final de cada flujo completado en WF-01.
- No bloquea la respuesta al usuario — puede ejecutarse en paralelo o tras enviar la respuesta.

### Nodos n8n relevantes

- `n8n-nodes-base.kv` — Data Store (logs)
- `n8n-nodes-base.set` — construcción del objeto de log

---

## Notas de implementación

### Flujo de subworkflows vs. nodos encadenados

Para MVP 1, WF-02 y WF-03 pueden implementarse como:
- Subworkflows llamados desde WF-01, o
- Secciones del mismo workflow separadas por nodos Switch

La separación en workflows independientes es recomendada para claridad y mantenibilidad, pero ambas opciones son válidas para el MVP.

### Variables de entorno y configuración

Toda configuración variable (calendar ID, TTL, número de WhatsApp) debe gestionarse como variables de n8n o como valores en el Data Store `project_prefixes`, no como valores hardcodeados en los nodos.

### Orden de construcción recomendado

1. WF-01 solo con webhook + log básico (sin IA)
2. Añadir QUERY_EVENTS con calendario real
3. Añadir clasificador de intención
4. Añadir CREATE_EVENT con pending_action
5. Añadir flujo de CONFIRM
6. Añadir WF-03 de logs
7. Añadir robustez (TTL, estados, errores)
