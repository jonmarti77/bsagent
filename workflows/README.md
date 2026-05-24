# workflows/ — Instrucciones de importación

Esta carpeta contiene templates de workflows de n8n listos para importar.

## Archivos

| Archivo | Descripción |
|---|---|
| `bsagent-calendar-mvp.template.json` | Template del workflow principal de BSAgent MVP 1 |

## Advertencia

**Los archivos de esta carpeta son templates sin credenciales.**

No contienen tokens reales, IDs de calendario, URLs privadas ni secretos de ningún tipo.
Antes de importar, leer las instrucciones de configuración en:

- [docs/whatsapp-setup.md](../docs/whatsapp-setup.md)
- [docs/google-calendar-setup.md](../docs/google-calendar-setup.md)

## Cómo importar en n8n

1. Abrir n8n → Workflows
2. Hacer clic en "Import from file" o "Import from JSON"
3. Seleccionar el archivo `.template.json`
4. Revisar todos los nodos — las referencias a credenciales aparecen como placeholders
5. Asignar las credenciales reales a cada nodo que las requiera
6. Revisar y actualizar las variables de configuración (calendar ID, etc.)
7. Activar el workflow

## Qué no exportar a este repositorio

- Exports de n8n con credenciales reales (contienen tokens y secretos)
- Exports con IDs de calendarios reales
- Exports con números de teléfono reales
- Cualquier archivo generado por n8n que contenga el campo `"credentials"` con valores reales

El `.gitignore` del proyecto está configurado para bloquear los patrones más comunes,
pero es responsabilidad del operador no subir archivos sensibles.
