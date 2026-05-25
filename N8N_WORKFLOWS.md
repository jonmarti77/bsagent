# N8N_WORKFLOWS.md — Diseño de workflows de BSAgent

## Visión general

BSAgent se implementa con tres workflows de producción y un workflow de validación previo.

| ID | Nombre | Responsabilidad | Estado |
| --- | --- | --- | --- |
| WF-00 | bsagent-calendar-query-test | Validación n8n → Google Calendar (solo lectura) | Completado — Fase 1A |
| WF-01 | bsagent-whatsapp-router | Entrada, deduplicación, contexto, enrutamiento | Fase 1 |
| WF-02 | bsagent-calendar-actions | Consulta y creación de eventos en Google Calendar | Fase 1 |
| WF-03 | bsagent-logs | Registro de interacciones | Fase 1 |

---

## WF-00 — bsagent-calendar-query-test

### Objetivo

Workflow temporal de validación. Verifica que n8n puede consultar Google Calendar correctamente antes de conectar WhatsApp o IA.

**No escribe en Calendar. Solo lectura.**

Una vez validado, el bloque de consulta (nodos 3–5) se reutiliza directamente en WF-02.

### Estado en n8n

| Campo | Valor |
| --- | --- |
| Workflow ID | `r5oLXt5Z716AubDk` |
| Nombre en n8n | BSAgent - Fase 1A - Calendar Query Test |
| Creado | 2026-05-25 |
| Credencial asignada | `kobo.ogiak` (googleCalendarOAuth2Api) |
| Estado | Activo — pruebas con datos simulados completadas |

### Resultados de las pruebas (datos simulados — 2026-05-25)

| Prueba | Salida | Estado |
| --- | --- | --- |
| `range = today` | `Tienes 2 eventos hoy:` `• 09:00–10:00 Reunion de equipo` `• 15:30–16:00 [KOBO] Revisar Resend` | ✅ |
| `range = tomorrow` | `Tienes 1 evento mañana:` `• 10:00–11:00 Llamada con Gesalaga` | ✅ |
| `range = week` | `Tienes 3 eventos esta semana:` `• 09:00–10:00 Revision semanal MartIT` `• 16:00–17:00 [Dibal] Demo producto` `• Todo el dia — ITV coche` | ✅ |

Zona horaria: todos los timestamps devueltos en `+02:00` (CEST, Europe/Madrid). Prefijos de proyecto preservados (`[KOBO]`, `[Dibal]`). Evento todo el día formateado correctamente.

**Pendiente:** ejecutar sin pin data contra Google Calendar real para validar la conexión API.

### Tipo

Temporal / prueba. Puede archivarse o eliminarse tras validar Fase 1A.

### Diseño de nodos

```
1. Manual Trigger
   Permite lanzar el workflow manualmente desde n8n.

2. Set — Entrada de prueba
   Define el rango a consultar.
   Campo: range = "today" | "tomorrow" | "week"
   Cambiar este valor para probar cada caso.

3. Code — Calcular rango de fechas
   Calcula startDateTime y endDateTime en Europe/Madrid.
   Devuelve también humanRangeLabel para el mensaje de respuesta.

4. Google Calendar — Consultar eventos
   Lista los eventos del calendario en el rango calculado.
   Credencial: GOOGLE_CALENDAR_CREDENTIAL
   Calendario: PRIMARY_CALENDAR_ID (variable n8n)
   Máximo: 20 eventos

5. Code — Formatear respuesta
   Convierte los eventos en texto legible.
   Ejemplo con eventos:
     "Tienes 2 eventos hoy:\n\n• 09:00–10:00 Reunión X\n• 15:30–16:00 [KOBO] Revisar Resend"
   Ejemplo sin eventos:
     "No tienes nada en la agenda de hoy."

6. Set — Output final
   Expone el campo "text" con la respuesta formateada.
   Punto de integración: en WF-01 este campo será el cuerpo del mensaje de WhatsApp.
```

### Decisión técnica: definición de "week"

`week` = lunes 00:00:00 hasta domingo 23:59:59 de la semana en curso (Europe/Madrid).

**Motivo:** Cuando se pregunta "qué tengo esta semana", la respuesta útil es la vista completa
de la semana, incluyendo eventos pasados del lunes o martes. Ayuda a Jon a ver el contexto
semanal completo, no solo los eventos futuros. Definición fija según `CALENDAR_RULES.md`.

