# Configuración — WhatsApp Business Cloud API

## Requisito previo

BSAgent usa **WhatsApp Business Cloud API oficial de Meta**.

No se usa WhatsApp Web, baileys, whatsapp-web.js ni ninguna solución no oficial.
El uso de APIs no oficiales viola los términos de servicio de WhatsApp y puede resultar en el bloqueo del número.

## Componentes necesarios

| Componente | Proveedor | Dónde se configura |
|---|---|---|
| Número de WhatsApp Business | Meta / operador | Meta Business Manager |
| WhatsApp Business Account (WABA) | Meta | Meta Business Manager |
| App de Meta (tipo Business) | Meta Developers | developers.facebook.com |
| Token de acceso permanente | Meta | Meta Developers → App |
| Webhook HTTPS | n8n | n8n (webhook node) |
| Verify token | Definido por Jon | n8n (webhook node) |

## Paso a paso

### 1. Crear la app en Meta for Developers

1. Ir a [developers.facebook.com](https://developers.facebook.com)
2. Crear una nueva app → Tipo: Business
3. Añadir el producto "WhatsApp"
4. Asociar una WhatsApp Business Account (WABA)

### 2. Configurar el número

- En el panel de WhatsApp → "Números de teléfono"
- Para pruebas: usar el número de test que Meta proporciona (hasta 5 contactos)
- Para producción: añadir y verificar el número real de Jon

### 3. Obtener el token de acceso

- Generar un token de acceso permanente (no el temporal de 24h)
- El token se guarda **únicamente** en n8n Credentials, nunca en archivos
- Credencial en n8n: `WHATSAPP_CLOUD_API_CREDENTIAL`

### 4. Configurar el webhook en n8n

1. Crear un nodo Webhook en n8n (WF-01)
2. Configurar método: POST
3. Anotar la URL pública del webhook (requiere n8n con HTTPS accesible desde internet)
4. Definir un `verify_token` personalizado (cadena aleatoria segura)
5. El `verify_token` también se guarda en n8n Credentials o como variable de entorno

### 5. Registrar el webhook en Meta

1. Ir al panel de la app → WhatsApp → Configuration
2. En "Webhook":
   - Callback URL: URL pública de n8n
   - Verify token: el valor definido en el paso anterior
3. Suscribirse al campo: `messages`
4. Meta hará una petición GET al webhook para verificar el token
5. n8n debe responder con el `hub.challenge` que envía Meta

### 6. Formato del webhook entrante

Meta envía un POST con este formato (simplificado):

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "changes": [{
      "value": {
        "messages": [{
          "id": "wamid.xxx",
          "from": "34600000000",
          "timestamp": "1234567890",
          "type": "text",
          "text": {
            "body": "qué tengo mañana"
          }
        }]
      }
    }]
  }]
}
```

El `message_id` es `entry[0].changes[0].value.messages[0].id`.

### 7. Enviar respuestas

BSAgent usa la API de WhatsApp para enviar mensajes de texto simples:

```
POST https://graph.facebook.com/v19.0/{phone_number_id}/messages
Authorization: Bearer {token}
Content-Type: application/json

