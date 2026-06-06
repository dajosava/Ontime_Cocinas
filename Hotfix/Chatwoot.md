# Incidente Técnico: Error de Envío de Mensajes Facebook Messenger en Chatwoot

## Resumen Ejecutivo

Se presentó una falla global en todos los canales de Facebook Messenger integrados con Chatwoot.

Los mensajes entrantes desde Facebook llegaban correctamente a Chatwoot, sin embargo, cualquier intento de responder desde la plataforma generaba errores de envío.

El problema afectaba a todas las páginas de Facebook conectadas.

---

# Entorno

## Plataforma

* Chatwoot v4.8.0
* Ruby on Rails
* Facebook Messenger
* Meta Graph API

## Servidor

```bash
/home/chatwoot/chatwoot
```

## Servicio involucrado

```bash
/home/chatwoot/chatwoot/app/services/facebook/send_on_facebook_service.rb
```

---

# Síntomas Observados

## Comportamiento reportado

Los usuarios enviaban mensajes desde Facebook Messenger y:

✅ Los mensajes ingresaban correctamente a Chatwoot

❌ Las respuestas generadas desde Chatwoot fallaban inmediatamente

❌ Chatwoot mostraba:

```text
Error al enviar mensaje
```

---

# Investigación Realizada

## Validación de Tokens

Se verificó:

* Page Access Tokens
* Estado de la aplicación Meta
* Permisos Messenger
* Estado Live de la aplicación

Resultado:

```text
Tokens válidos
Aplicación en modo Live
Permisos correctos
```

---

## Validación de Webhooks

Se revisaron los logs:

```bash
journalctl -u chatwoot-web.1.service -f
```

Resultado:

```text
Los mensajes entrantes llegaban correctamente.
```

Conclusión:

```text
El webhook no era el problema.
```

---

## Validación de PSID

Se realizaron pruebas directas contra Meta utilizando:

```ruby
https://graph.facebook.com/v18.0/me/messages
```

Inicialmente algunos PSID devolvían:

```json
{
  "error": {
    "message": "(#100) No se encontró al usuario correspondiente",
    "error_subcode": 2018001
  }
}
```

Posteriormente se probó con un contacto activo y reciente.

---

## Prueba Directa contra Meta

Payload utilizado:

```json
{
  "recipient": {
    "id": "28112143165053912"
  },
  "message": {
    "text": "test"
  },
  "messaging_type": "RESPONSE"
}
```

Resultado:

```http
HTTP 200 OK
```

Respuesta:

```json
{
  "recipient_id": "28112143165053912",
  "message_id": "m_xxxxxxxxx"
}
```

Conclusión:

```text
Meta aceptaba mensajes correctamente.
Los tokens eran válidos.
Los PSID eran válidos.
El problema estaba dentro de Chatwoot.
```

---

# Análisis de Código

Se inspeccionó el archivo:

```bash
/home/chatwoot/chatwoot/app/services/facebook/send_on_facebook_service.rb
```

Se encontró la siguiente implementación:

```ruby
def fb_text_message_params
  {
    recipient: { id: contact.get_source_id(inbox.id) },
    message: fb_text_message_payload,
    messaging_type: 'MESSAGE_TAG',
    tag: 'ACCOUNT_UPDATE'
  }
end
```

Y para adjuntos:

```ruby
def fb_attachment_message_params(attachment)
  {
    recipient: { id: contact.get_source_id(inbox.id) },
    message: {
      attachment: {
        type: attachment_type(attachment),
        payload: {
          url: attachment.download_url
        }
      }
    },
    messaging_type: 'MESSAGE_TAG',
    tag: 'ACCOUNT_UPDATE'
  }
end
```

---

# Causa Raíz

Chatwoot enviaba todos los mensajes utilizando:

```ruby
messaging_type: 'MESSAGE_TAG'
tag: 'ACCOUNT_UPDATE'
```

Meta actualmente restringe el uso de:

```text
ACCOUNT_UPDATE
```

Este tag solamente puede utilizarse para eventos específicos relacionados con cuentas de usuario.

No puede utilizarse para:

* Atención al cliente
* Conversaciones normales
* Ventas
* Seguimientos comerciales

Por esta razón Meta rechazaba los mensajes generados por Chatwoot.

---

# Solución Implementada

Se modificó el archivo:

```bash
/home/chatwoot/chatwoot/app/services/facebook/send_on_facebook_service.rb
```

## Antes

```ruby
messaging_type: 'MESSAGE_TAG',
tag: 'ACCOUNT_UPDATE'
```

## Después

```ruby
messaging_type: 'RESPONSE'
```

---

## Método corregido

```ruby
def fb_text_message_params
  {
    recipient: { id: contact.get_source_id(inbox.id) },
    message: fb_text_message_payload,
    messaging_type: 'RESPONSE'
  }
end
```

---

## Método de adjuntos corregido

```ruby
def fb_attachment_message_params(attachment)
  {
    recipient: { id: contact.get_source_id(inbox.id) },
    message: {
      attachment: {
        type: attachment_type(attachment),
        payload: {
          url: attachment.download_url
        }
      }
    },
    messaging_type: 'RESPONSE'
  }
end
```

---

# Reinicio de Servicios

Después del cambio se reiniciaron los servicios:

```bash
sudo systemctl restart chatwoot-web.1
sudo systemctl restart chatwoot-worker.1
```

---

# Resultado Final

## Antes

```text
Mensajes entrantes: OK
Mensajes salientes: FAIL
```

## Después

```text
Mensajes entrantes: OK
Mensajes salientes: OK
```

---

# Validación Final

Prueba realizada desde Chatwoot:

```text
Usuario escribe desde Facebook Messenger
↓
Mensaje llega a Chatwoot
↓
Agente responde desde Chatwoot
↓
Meta acepta mensaje
↓
Cliente recibe respuesta
```

Resultado:

```text
HTTP 200 OK
```

---

# Estado Final

✅ Problema identificado

✅ Causa raíz encontrada

✅ Código corregido

✅ Facebook Messenger operativo nuevamente

✅ Todos los inboxes afectados recuperados

Fecha de resolución: 06-Jun-2026 5:19 PM