Si la semana ya ha empezado (ej: hoy es miércoles), el rango incluirá también lunes y martes.

### Código del nodo 3 — Calcular rango de fechas

```javascript
// DateTime está disponible como global en n8n Code nodes — no requiere import
const tz = 'Europe/Madrid';
const now = DateTime.now().setZone(tz);
const range = $input.first().json.range;

let startDateTime, endDateTime, humanRangeLabel;

if (range === 'today') {
  startDateTime = now.startOf('day');
  endDateTime = now.endOf('day');
  humanRangeLabel = 'hoy';
} else if (range === 'tomorrow') {
  const mañana = now.plus({ days: 1 });
  startDateTime = mañana.startOf('day');
  endDateTime = mañana.endOf('day');
  humanRangeLabel = 'mañana';
} else if (range === 'week') {
  // Lunes–domingo de la semana en curso (ISO week, Luxon usa lunes como inicio)
  startDateTime = now.startOf('week');
  endDateTime = now.endOf('week');
  humanRangeLabel = 'esta semana';
} else {
  throw new Error(`Rango no válido: ${range}. Usar: today | tomorrow | week`);
}

return [{
  json: {
    range,
    startDateTime: startDateTime.toISO(),
    endDateTime: endDateTime.toISO(),
    timezone: tz,
    humanRangeLabel
  }
}];
```

### Código del nodo 5 — Formatear respuesta

```javascript
// DateTime está disponible como global en n8n Code nodes — no requiere import
const tz = 'Europe/Madrid';

// Leer todos los ítems devueltos por Google Calendar
const items = $input.all();

// Leer humanRangeLabel del nodo anterior
const humanRangeLabel = $('Calcular rango de fechas').first().json.humanRangeLabel;

// Google Calendar devuelve array vacío si no hay eventos
// En n8n, si el nodo no devuelve ítems, items puede estar vacío o tener un solo ítem sin "id"
const eventos = items.filter(item => item.json && item.json.id);

if (eventos.length === 0) {
  return [{ json: { text: `No tienes nada en la agenda de ${humanRangeLabel}.` } }];
}

const lineas = eventos.map(item => {
  const ev = item.json;
  const titulo = ev.summary || '(sin título)';

  // Evento de todo el día: start.date presente, start.dateTime ausente
  if (ev.start?.date && !ev.start?.dateTime) {
    return `• Todo el día — ${titulo}`;
  }

  const inicio = DateTime.fromISO(ev.start.dateTime).setZone(tz).toFormat('HH:mm');
  const fin    = DateTime.fromISO(ev.end.dateTime).setZone(tz).toFormat('HH:mm');
  return `• ${inicio}–${fin} ${titulo}`;
});

const n = eventos.length;
const cabecera = `Tienes ${n} evento${n !== 1 ? 's' : ''} ${humanRangeLabel}:`;
const texto = `${cabecera}\n\n${lineas.join('\n')}`;

return [{ json: { text: texto } }];
```

### Configuración requerida en n8n

| Elemento | Tipo | Valor |
|---|---|---|
| `GOOGLE_CALENDAR_CREDENTIAL` | Credencial OAuth2 | Configurar en n8n Credentials |
| `PRIMARY_CALENDAR_ID` | Variable n8n | ID del calendario objetivo |

Para configurar la variable: n8n → Settings → Variables → New Variable.

### Entrada

```json
{ "range": "today" }
```

Cambiar `"today"` por `"tomorrow"` o `"week"` para los otros rangos.

### Salida esperada

```json
{
  "text": "Tienes 2 eventos hoy:\n\n• 09:00–10:00 Reunión de equipo\n• 15:30–16:00 [KOBO] Revisar Resend"
}
```

### Límites

- Solo lectura. No crea, modifica ni elimina eventos.
- No tiene WhatsApp. La salida queda en el panel de ejecución de n8n.
- No tiene IA ni clasificador de intención.
- Máximo 20 eventos por consulta.
- No persiste nada en Data Stores.

### Reutilización en WF-02

Los nodos 3, 4 y 5 de este workflow se copian directamente en WF-02 como bloque QUERY_EVENTS.
La entrada (`range`, `startDateTime`, `endDateTime`) vendrá del clasificador de intención en lugar del nodo Set manual.

---

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
