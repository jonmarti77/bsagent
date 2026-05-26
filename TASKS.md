# TASKS.md — Backlog de BSAgent por fases

## Fase 0 — Documentación base

Estado: **COMPLETADA** — 2026-05-25

- [x] Inicializar repositorio Git
- [x] Crear `.gitignore` con protección de credenciales
- [x] Crear `README.md`
- [x] Crear `AGENTS.md` — reglas operativas para agentes IA
- [x] Crear `ARCHITECTURE.md` — arquitectura y decisiones técnicas
- [x] Crear `OPERATIONS.md` — comportamiento operativo completo
- [x] Crear `TASKS.md` — backlog por fases (este archivo)
- [x] Crear `INTENTS.md` — intenciones permitidas en MVP 1
- [x] Crear `CALENDAR_RULES.md` — reglas de interpretación del calendario
- [x] Crear `N8N_WORKFLOWS.md` — diseño de los tres workflows
- [x] Crear `prompts/system-calendar-agent.md` — system prompt del agente
- [x] Crear `prompts/intent-classifier.md` — prompt de clasificación de intención
- [x] Crear `docs/whatsapp-setup.md` — configuración WhatsApp Business Cloud API
- [x] Crear `docs/google-calendar-setup.md` — configuración Google Calendar OAuth2
- [x] Crear `docs/datastore-model.md` — modelo de Data Stores
- [x] Crear `workflows/README.md` — instrucciones de importación
- [x] Crear `workflows/bsagent-calendar-mvp.template.json` — template sin credenciales

---

## Fase 1A — Calendar Query Test (validación n8n → Google Calendar)

Estado: **VALIDADA CONTRA API REAL** — 2026-05-26

Prerequisitos:

- Credencial `GOOGLE_CALENDAR_CREDENTIAL` configurada en n8n (Google Calendar OAuth2)
- Variable `PRIMARY_CALENDAR_ID` configurada en n8n (Settings → Variables)
- n8n funcionando (no requiere URL pública todavía)

Tareas:

