# Modelo de Data Stores — BSAgent MVP 1

## Por qué Data Store y no base de datos externa

En MVP 1, BSAgent usa el **n8n Data Store** (nodo KV) como capa de persistencia.

**Motivos:**
- Volumen mínimo (un solo usuario, pocas interacciones diarias)
- Sin necesidad de consultas complejas
- Reduce dependencias externas y simplifica el setup inicial
- Las operaciones requeridas son clave-valor simples: leer, escribir, actualizar

**Cuándo migrar a Supabase:**
- Si el número de usuarios crece más allá de 3-5
- Si se necesitan consultas por rango de fechas, filtros complejos o reporting
- Si el volumen de logs hace el Data Store inmanejable
- Si se añaden funcionalidades que requieren relaciones entre entidades

---

## Store 1 — processed_messages

**Objetivo:** Evitar procesar dos veces el mismo webhook de WhatsApp.

**Clave:** `processed_{message_id}`

**Valor:**
```json
{
  "message_id": "wamid.HBgLMzQ2MDA...",
  "phone_number": "+34600000000",
  "processed_at": "2026-05-25T10:00:00+02:00"
}
```

**Reglas:**
- Registrar antes de clasificar la intención.
- Si ya existe la clave: ignorar el mensaje sin procesarlo.
- Sin TTL en MVP 1 (limpiar manualmente si el store crece).
- La clave incluye el `message_id` completo (no truncado).

---

## Store 2 — pending_actions

**Objetivo:** Guardar acciones pendientes de confirmación.

**Clave:** `pending_action_{phone_number}`

**Valor completo:**
```json
{
  "action_id": "550e8400-e29b-41d4-a716-446655440000",
  "phone_number": "+34600000000",
  "intent": "CREATE_EVENT",
  "params": {
    "title": "[KOBO] Revisar Resend",
    "project": "KOBO",
    "prefix": "[KOBO]",
    "date": "2026-05-26",
    "start_time": "10:00",
    "duration_minutes": 30,
    "timezone": "Europe/Madrid",
    "description": ""
  },
  "status": "pending",
  "created_at": "2026-05-25T10:00:00+02:00",
  "expires_at": "2026-05-25T10:10:00+02:00",
  "executed_at": null,
  "result_event_id": null,
  "error": null
}
```

### Estados permitidos

| Estado | Descripción | ¿Puede ejecutarse? |
|---|---|---|
| `pending` | Esperando confirmación del usuario | Sí, si no ha caducado |
| `executing` | En proceso de creación en Calendar | No (ya en ejecución) |
| `executed` | Evento creado exitosamente | No (ya ejecutado) |
| `expired` | TTL superado sin confirmación | No |
| `cancelled` | Cancelado por el usuario | No |
| `failed` | Error al crear en Calendar | No (requiere nueva solicitud) |

### Reglas de idempotencia

- Solo una pending_action activa por usuario en MVP 1.
- CONFIRM solo puede ejecutar una acción en estado `pending` y no caducada.
- La ejecución usa **únicamente** los params guardados — nunca datos nuevos del mensaje de confirmación.
- Una acción `executed` no puede ejecutarse dos veces.
- Una acción `cancelled` no puede ejecutarse.
- Una acción `expired` no puede ejecutarse.
- Antes de guardar una nueva pending_action: verificar que no hay una `pending` activa sin resolver.

### TTL

- **Valor recomendado:** 10 minutos
- `expires_at = created_at + 10 minutos`
- La expiración se evalúa en el momento del CONFIRM, no automáticamente
- En MVP 1 no hay job de limpieza automático — las acciones caducadas quedan en el store con estado `expired`

---

## Store 3 — logs

**Objetivo:** Trazabilidad básica de cada interacción.

**Clave:** `log_{timestamp_unix}_{uuid_corto}`

Ejemplo: `log_1748167200_a3f9`

**Valor:**
```json
{
  "phone_number": "+34600000000",
  "message_id": "wamid.HBgLMzQ2MDA...",
  "user_message": "mañana a las 10 revisar Resend de KOBO",
  "intent": "CREATE_EVENT",
  "confidence": 0.95,
  "params": {
    "title": "[KOBO] Revisar Resend",
    "date": "2026-05-26",
    "start_time": "10:00",
    "duration_minutes": 30,
    "timezone": "Europe/Madrid"
  },
  "result": "PENDING_CONFIRMATION",
  "calendar_event_id": null,
  "error": null,
  "created_at": "2026-05-25T10:00:00+02:00"
}
```

### Valores de `result`

| Valor | Significado |
|---|---|
| `QUERIED` | Consulta de agenda ejecutada |
| `PENDING_CONFIRMATION` | CREATE_EVENT guardado, esperando confirmación |
| `EVENT_CREATED` | Evento creado exitosamente en Calendar |
| `CANCELLED` | Acción cancelada por el usuario |
| `EXPIRED` | Acción caducada sin confirmar |
| `AMBIGUOUS` | Mensaje ambiguo, se pidió aclaración |
| `DEDUPLICATED` | Mensaje ignorado por duplicado |
| `ERROR_CALENDAR` | Error al crear el evento en Calendar |
| `ERROR_CLASSIFIER` | Error en el clasificador de intención |

---

## Store 4 — project_prefixes

**Objetivo:** Configuración centralizada de prefijos de proyecto.

**Clave:** `project_prefixes`

**Valor:**
```json
{
  "kobo": "[KOBO]",
  "gesalaga": "[Gesalaga]",
  "martit": "[MartIT]",
  "dibal": "[Dibal]"
}
```

**Reglas:**
- Las claves están en minúsculas para comparación insensible a mayúsculas.
- El clasificador recibe los prefijos como parte del system prompt (valor estático).
- Este store sirve como referencia para n8n si se necesita validar o enriquecer la respuesta del clasificador.
- Para añadir un nuevo proyecto: actualizar este store y el system prompt.

---

## Resumen de stores

| Store | Clave | Propósito | TTL |
|---|---|---|---|
| `processed_messages` | `processed_{message_id}` | Deduplicación | Sin TTL en MVP 1 |
| `pending_actions` | `pending_action_{phone_number}` | Acciones pendientes | 10 minutos |
| `logs` | `log_{timestamp}_{uuid}` | Trazabilidad | Sin TTL en MVP 1 |
| `project_prefixes` | `project_prefixes` | Configuración | Permanente |

---

## Consideraciones para migración a Supabase

Cuando se migre a Supabase, el modelo de tablas sería:

```sql
-- processed_messages → tabla simple con índice en message_id
CREATE TABLE processed_messages (
  message_id TEXT PRIMARY KEY,
  phone_number TEXT NOT NULL,
  processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- pending_actions → tabla con estado y TTL
CREATE TABLE pending_actions (
  action_id UUID PRIMARY KEY,
  phone_number TEXT NOT NULL,
  intent TEXT NOT NULL,
  params JSONB NOT NULL,
  status TEXT NOT NULL DEFAULT 'pending',
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at TIMESTAMPTZ NOT NULL,
  executed_at TIMESTAMPTZ,
  result_event_id TEXT,
  error TEXT
);

-- logs → append-only, sin borrado
CREATE TABLE interaction_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_number TEXT NOT NULL,
  message_id TEXT NOT NULL,
  user_message TEXT NOT NULL,
  intent TEXT NOT NULL,
  confidence FLOAT,
  params JSONB,
  result TEXT NOT NULL,
  calendar_event_id TEXT,
  error TEXT,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- project_prefixes → tabla de configuración
CREATE TABLE project_prefixes (
  project_key TEXT PRIMARY KEY,
  prefix TEXT NOT NULL
);
```
