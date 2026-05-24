# OPERATIONS.md — Comportamiento operativo de BSAgent

## Principio fundamental

BSAgent nunca crea, modifica ni elimina nada en Google Calendar sin confirmación explícita del usuario.

Esta regla no tiene excepciones en MVP 1.

## Flujo de un mensaje entrante

### Paso 1 — Recepción y deduplicación

1. Llega webhook de WhatsApp con el mensaje.
2. Extraer `message_id` del payload.
3. Consultar el store `processed_messages` con clave `processed_{message_id}`.
4. Si ya existe: ignorar el mensaje silenciosamente y no procesar.
5. Si no existe: continuar al paso 2.
6. Registrar `message_id` en `processed_messages` inmediatamente.

**Motivo:** WhatsApp puede reenviar el mismo webhook más de una vez.

### Paso 2 — Contexto previo

1. Consultar `pending_actions` buscando una acción activa para este `phone_number`.
2. Si existe acción en estado `pending` y no caducada: guardar en contexto para el paso 3.
3. Si existe acción pero está `expired`, `executed`, `cancelled` o `failed`: ignorar y continuar sin contexto previo.

### Paso 3 — Clasificación de intención

1. Enviar mensaje al clasificador de intención (IA).
2. El clasificador devuelve únicamente JSON válido — ver `INTENTS.md` para el contrato.
3. Evaluar el intent recibido.

### Paso 4 — Enrutamiento por intent

| Intent | Acción |
|---|---|
| QUERY_EVENTS | Consultar Google Calendar → responder |
| CREATE_EVENT (completo) | Guardar pending_action → pedir confirmación |
| CREATE_EVENT (incompleto) | Responder con clarification_question del JSON |
| CONFIRM | Ver flujo de confirmación |
| CANCEL | Ver flujo de cancelación |
| AMBIGUOUS | Responder con clarification_question del JSON |

## Flujo de confirmación (CREATE_EVENT)

1. Usuario pide crear evento con todos los campos necesarios.
2. IA devuelve `CREATE_EVENT` con params completos.
3. Workflow valida que title, date y start_time están presentes.
4. Workflow guarda `pending_action` con:
   - `status: pending`
   - `expires_at: created_at + 10 minutos`
   - `params` tal cual vienen del clasificador
5. Workflow responde por WhatsApp con resumen claro:
   ```
   Voy a crear este evento en tu calendario:
   
   📅 [KOBO] Revisar Resend
   📆 Mañana, 26 de mayo
   🕙 10:00 — 10:30
   
   ¿Confirmas? (responde "sí" o "cancelar")
   ```
6. Usuario responde.
7. Workflow clasifica como CONFIRM o CANCEL.

### Si CONFIRM

1. Recuperar `pending_action` del store para este `phone_number`.
2. Validar:
   - Existe → si no: responder "No hay nada pendiente de confirmar."
   - `status == pending` → si no: responder según estado (ver tabla de estados)
   - `expires_at > ahora` → si caducada: responder "La confirmación ha caducado. Puedes volver a pedir el evento."
3. Cambiar `status` a `executing`.
4. Crear evento en Google Calendar usando **únicamente los params de pending_action**.
5. Cambiar `status` a `executed`.
6. Guardar `result_event_id` y `executed_at`.
7. Responder: "Evento creado. [título] el [fecha] a las [hora]."
8. Registrar log.

### Si CANCEL

1. Recuperar `pending_action` activa para este `phone_number`.
2. Si existe y está en `pending`: cambiar `status` a `cancelled`.
3. Responder: "Cancelado. No se ha creado ningún evento."
4. Si no existe acción activa: responder "No hay nada pendiente de cancelar."

## Estados de pending_action

| Estado | Significado | ¿Puede ejecutarse? |
|---|---|---|
| `pending` | Esperando confirmación | Sí, si no ha caducado |
| `executing` | En proceso de creación | No (en ejecución) |
| `executed` | Evento creado exitosamente | No (ya ejecutado) |
| `expired` | TTL superado sin confirmación | No |
| `cancelled` | Cancelado por el usuario | No |
| `failed` | Error al crear en Calendar | No (requiere nueva solicitud) |

## Reglas ante mensajes ambiguos

- Si el clasificador devuelve `AMBIGUOUS`, responder con la `clarification_question` del JSON.
- No ejecutar ninguna acción.
- No guardar pending_action.
- Ejemplo: "¿A qué hora sería la reunión con Borja?"

## Reglas ante mensajes incompletos (CREATE_EVENT)

- Si `missing_fields` no está vacío en el JSON del clasificador, preguntar por los campos que faltan.
- No guardar pending_action hasta tener todos los campos.
- Ejemplo: "¿Para qué fecha sería el evento?"

## Gestión de acción pendiente cuando llega nueva solicitud

Si existe una `pending_action` en estado `pending` para el usuario y llega una nueva solicitud (no CONFIRM ni CANCEL):

1. Notificar al usuario que hay una acción pendiente sin confirmar.
2. Preguntar si quiere cancelarla o confirmarla antes de continuar.
3. No procesar la nueva solicitud hasta resolver la anterior.
4. Respuesta sugerida:
   ```
   Tienes un evento pendiente de confirmar:
   [KOBO] Revisar Resend — mañana a las 10:00
   
   ¿Confirmas ese evento, lo cancelas, o quieres una cosa diferente?
   ```

## Gestión de errores de Google Calendar

- Si la API de Calendar falla al crear:
  1. Cambiar `status` de pending_action a `failed`.
  2. Guardar el error en el campo `error` de pending_action.
  3. Responder al usuario: "Ha habido un error al crear el evento. Puedes intentarlo de nuevo."
  4. Registrar log con el error.
- No reintentar automáticamente.

## Zona horaria

- Toda interpretación de fechas y horas usa `Europe/Madrid`.
- Toda fecha almacenada en stores incluye offset (`+02:00` o `+01:00` según DST).
- El clasificador recibe instrucción explícita de usar `Europe/Madrid`.

## Deduplicación

- Clave: `processed_{message_id}` en el store `processed_messages`.
- Se registra **antes** de clasificar la intención.
- No tiene TTL en MVP 1 — el store puede limpiarse manualmente si crece demasiado.

## Logging mínimo

Cada interacción genera un registro en el store `logs` con:

- `phone_number`
- `message_id`
- `user_message`
- `intent` detectado
- `confidence`
- `params`
- `result` (PENDING_CONFIRMATION, EVENT_CREATED, QUERIED, ERROR, etc.)
- `calendar_event_id` (si aplica)
- `error` (si aplica)
- `created_at`

## Reglas de respuesta

- Responder siempre en español.
- Mensajes cortos y claros.
- Sin jerga técnica en las respuestas al usuario.
- Confirmar siempre la acción realizada o el siguiente paso esperado.
- Si no se entiende el mensaje, pedir aclaración — no asumir.
