# Prompt — Clasificador de intención

> Este archivo documenta el prompt de usuario que acompaña al system prompt en el clasificador.
> Se usa como referencia para construir el nodo LLM en n8n y para validar el comportamiento del clasificador.

---

## Prompt de usuario (template)

```
Fecha y hora actual: {{fecha_hora_actual}} (Europe/Madrid)

Mensaje del usuario: "{{mensaje_usuario}}"

{{#if accion_pendiente}}
Contexto: Hay una acción pendiente de confirmación:
- Tipo: {{accion_pendiente.intent}}
- Título: {{accion_pendiente.params.title}}
- Fecha: {{accion_pendiente.params.date}}
- Hora: {{accion_pendiente.params.start_time}}
- Estado: {{accion_pendiente.status}}
{{/if}}

Clasifica la intención del mensaje y devuelve ÚNICAMENTE el JSON.
```

---

## Ejemplos de clasificación

### Consulta de agenda — hoy

**Mensaje:** "qué tengo hoy"

```json
{
  "intent": "QUERY_EVENTS",
  "confidence": 0.98,
  "params": {
    "range": "today",
    "from": null,
    "to": null,
    "title": null,
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Consulta de agenda — mañana

**Mensaje:** "qué tengo mañana"

```json
{
  "intent": "QUERY_EVENTS",
  "confidence": 0.98,
  "params": {
    "range": "tomorrow",
    "from": null,
    "to": null,
    "title": null,
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Consulta de agenda — esta semana

**Mensaje:** "qué tengo esta semana"

```json
{
  "intent": "QUERY_EVENTS",
  "confidence": 0.97,
  "params": {
    "range": "week",
    "from": null,
    "to": null,
    "title": null,
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Creación de evento con proyecto — completo

**Mensaje:** "mañana a las 10 revisar Resend de KOBO"
*(Fecha actual: 2026-05-25)*

```json
{
  "intent": "CREATE_EVENT",
  "confidence": 0.95,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": "[KOBO] Revisar Resend",
    "project": "KOBO",
    "prefix": "[KOBO]",
    "date": "2026-05-26",
    "start_time": "10:00",
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": true,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Creación de evento sin proyecto — completo

**Mensaje:** "ponme reunión con Borja el viernes a las 9"
*(Fecha actual: 2026-05-25, próximo viernes = 2026-05-29)*

```json
{
  "intent": "CREATE_EVENT",
  "confidence": 0.93,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": "Reunión con Borja",
    "project": null,
    "prefix": null,
    "date": "2026-05-29",
    "start_time": "09:00",
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": true,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Confirmación

**Mensaje:** "sí"
*(Con acción pendiente en contexto)*

```json
{
  "intent": "CONFIRM",
  "confidence": 0.99,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": null,
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Cancelación

**Mensaje:** "cancelar"

```json
{
  "intent": "CANCEL",
  "confidence": 0.99,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": null,
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

---

### Ambiguo — sin fecha

**Mensaje:** "reunión con Borja"

```json
{
  "intent": "AMBIGUOUS",
  "confidence": 0.60,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": "Reunión con Borja",
    "project": null,
    "prefix": null,
    "date": null,
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": ["date", "start_time"],
  "needs_confirmation": false,
  "ambiguous": true,
  "clarification_question": "¿Para qué fecha y hora sería la reunión con Borja?"
}
```

---

### Ambiguo — sin hora

**Mensaje:** "mañana reviso lo de KOBO"

```json
{
  "intent": "AMBIGUOUS",
  "confidence": 0.65,
  "params": {
    "range": null,
    "from": null,
    "to": null,
    "title": "[KOBO] Revisar",
    "project": "KOBO",
    "prefix": "[KOBO]",
    "date": "2026-05-26",
    "start_time": null,
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": null
  },
  "missing_fields": ["start_time"],
  "needs_confirmation": false,
  "ambiguous": true,
  "clarification_question": "¿A qué hora quieres revisar lo de KOBO mañana?"
}
```

---

## Notas para el implementador

- Si el mensaje del usuario es solo "ok", "vale", "perfecto" u equivalentes **y existe una acción pendiente en el contexto**, clasificar como CONFIRM.
- Si el mensaje es "ok" o similar pero **no hay acción pendiente**, clasificar como AMBIGUOUS con clarification_question.
- La fecha de los ejemplos de CREATE_EVENT debe calcularse dinámicamente a partir de la fecha actual proporcionada en el prompt, no usarse como valor fijo.
- El campo `confidence` refleja la certeza del clasificador. Por debajo de 0.75 considerar añadir clarification_question aunque el intent parezca claro.
