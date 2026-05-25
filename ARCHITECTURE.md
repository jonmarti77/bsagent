# ARCHITECTURE.md — Arquitectura de BSAgent

## Visión general

BSAgent es un sistema de automatización que conecta WhatsApp con Google Calendar a través de n8n, usando IA para clasificar la intención del usuario.

El diseño prioriza:
- **Simplicidad** — mínima complejidad estructural para MVP
- **Trazabilidad** — cada acción queda registrada
- **Seguridad** — ninguna escritura sin confirmación explícita
- **Mantenibilidad** — flujos separados por responsabilidad

## Flujo principal

```
Usuario (WhatsApp)
       ↓
  WhatsApp Business Cloud API (Meta webhook)
       ↓
  n8n — WF-01 bsagent-whatsapp-router
       ├── Deduplicación (processed_messages)
       ├── Recuperar pending_action activa si existe
       ├── Clasificador de intención (IA)
       └── Enrutamiento por intent
              ↓
  n8n — WF-02 bsagent-calendar-actions
       ├── QUERY_EVENTS → Google Calendar API → respuesta inmediata
       └── CREATE_EVENT → guardar pending_action → pedir confirmación
              ↓ (tras CONFIRM del usuario)
       └── Ejecutar creación en Google Calendar
              ↓
  n8n — WF-03 bsagent-logs
       └── Registrar interacción completa
              ↓
  WhatsApp Business Cloud API → respuesta al usuario
```

## Componentes

### WhatsApp Business Cloud API (Meta)

- Canal de entrada y salida
- Webhook HTTPS hacia n8n
- Verificación de token
- Número de producción de Jon

### n8n

- Motor de automatización
- Aloja los tres workflows
- Gestiona credenciales de forma segura
- Proporciona el Data Store para persistencia temporal

### Clasificador de intención (IA)

- Recibe el mensaje de texto del usuario
- Devuelve JSON estructurado con intent, params y metadata
- Modelo: Claude (Anthropic) u OpenAI GPT
- Prompt definido en `prompts/intent-classifier.md`
- No inventa datos — si faltan campos, devuelve AMBIGUOUS

### Google Calendar API

- OAuth2 via n8n Credentials
- Operaciones permitidas en MVP 1:
  - Listar eventos (read)
  - Crear evento (write, solo tras confirmación)
- Zona horaria: Europe/Madrid
- Calendario principal de agenda configurado en n8n: Personal
- Calendario secundario de gastos, solo lectura futura: Gastos

### Política multi-calendario simple

BSAgent soportará una política multi-calendario mínima, sin diseñar todavía una
capa multi-calendario compleja ni una capa financiera.

| Calendario | Rol | Uso |
|---|---|---|
| Personal | Primary agenda calendar | Agenda diaria, consultas por defecto y futuras creaciones con confirmación |
| Gastos | Secondary read-only expenses calendar | Consultas explícitas de gastos, cargos o pagos |

`QUERY_EVENTS` consulta Personal por defecto. Una intención futura como
`QUERY_EXPENSE_EVENTS` consultará Gastos cuando el usuario pregunte explícitamente
por gastos/cargos/pagos. `CREATE_EVENT` escribirá solo en Personal durante MVP 1.

### n8n Data Store

- Persistencia temporal para MVP
- Cuatro stores: processed_messages, pending_actions, logs, project_prefixes
- No requiere base de datos externa en esta fase
- Ver `docs/datastore-model.md` para el modelo completo

## Decisiones técnicas

### Por qué n8n

- Agilidad en MVP — sin infraestructura custom
- Credenciales gestionadas de forma segura dentro de n8n
- Visual y auditable
- Suficiente para el volumen esperado (uso personal/profesional de Jon)

### Por qué Data Store y no Supabase en MVP 1

- El volumen de datos es mínimo (un solo usuario)
- No hay necesidad de consultas complejas en MVP
- Reduce dependencias externas
- La migración a Supabase es explícita en TASKS.md para cuando el Data Store sea insuficiente

### Por qué tres workflows separados

- Responsabilidad única por workflow
- Facilita depuración y monitorización por separado
- WF-03 puede activarse o desactivarse sin afectar a los demás
- Permite evolucionar cada parte de forma independiente

### Por qué confirmación obligatoria para CREATE_EVENT

- BSAgent tiene acceso de escritura a un calendario real de producción
- Un error de IA no debería crear eventos directamente
- El coste de una confirmación extra es bajo; el coste de un evento creado erróneamente es alto
- Esta regla no tiene excepciones en MVP 1

### Por qué solo CREATE_EVENT y no UPDATE/DELETE

- Menor riesgo de pérdida de datos en MVP
- Simplifica la lógica de estados
- UPDATE y DELETE requieren identificar el evento correcto, lo que añade ambigüedad
- Se añaden en fases posteriores cuando el sistema sea estable

## Límites del MVP 1

| Capacidad | Estado |
|---|---|
| Consultar agenda | Incluido |
| Crear eventos con confirmación | Incluido |
| Modificar eventos | Excluido |
| Eliminar eventos | Excluido |
| Notificaciones proactivas | Excluido |
| Lectura de correos | Excluido |
| Multiusuario | Excluido |
| RAG / memoria documental | Excluido |
| Tareas por proyecto | Excluido |

## Fases futuras

| Fase | Contenido |
|---|---|
| MVP 2 | UPDATE_EVENT con confirmación |
| MVP 3 | DELETE_EVENT con doble confirmación |
| MVP 4 | Memoria documental de proyectos |
| MVP 5 | RAG sobre documentación de proyectos |
| MVP 6 | Migración a Supabase si Data Store no escala |
| MVP 7 | Multiusuario |

## Consideraciones de seguridad

- Las credenciales nunca viven en archivos del repositorio
- El `.gitignore` bloquea los patrones de archivos sensibles más comunes
- pending_actions usa TTL para evitar confirmaciones huérfanas
- La deduplicación por message_id evita doble ejecución por reentregas del webhook
- Una acción executed no puede ejecutarse dos veces
