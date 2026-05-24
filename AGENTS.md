# AGENTS.md — Reglas operativas para agentes IA en BSAgent

Este archivo prescribe el comportamiento obligatorio de cualquier agente IA (Claude Code, Codex u otros) que trabaje en este proyecto.

## Orden de lectura obligatorio

Antes de tocar cualquier archivo, leer en este orden:

1. `AGENTS.md` (este archivo)
2. `ARCHITECTURE.md`
3. `OPERATIONS.md`
4. `N8N_WORKFLOWS.md`
5. `INTENTS.md`
6. `CALENDAR_RULES.md`

## Reglas obligatorias

### Antes de modificar

- Leer siempre los archivos afectados antes de editar.
- No sobrescribir documentación existente sin revisar su contenido.
- Identificar si la tarea afecta alguna zona sensible antes de actuar.
- Para cambios en lógica de confirmación, pending_actions o Data Stores: mostrar propuesta antes de aplicar.

### Documentar antes de construir

- No crear workflows en n8n sin que la documentación correspondiente esté aprobada.
- Cualquier nuevo workflow debe estar descrito en `N8N_WORKFLOWS.md` antes de implementarse.
- Cualquier cambio en el modelo de stores debe estar reflejado en `docs/datastore-model.md`.

### Credenciales y seguridad

- No tocar credenciales de ningún tipo.
- No añadir tokens, secrets ni API keys en ningún archivo del repositorio.
- Las credenciales deben vivir exclusivamente en n8n Credentials.
- No exportar workflows con credenciales reales.
- No usar WhatsApp Web no oficial ni scraping.

### Restricciones de MVP 1

- **No crear** UPDATE_EVENT ni DELETE_EVENT en ningún archivo ni workflow.
- **No crear** workflows directamente en n8n sin autorización explícita de Jon.
- **No añadir** RAG ni memoria documental completa.
- **No añadir** lógica multiusuario.
- **No añadir** notificaciones proactivas.

### Commits y cambios

- Commits pequeños y atómicos — un commit = un cambio lógico.
- Sin commits ni push autónomos — solo cuando Jon lo pida explícitamente.
- Sin refactors oportunistas — no limpiar código que no es parte de la tarea.
- Idioma: español en todos los comentarios, docs, variables descriptivas y mensajes de UI.

### Después de cada cambio

Actualizar los archivos `.md` afectados según la tabla del CLAUDE.md global.
No dejar documentación desincronizada del código o los workflows.

## Zonas sensibles

| Zona | Motivo | Restricción |
|---|---|---|
| pending_actions | Controla escritura en Calendar | Cambios solo con propuesta previa |
| Lógica de confirmación | Impide acciones no autorizadas | No modificar sin revisión explícita |
| Credenciales Google/WhatsApp | Acceso a sistemas externos reales | Solo en n8n Credentials |
| Clasificador de intención | Punto de entrada de toda la lógica | Cambios con ejemplos de prueba |
| TTL y estados de pending_action | Idempotencia y seguridad | No reducir TTL sin justificación |

## Qué puede hacer el agente sin autorización explícita

- Crear y modificar archivos `.md`
- Actualizar `.gitignore`
- Actualizar templates en `workflows/` (sin credenciales reales)
- Actualizar prompts en `prompts/`
- Actualizar documentación en `docs/`

## Qué requiere autorización explícita de Jon

- Crear o modificar workflows directamente en n8n
- Conectar o modificar credenciales
- Hacer push a GitHub
- Añadir cualquier intención fuera del alcance del MVP 1
- Modificar la lógica de confirmación o los estados de pending_action
