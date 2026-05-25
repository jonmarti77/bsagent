# CALENDAR_RULES.md — Reglas de interpretación del calendario

## Zona horaria

- Toda operación usa `Europe/Madrid` (CET/CEST, UTC+1/UTC+2 según DST).
- El clasificador recibe instrucción explícita de usar esta zona horaria.
- Todos los datetimes almacenados en Data Stores incluyen offset explícito.
- La API de Google Calendar recibe el timezone explícito en cada petición.

## Calendar policy

BSAgent usa una política simple de dos calendarios:

### Personal

- Uso: agenda diaria principal.
- Lectura: sí.
- Escritura futura `CREATE_EVENT`: sí, siempre con confirmación.
- Aparece por defecto en consultas de agenda.
- Es el calendario objetivo de `QUERY_EVENTS` en MVP 1A/MVP 1.

### Gastos

- Uso: cargos y gastos mensuales aproximados.
- Lectura: sí, solo cuando el usuario pregunte explícitamente por gastos, cargos o pagos.
- Escritura: no en MVP 1.
- No aparece por defecto en consultas como "agenda hoy", "agenda mañana" o "agenda semana".
- Puede aparecer en consultas específicas como:
  - "qué gastos tengo esta semana"
  - "qué cargos tengo este mes"
  - "qué pagos entran mañana"

### Reglas de selección de calendario

- `QUERY_EVENTS` usa por defecto el calendario Personal.
- `QUERY_EXPENSE_EVENTS` o equivalente futuro usará el calendario Gastos.
- `CREATE_EVENT` usará el calendario Personal.
- Nunca crear eventos en Gastos en MVP 1.
- Si el usuario pide "agenda", no incluir Gastos.
- Si el usuario pide "gastos", "cargos" o "pagos", consultar Gastos.

## Duración por defecto

- Si el usuario no especifica duración: **30 minutos**.
- Si el usuario especifica duración: usar la indicada.
- Ejemplos de duraciones reconocidas:
  - "una hora" → 60 minutos
  - "dos horas" → 120 minutos
  - "media hora" → 30 minutos
  - "45 minutos" → 45 minutos
  - "todo el día" → evento de día completo (allDay: true)

## Interpretación de referencias temporales

| Referencia | Significado | Regla |
|---|---|---|
| "hoy" | Fecha actual en Europe/Madrid | range: today |
| "mañana" | Fecha actual + 1 día | range: tomorrow |
| "esta semana" | Lunes al domingo de la semana en curso | range: week |
| "el lunes" | Próximo lunes (si hoy es lunes, el siguiente) | Calcular desde fecha actual |
| "el viernes" | Próximo viernes | Calcular desde fecha actual |
| "el [día] que viene" | La semana siguiente | Semana +1 |
| Fecha explícita ("26 de mayo") | Convertir a YYYY-MM-DD | Validar que no es pasado |

## Reglas para CREATE_EVENT

### Campos obligatorios

| Campo | Regla si falta |
|---|---|
| `title` | Solicitar al usuario |
| `date` | Solicitar al usuario → AMBIGUOUS con clarification_question |
| `start_time` | Solicitar al usuario → AMBIGUOUS con clarification_question |

### Campos opcionales con valores por defecto

| Campo | Valor por defecto |
|---|---|
| `duration_minutes` | 30 |
| `description` | "" (vacío) |
| `timezone` | Europe/Madrid |
| `prefix` | "" (sin prefijo si no hay proyecto) |

### Eventos en el pasado

- Si la fecha inferida es anterior al momento actual: no crear.
- Responder: "Esa fecha ya ha pasado. ¿Quieres crear el evento para otra fecha?"
- Excepción: si el usuario lo pide explícitamente ("quiero registrar que ayer...") → permitir, pero pedir confirmación explicando que la fecha es pasada.

### Eventos sin hora específica

- Si el usuario dice "el lunes" sin hora → solicitar hora → AMBIGUOUS.
- No asumir una hora por defecto para CREATE_EVENT.

### Eventos de todo el día

- Si el usuario dice "todo el día" o similar: crear evento allDay.
- No solicitar hora en ese caso.

## Prefijos de proyecto

Cuando el mensaje menciona un proyecto conocido, el clasificador añade el prefijo correspondiente al título del evento.

| Proyecto detectado | `project` | `prefix` | Ejemplo de título |
|---|---|---|---|
| KOBO / kobo | KOBO | [KOBO] | [KOBO] Revisar Resend |
| Gesalaga / gesalaga | Gesalaga | [Gesalaga] | [Gesalaga] Reunión de seguimiento |
| MartIT / martit / Mart IT | MartIT | [MartIT] | [MartIT] Llamada soporte |
| Dibal / dibal | Dibal | [Dibal] | [Dibal] Demo producto |

**Regla:** El prefijo va siempre al principio del título, separado por un espacio.

Si el título ya incluye el prefijo, no duplicarlo.

## Reglas de confirmación

| Intención | ¿Requiere confirmación? |
|---|---|
| QUERY_EVENTS | No |
| CREATE_EVENT | Sí, siempre |
| CONFIRM | No (es la confirmación) |
| CANCEL | No |
| AMBIGUOUS | No (no ejecuta nada) |

El resumen de confirmación debe incluir:
- Título completo (con prefijo si aplica)
- Fecha en formato legible ("mañana, 26 de mayo" o "lunes, 27 de mayo")
- Hora de inicio y hora de fin
- Duración si no es la estándar de 30 min

## Reglas para QUERY_EVENTS

| Rango | Definición |
|---|---|
| `today` | Desde 00:00 hasta 23:59:59 de hoy en Europe/Madrid |
| `tomorrow` | Desde 00:00 hasta 23:59:59 de mañana en Europe/Madrid |
| `week` | Desde el lunes de la semana en curso (00:00) hasta el domingo (23:59:59) |
| `custom` | Rango libre — `from` y `to` obligatorios en ISO 8601 |

Si no hay eventos: responder "No tienes nada [hoy / mañana / esta semana]."

Si hay eventos: listarlos con hora, título y duración (si aplica).

Formato de respuesta recomendado:

```
Tienes 2 eventos mañana:

• 09:00 — Reunión de equipo (1h)
• 15:30 — [KOBO] Revisar Resend (30 min)
```