- [x] Revisar credencial Google Calendar OAuth2 en n8n
- [x] Configurar variable `PRIMARY_CALENDAR_ID` en n8n
- [x] Crear workflow WF-00 en n8n directamente vía MCP (ID: r5oLXt5Z716AubDk)
- [x] Asignar credencial real al nodo "Consultar Google Calendar" (credencial: `jonmarti77 calendar`)
- [x] Confirmar que no hay pin data en el workflow (pinData: {} en todas las ejecuciones)
- [x] Confirmar que Always Output Data está activo en el nodo Google Calendar
- [x] Probar `range = today` contra API real — 0 eventos, respuesta legible (ejecución #7, 341ms)
- [x] Probar `range = tomorrow` contra API real — 0 eventos, respuesta legible (ejecución #8, 176ms)
- [x] Probar `range = week` contra API real — 1 evento real devuelto: "Lorea - Cumpleaños" todo el día 31/05 (ejecución #9, 177ms)
- [x] Verificar que la zona horaria de los eventos devueltos es Europe/Madrid (timestamps +02:00 CEST)
- [x] Verificar que no se ha creado ni modificado ningún evento (workflow solo lectura)
- [x] Documentar resultado de cada prueba (ver N8N_WORKFLOWS.md#wf-00)
- [x] No conectar WhatsApp todavía
- [x] No añadir IA todavía

Validación real completada el 2026-05-26:

- Calendario consultado: Personal (`jonmarti77@gmail.com`)
- Credencial usada: jonmarti77 calendar (googleCalendarOAuth2Api)
- Pin data: no usado en ninguna prueba
- today: sin eventos — respuesta "No tienes nada en la agenda de hoy." ✅
- tomorrow: sin eventos — respuesta "No tienes nada en la agenda de mañana." ✅
- week: 1 evento real — "Tienes 1 evento esta semana: • Todo el dia — Lorea - Cumpleaños" ✅
- No se creó, modificó ni eliminó ningún evento en Google Calendar ✅

---

## Fase 1 — Consulta de agenda (integración completa)

Estado: **PENDIENTE**

Prerequisitos:

- Fase 1A completada y validada
- WhatsApp Business Cloud API configurada
- n8n accesible con webhook HTTPS público

Tareas:

- [ ] Configurar webhook de WhatsApp en n8n (WF-01)
- [ ] Implementar recepción y normalización del mensaje entrante
- [ ] Implementar deduplicación por `message_id` usando Data Store
- [ ] Implementar llamada al clasificador de intención (IA)
- [ ] Implementar clasificación `QUERY_EVENTS`
- [ ] Reutilizar bloque de consulta validado en Fase 1A
- [ ] Implementar envío de respuesta por WhatsApp
- [ ] Implementar log básico de interacción (WF-03)
- [ ] Probar consulta con número de test de WhatsApp
- [ ] Validar zona horaria Europe/Madrid en resultados

---

## Fase 2 — Creación de eventos con confirmación

Estado: **PENDIENTE**

Prerequisito: Fase 1 completada y validada

Tareas:

- [ ] Implementar clasificación `CREATE_EVENT`
- [ ] Implementar validación de campos obligatorios (title, date, start_time)
- [ ] Implementar detección de proyecto y prefijo
- [ ] Implementar guardado de `pending_action` en Data Store
- [ ] Implementar mensaje de confirmación con resumen del evento
- [ ] Implementar clasificación `CONFIRM`
- [ ] Implementar recuperación y validación de `pending_action`
- [ ] Implementar cambio de estado `pending → executing`
- [ ] Implementar creación de evento en Google Calendar
- [ ] Implementar cambio de estado `executing → executed`
- [ ] Implementar guardado de `result_event_id`
- [ ] Implementar respuesta de confirmación al usuario
- [ ] Implementar clasificación `CANCEL`
- [ ] Implementar cancelación de `pending_action`
- [ ] Implementar log de éxito y error de creación
- [ ] Probar flujo completo con número de test
- [ ] Validar que eventos creados tienen zona horaria correcta

---

## Fase 3 — Robustez

Estado: **PENDIENTE**

Prerequisito: Fase 2 completada y validada

Tareas:

- [ ] Implementar TTL automático (expiración a los 10 minutos)
- [ ] Implementar detección y respuesta ante acción pendiente cuando llega nueva solicitud
- [ ] Implementar control de estado `executing` (evitar doble ejecución)
- [ ] Implementar manejo de error de Calendar API (estado `failed`)
- [ ] Implementar respuesta ante `AMBIGUOUS` con clarification_question
- [ ] Implementar respuesta ante `CREATE_EVENT` incompleto (missing_fields)
- [ ] Implementar respuesta ante confirmación caducada
- [ ] Implementar respuesta ante CONFIRM sin pending_action activa
- [ ] Revisar y ampliar cobertura de logs
- [ ] Hacer prueba de carga básica (varios mensajes seguidos)
- [ ] Documentar cualquier edge case descubierto en OPERATIONS.md

---

## Fase futura — Calendar (extensiones)

No tiene fecha asignada. Se priorizan cuando el MVP 1-3 esté estable.

- [ ] UPDATE_EVENT con confirmación
- [ ] DELETE_EVENT con doble confirmación
- [ ] Memoria documental de proyectos (KOBO, Gesalaga, MartIT, Dibal)
- [ ] RAG sobre documentación de proyectos
- [ ] Tareas y seguimiento por proyecto
- [ ] Migración de Data Store a Supabase si el volumen lo justifica
- [ ] Multiusuario
- [ ] Notificaciones proactivas (recordatorios)
- [ ] Lectura de correos
- [ ] Integración con otras herramientas (Notion, Linear, etc.)

---

## Fase futura — Expenses Calendar Query

Estado: **PENDIENTE**

Objetivo: consultar el calendario Gastos en solo lectura cuando el usuario pregunte
explícitamente por gastos, cargos o pagos.

Checklist:

- [ ] Definir intención `QUERY_EXPENSE_EVENTS`
- [ ] Decidir rango soportado: `tomorrow`, `week`, `month`
- [ ] Consultar calendario Gastos en solo lectura
- [ ] Formatear cargos por fecha
- [ ] No mezclar con agenda Personal por defecto
- [ ] No crear eventos en Gastos

---

## Módulos futuros — documentados, no implementados

Ver diseño completo en [FUTURE_MODULES.md](FUTURE_MODULES.md).

### Shopping List Module

Prioridad: **Alta** (después de Calendar MVP estabilizado)

- [ ] Diseñar intenciones: ADD_SHOPPING_ITEMS, QUERY_SHOPPING_LIST, MARK_ITEM_BOUGHT, CLEAR_SHOPPING_LIST
- [ ] Definir modelo de datos `shopping_items`
- [ ] Decidir persistencia: Data Store → Supabase
- [ ] Construir workflow n8n del módulo
- [ ] Integrar con WF-01 (clasificador de intención multimodulo)
- [ ] Probar desde WhatsApp

### Wishlist Module

Prioridad: **Media** (después de Shopping List)

- [ ] Diseñar intenciones: ADD_WISHLIST_ITEM, QUERY_WISHLIST, UPDATE_WISHLIST_ITEM, DISCARD_WISHLIST_ITEM
- [ ] Definir modelo de datos `wishlist_items` con categorías y estados
- [ ] Decidir persistencia: Data Store → Supabase
- [ ] Construir workflow n8n del módulo
- [ ] Integrar con WF-01

### Car Maintenance Module

Prioridad: **Media-alta** (historial del coche se pierde con el tiempo)

- [ ] Definir modelo de datos relacional: `vehicles` + `vehicle_events`
- [ ] Diseñar intenciones: LOG_CAR_EVENT, QUERY_CAR_HISTORY, QUERY_NEXT_MAINTENANCE, QUERY_CAR_COSTS
- [ ] Crear tablas en Supabase (persistencia desde el inicio)
- [ ] Revisar y documentar límites de seguridad del módulo
- [ ] Construir workflow n8n del módulo
- [ ] Integrar con WF-01

### Training & Nutrition Module

Prioridad: **Media** (puede subir cuando Calendar MVP esté estable)

- [ ] Migrar y consolidar datos históricos desde ChatGPT proyecto Salud
- [ ] Definir modelo de datos: sessions, exercises, sets, body_weight_logs, nutrition_logs
- [ ] Crear tablas en Supabase
- [ ] Diseñar intenciones: LOG_TRAINING_SESSION, LOG_BODY_WEIGHT, LOG_NUTRITION, QUERY_TRAINING_PROGRESS, QUERY_WEEKLY_SUMMARY
- [ ] Construir workflow n8n del módulo
- [ ] Integrar con WF-01