{
  "messaging_product": "whatsapp",
  "to": "34600000000",
  "type": "text",
  "text": {
    "body": "Tienes 2 eventos mañana..."
  }
}
```

El `phone_number_id` es el ID del número de WhatsApp en Meta (no el número de teléfono).

## Diferencia entre número de test y número real

| Aspecto | Número de test | Número real |
|---|---|---|
| Destinatarios | Hasta 5 (deben ser verificados) | Ilimitados |
| Disponibilidad | Solo durante desarrollo | Siempre |
| Coste | Gratuito | Según tarifa de conversación |
| Verificación | No requiere verificación del número | Requiere verificación |
| Recomendación | Usar para Fase 1 y Fase 2 | Activar en Fase 3 tras validar |

## Prueba Fase 1B — WhatsApp Connectivity Test

Esta sección cubre la validación del canal WhatsApp usando el workflow WF-1B
(`BSAgent - Fase 1B - WhatsApp Connectivity Test`, ID: `yHKEeuNRZoOo8L5A`).

### Estado del workflow (2026-05-26)

| Componente | Estado |
| --- | --- |
| Workflow creado en n8n | ✅ |
| Credencial trigger (`whatsAppTriggerApi`) | ✅ Auto-asignada: `WhatsApp OAuth account - Chatbot GPT` |
| Credencial envío (`whatsAppApi`) | ❌ Pendiente — crear en n8n Credentials |
| Variable `WHATSAPP_PHONE_NUMBER_ID` | ❌ Pendiente — crear en n8n Settings → Variables |
| Workflow activo | ❌ Pendiente — activar en n8n |
| Webhook registrado en Meta | ❌ Pendiente |

### Pasos para completar la configuración

#### 1. Crear credencial de envío en n8n

1. Ir a n8n → Credentials → New → WhatsApp Business Cloud
2. Introducir el token de acceso permanente de Meta
3. Nombre sugerido: `WHATSAPP_API_CREDENTIAL`
4. Asignar a los nodos de envío en WF-1B

#### 2. Configurar variable WHATSAPP_PHONE_NUMBER_ID

1. Ir a n8n → Settings → Variables → New Variable
2. Nombre: `WHATSAPP_PHONE_NUMBER_ID`
3. Valor: el phone_number_id de Meta (visible en Meta Developers → WhatsApp → API Setup)
4. Este ID es distinto del número de teléfono real

#### 3. Activar el workflow y registrar el webhook

1. Abrir WF-1B en n8n y activarlo (toggle Active)
2. Copiar la URL del webhook desde el nodo WhatsApp Trigger
3. Ir a Meta for Developers → App → WhatsApp → Configuration → Webhook
4. Callback URL: URL del webhook de n8n
5. Verify Token: el valor configurado en la credencial trigger
6. Suscribir al campo: `messages`

### Respuestas esperadas en WhatsApp

| Tipo de mensaje | Respuesta |
| --- | --- |
| Texto: `hola` | `BSAgent ha recibido tu mensaje: hola` |
| Imagen, audio, documento | `BSAgent ha recibido un mensaje, pero esta fase solo procesa texto.` |
| Status update (delivered, read) | Sin respuesta (workflow se detiene en normalización) |

### Deduplicación — estado actual

La deduplicación por `message_id` está planificada pero pendiente:

- El nodo Data Store KV (`n8n-nodes-base.kv`) no está disponible en esta instalación
- Para la prueba inicial de conectividad esto es aceptable
- La deduplicación se añadirá en Fase 1 (WF-01) cuando esté disponible el KV store

### Errores comunes en Fase 1B

| Error | Causa probable | Solución |
| --- | --- | --- |
| Webhook no verifica | Workflow no activo o token incorrecto | Activar workflow antes de registrar webhook en Meta |
| 401 al enviar | Token de acceso inválido | Revisar credencial `whatsAppApi` en n8n |
| Variable no definida | `WHATSAPP_PHONE_NUMBER_ID` no configurada | Crear variable en n8n Settings → Variables |
| Mensajes no llegan | Campo `messages` no suscrito | Activar en Meta Developers → Webhook Fields |
| Número no autorizado | En test: número no en lista de contactos | Añadir número en Meta Developers → WhatsApp → Test Numbers |

### Confirmar solo lectura

Después de las pruebas de Fase 1B, confirmar en Google Calendar que no se ha creado,
modificado ni eliminado ningún evento. WF-1B no tiene nodos de Calendar ni de IA.

---

## Errores comunes

| Error | Causa probable | Solución |
|---|---|---|
| Webhook no verifica | URL no accesible o verify_token incorrecto | Revisar URL pública de n8n y token |
| Mensajes no llegan | Suscripción al campo `messages` no activa | Activar en Meta Developers |
| 401 Unauthorized | Token caducado o incorrecto | Regenerar token permanente |
| Número no autorizado | Destinatario no verificado en test | Añadir destinatario en Meta Developers |
