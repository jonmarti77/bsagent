# INTENTS.md — Intenciones permitidas en BSAgent MVP 1

## Contrato JSON del clasificador de intención

El clasificador IA devuelve **únicamente JSON válido**. Sin markdown, sin explicaciones, sin texto fuera del JSON.

```json
{
  "intent": "QUERY_EVENTS | CREATE_EVENT | CONFIRM | CANCEL | AMBIGUOUS",
  "confidence": 0.0,
  "params": {
    "range": "today | tomorrow | week | custom | null",
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
  "needs_confirmation": true,
  "ambiguous": false,
  "clarification_question": null
}
```

---

## Intenciones del MVP 1

### QUERY_EVENTS

El usuario quiere consultar su agenda.

**Reglas:**
- `needs_confirmation: false`
- `range` obligatorio: `today`, `tomorrow`, `week` o `custom`
- Si `range = custom`: `from` y `to` deben estar informados (formato ISO 8601)
- No requiere ningún campo de evento (title, date, etc.)

**Ejemplos de activación:**
- "qué tengo hoy"
- "qué hay mañana"
- "enséñame la agenda de esta semana"
- "tengo algo el viernes?"
- "cuáles son mis reuniones de mañana"

---

### CREATE_EVENT

El usuario quiere crear un nuevo evento en Google Calendar.

**Reglas:**
- `needs_confirmation: true` — siempre, sin excepción
- `title` obligatorio
- `date` obligatorio (formato YYYY-MM-DD)
- `start_time` obligatorio (formato HH:MM)
- `duration_minutes` por defecto: 30
- `timezone`: siempre `Europe/Madrid`
- Si falta `date` o `start_time`: devolver `AMBIGUOUS` con `clarification_question`
- Si detecta proyecto conocido: informar `project` y `prefix`
- `missing_fields`: lista los campos que faltan

**Proyectos conocidos y sus prefijos:**
| Nombre en mensaje | project | prefix |
|---|---|---|
| KOBO / kobo | KOBO | [KOBO] |
| Gesalaga / gesalaga | Gesalaga | [Gesalaga] |
| MartIT / martit / Mart IT | MartIT | [MartIT] |
| Dibal / dibal | Dibal | [Dibal] |

**Ejemplos de activación:**
- "mañana a las 10 revisar Resend de KOBO"
- "ponme reunión con Borja el viernes a las 9"
- "crea un evento el lunes a las 15:00 para revisar MartIT"
- "añade a mi calendario el jueves a las 11 llamada con Gesalaga"

---

### CONFIRM

El usuario confirma una acción pendiente existente.

**Reglas:**
- No contiene parámetros de evento
- `params` vacíos — la ejecución recupera los datos de `pending_actions`
- No se combinan datos nuevos con la confirmación
- `needs_confirmation: false`

**Ejemplos de activación:**
- "sí"
- "si"
- "ok"
- "confirmo"
- "adelante"
- "hazlo"
- "venga"
- "perfecto"

---

### CANCEL

El usuario cancela una acción pendiente.

**Reglas:**
- Cancela la `pending_action` activa para este usuario
- `needs_confirmation: false`
- Si no hay acción activa: responder que no hay nada pendiente

**Ejemplos de activación:**
- "cancelar"
- "cancela"
- "no"
- "olvídalo"
- "no lo hagas"
- "para"
- "mejor no"

---

### AMBIGUOUS

El mensaje no puede clasificarse con suficiente certeza.

**Reglas:**
- `ambiguous: true`
- `clarification_question` debe estar informada
- No ejecuta ninguna acción
- No guarda `pending_action`
- Responder al usuario con la `clarification_question`

**Casos típicos:**
- Solicitud de evento sin fecha
- Solicitud de evento sin hora
- Mensaje con múltiples intenciones posibles
- Mensaje demasiado vago para determinar intent

**Ejemplos de activación:**
- "reunión con Borja" (sin fecha ni hora)
- "mañana reviso lo de KOBO" (sin hora)
- "llámame" (sin fecha, hora ni contexto de Calendar)
- "quiero añadir algo" (sin ningún dato)

---

## Intenciones excluidas del MVP 1

| Intent | Estado | Motivo |
|---|---|---|
| UPDATE_EVENT | Excluido | Requiere identificar evento existente; añade ambigüedad y riesgo |
| DELETE_EVENT | Excluido | Riesgo alto de pérdida de datos; requiere confirmación doble |

Estas intenciones se añadirán en fases posteriores cuando el sistema base sea estable.

---

## Intenciones futuras documentadas

### QUERY_EXPENSE_EVENTS

Estado: **future / not implemented in MVP 1A**.

El usuario quiere consultar cargos, gastos o pagos anotados en el calendario secundario
`Gastos`.

**Reglas futuras previstas:**
- No está activa en MVP 1A.
- No debe romper ni ampliar el contrato activo del MVP 1A.
- Solo se implementará después de validar Calendar básico con `QUERY_EVENTS`.
- Usará `calendar_scope: "expenses"`.
- Consultará el calendario Gastos en solo lectura.
- No mezclará resultados con la agenda Personal por defecto.
- No creará eventos en Gastos.

**Ejemplo futuro de activación:**

Input:
```text
Qué gastos tengo esta semana
```

Output futuro:
```json
{
  "intent": "QUERY_EXPENSE_EVENTS",
  "confidence": 0.92,
  "params": {
    "range": "week",
    "calendar_scope": "expenses",
    "timezone": "Europe/Madrid"
  },
  "missing_fields": [],
  "needs_confirmation": false,
  "ambiguous": false,
  "clarification_question": null
}
```

En MVP 1A solo está activo `QUERY_EVENTS` para el calendario Personal.
`QUERY_EXPENSE_EVENTS` se implementará después de validar Calendar básico.
