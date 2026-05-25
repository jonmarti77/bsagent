# BSAgent

Asistente operativo personal/profesional de Jon, accesible desde WhatsApp.

## Qué es BSAgent

BSAgent es un agente de automatización que permite a Jon gestionar su agenda de Google Calendar directamente desde WhatsApp, usando lenguaje natural en español.

El sistema recibe mensajes de WhatsApp, los clasifica mediante IA, actúa sobre Google Calendar y responde por el mismo canal.

## Problema que resuelve

Consultar y crear eventos en Google Calendar sin abrir ninguna app, directamente desde WhatsApp, con confirmación explícita antes de cualquier escritura.

## Alcance del MVP 1 — BSAgent Calendar

Lo que hace:

- Recibir mensajes desde WhatsApp (WhatsApp Business Cloud API oficial)
- Consultar agenda: hoy, mañana, esta semana
- Crear eventos con confirmación previa obligatoria
- Detectar proyectos conocidos y prefijar eventos automáticamente
- Deduplicar mensajes entrantes
- Registrar logs básicos de cada interacción

Lo que NO hace en MVP 1:

- Modificar eventos existentes
- Eliminar eventos existentes
- Notificaciones proactivas
- Lectura de correos
- Memoria documental de proyectos
- RAG
- Multiusuario

## Principio crítico

**BSAgent puede escribir en Google Calendar, pero nunca crea, modifica ni borra nada sin confirmación explícita del usuario.**

## Diseño modular

BSAgent está diseñado como sistema modular. El Calendar MVP es la base operativa.
Cada módulo futuro añade capacidades sin romper los anteriores.

| Módulo | Estado |
| --- | --- |
| Calendar (MVP actual) | En construcción — Fase 1A |
| Shopping List | Documentado — no implementado |
| Wishlist | Documentado — no implementado |
| Car Maintenance | Documentado — no implementado |
| Training & Nutrition | Documentado — no implementado |

Ver [FUTURE_MODULES.md](FUTURE_MODULES.md) para el diseño de los módulos futuros.

## Estado actual

> Fase 1A — Validación n8n → Google Calendar. Template WF-00 listo para importar en n8n.

## Estructura del proyecto

```
BSAgent/
├── AGENTS.md              Reglas operativas para agentes IA
├── ARCHITECTURE.md        Arquitectura general y decisiones técnicas
├── FUTURE_MODULES.md      Módulos futuros documentados (no implementados)
├── OPERATIONS.md          Comportamiento operativo, reglas, estados
├── TASKS.md               Backlog por fases
├── INTENTS.md             Intenciones permitidas en MVP 1
├── CALENDAR_RULES.md      Reglas de interpretación del calendario
├── N8N_WORKFLOWS.md       Diseño de workflows n8n
├── README.md              Este archivo
├── .gitignore             Protege credenciales y archivos sensibles
├── workflows/
│   ├── README.md                              Instrucciones de importación
│   ├── bsagent-calendar-mvp.template.json     Template MVP completo (sin credenciales)
│   └── bsagent-calendar-query-test.template.json   Template Fase 1A — solo consulta
├── prompts/
│   ├── system-calendar-agent.md   System prompt del agente
│   └── intent-classifier.md      Prompt de clasificación de intención
└── docs/
    ├── whatsapp-setup.md          Configuración WhatsApp Business Cloud API
    ├── google-calendar-setup.md   Configuración credencial Google Calendar
    └── datastore-model.md         Modelo de Data Stores en n8n
```

## Advertencia de seguridad

**No subas credenciales a este repositorio.**

Las credenciales deben vivir exclusivamente en n8n Credentials.
El `.gitignore` está configurado para bloquear los archivos sensibles más comunes,
pero es responsabilidad del operador no exportar workflows con credenciales reales.

## Tecnologías

| Capa | Tecnología |
| --- | --- |
| Canal de entrada | WhatsApp Business Cloud API (Meta) |
| Automatización | n8n |
| Clasificación de intención | Claude (Anthropic) u OpenAI |
| Calendario | Google Calendar API (OAuth2) |
| Persistencia temporal | n8n Data Store |
| Zona horaria | Europe/Madrid |
