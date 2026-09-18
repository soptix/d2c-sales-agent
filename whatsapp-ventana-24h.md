# Responder fuera de la ventana de 24 horas

Propuesta de solución para WhatsApp, Instagram y Messenger. Estado al 14 de agosto de 2026.

Los dos primeros bloques son de WhatsApp, que es el canal con volumen real. Instagram y Messenger funcionan con un mecanismo distinto y tienen su propia sección más abajo.

## El problema, con precisión

Meta cierra la conversación 24 horas después del último mensaje del cliente. Pasado ese punto, cualquier mensaje libre se rechaza con **error 131047 (re-engagement message)**. Hoy el puente lo intenta igual, YCloud devuelve el error, y el puente escribe una nota privada. El mensaje nunca sale y Chatwoot lo muestra como enviado.

Son tres casos distintos que hoy se tratan igual:

| Caso | Frecuencia | Hoy |
|---|---|---|
| La asesora responde a un lead que escribió hace 2 días | alta | falla |
| La asesora hace seguimiento comercial por iniciativa propia | alta | falla |
| El bot responde | siempre | funciona, contesta en segundos |

El bot nunca toca el límite. El problema es exclusivamente de la asesora humana, justo en los dos momentos que cierran ventas.

### El caso del fin de semana

Es el disparador más frecuente de todo esto, y conviene tenerlo claro porque se presta a un malentendido.

**La ventana la marca el último mensaje del cliente. Nada más.** No la mueve asignar la conversación, ni que una asesora la abra, ni que el bot responda. Asignar es una acción interna de Chatwoot, no sale hacia Meta y no reactiva nada. Asignar el lunes funciona exactamente igual que asignar el sábado.

Y asignar el sábado sería peor: el agente de IA ignora las conversaciones asignadas (condición `conversation.meta.assignee` en `Is Chatwoot inbound text?`), así que el cliente pasaría el fin de semana en silencio en vez de recibir respuesta del bot en segundos.

Qué pasa entonces con un mensaje del sábado que se atiende el lunes:

| Canal | Sábado 3pm → lunes 8am (41 horas) |
|---|---|
| Instagram / Messenger | entra sin problema, la ventana con `HUMAN_AGENT` es de 7 días |
| WhatsApp | fuera de ventana, exige plantilla |

En WhatsApp la cuenta es simple: **un mensaje del sábado siempre queda fuera** el lunes. Uno del domingo por la tarde suele salvarse, porque domingo 3pm más 24 horas llega hasta el lunes 3pm.

Esto es lo que hace que la Pieza 2 valga la pena. Sin ella, cada lunes en la mañana las asesoras escriben mensajes que no salen. Con ella, el lunes escriben normal, el cliente recibe la plantilla, toca "Sí, cuéntame", y el mensaje de la asesora se entrega solo.

**El reloj de los 7 días tampoco se extiende.** Corre desde el último mensaje del cliente, así que un mensaje del sábado en Instagram hay que responderlo antes del sábado siguiente. Solo se reinicia cuando el cliente vuelve a escribir.


## La única salida legal: plantillas aprobadas

No hay workaround. Fuera de la ventana, Meta solo entrega **mensajes de plantilla aprobados previamente**. Todo lo demás se rechaza. Así que la solución no es esquivar el límite, es darle al puente soporte de plantillas y hacer que la ventana sea visible antes de que la asesora escriba.

Detalle importante que juega a favor: cuando el cliente responde a la plantilla (o toca un botón de respuesta rápida), la ventana se reabre y a partir de ahí se puede escribir libre otra vez.

## Diseño propuesto

**Corrección frente a la primera versión de este documento.** Verifiqué el código de Chatwoot v4.16, el que corre en el servidor, y trae soporte nativo de plantillas y de ventana de respuesta **también para canales API**. El comentario está explícito en `ReplyBox.vue`:

> `// We support templates for API channels if someone updates templates manually via API`

Eso elimina los atributos personalizados y el cron que proponía la Pieza 1 original. El trabajo real baja a configurar dos cosas y agregar una rama en el puente.

### Pieza 1: encender la ventana nativa del canal API

`Channel::Api` acepta un atributo `agent_reply_time_window`, en horas. Con él, `Conversations::MessageWindowService` calcula `can_reply` a partir del **último mensaje entrante** de la conversación, que es exactamente el mismo criterio de Meta.

```
PATCH /api/v1/accounts/1/inboxes/1
{ "channel": { "additional_attributes": { "agent_reply_time_window": "23" } } }
```

23 y no 24, para dejar una hora de margen frente al desfase de reloj entre el servidor (`Etc/UTC`), n8n (`America/Bogota`) y Meta.

Qué pasa en la interfaz cuando se vence: el editor **se deshabilita** y muestra *"You can only reply using a template message due to message window restriction"*. El botón de plantillas queda como única salida. La pestaña de nota privada sigue habilitada, así que las asesoras pueden seguir dejando notas internas.

Esto reemplaza por completo los atributos `ultimo_inbound` y `ventana_whatsapp` y el cron de cierre. Es una llamada de API, no hay nada que construir ni que mantener.

### Pieza 2: cargar las plantillas aprobadas en el inbox

Las plantillas viven en `additional_attributes.message_templates` del canal, en el formato de Meta:

```
PATCH /api/v1/accounts/1/inboxes/1
{
  "channel": {
    "additional_attributes": {
      "agent_reply_time_window": "23",
      "message_templates": [
        {
          "id": "1",
          "name": "reapertura_asesor",
          "status": "approved",
          "language": "es",
          "category": "MARKETING",
          "components": [
            { "type": "BODY", "text": "Hola {{1}}, te escribe {{2}} de Constructora Aurora. Quedó pendiente tu consulta sobre {{3}} y te tengo la información. Responde a este mensaje y seguimos por aquí." }
          ]
        }
      ]
    }
  }
}
```

El `PATCH` reemplaza el objeto completo, así que hay que mandar `agent_reply_time_window` junto con las plantillas o se pierde.

Filtros que aplica el selector, verificados en `getFilteredWhatsAppTemplates`. Una plantilla que no los cumpla simplemente no aparece, sin mensaje de error:

- `status` debe ser `approved` (no distingue mayúsculas).
- Se descartan las de categoría `AUTHENTICATION`.
- Se descartan las que se llaman `customer_satisfaction_survey*`.
- Se descartan las que traen componentes `LIST`, `PRODUCT`, `CATALOG`, `CALL_PERMISSION_REQUEST`, o un `HEADER` de formato `LOCATION`.
- Tiene que traer `status` y `components`, o se ignora.

**Mantenimiento:** esta lista no se sincroniza sola, porque YCloud no es un proveedor nativo. Vale la pena un workflow diario que lea las plantillas de YCloud y haga el `PATCH`, para que un cambio de estado en Meta (una plantilla pausada, por ejemplo) no deje a las asesoras enviando algo que ya no existe.

### Pieza 3: una rama en el puente

Es el único desarrollo obligatorio.

Cuando la asesora envía desde el selector, Chatwoot crea el mensaje con el texto ya renderizado y guarda en `additional_attributes.template_params`:

```json
{ "name": "reapertura_asesor", "category": "MARKETING", "language": "es",
  "namespace": "...", "processed_params": { "1": "Juan", "2": "Ana", "3": "Villa Serena" } }
```

`Message#webhook_data` incluye `additional_attributes`, así que eso llega tal cual al webhook `chatwoot-outbound` que ya consume el puente. No hay que tocar nada del lado de Chatwoot.

En el nodo `Enviar por YCloud`, una condición sobre `body.additional_attributes.template_params`:

- **Si viene:** payload de plantilla (`type: "template"`, con `name`, `language` y los `processed_params` en orden dentro de `components[].parameters`).
- **Si no viene:** el payload de texto o adjunto de hoy, sin cambios.

### Pieza 4 (opcional): envío automático con mensaje en cola

Con las piezas 1 a 3, la asesora ya puede responder fuera de ventana, pero tiene que elegir la plantilla a mano. La automatización de la primera versión de este documento sigue teniendo sentido para un caso concreto: el lunes en la mañana, cuando hay decenas de conversaciones vencidas del fin de semana.

La idea es la misma: la asesora escribe normal, el puente detecta que la ventana está cerrada, envía la plantilla de reapertura, guarda el texto en cola en un atributo de conversación, y lo entrega solo cuando el cliente responde.

```
Asesora escribe con la ventana cerrada
  ├─ se envía la plantilla de reapertura
  ├─ el texto queda en cola en la conversación
  └─ nota privada: "tu mensaje sale apenas el cliente responda"

Cliente responde
  ├─ se entrega el mensaje en cola
  ├─ se limpia la cola
  └─ nota privada: "tu mensaje pendiente ya se entregó"
```

Tres reglas para que no se vuelva spam:

1. **Máximo una plantilla de reapertura por conversación cada 24 horas**, con el atributo `ultimo_template_at`. Cinco mensajes seguidos de la asesora producen una plantilla y cinco textos en la cola.
2. **Solo mensajes de humano disparan plantilla.** El bot nunca reabre una conversación.
3. **Al encolar, si la conversación está sin asignar, se asigna a quien escribió.** El agente de IA ignora las conversaciones asignadas (condición `conversation.meta.assignee` en `Is Chatwoot inbound text?`), así que esto evita que el bot y el mensaje en cola salgan a la vez.

Choca con la Pieza 1: si el editor está deshabilitado, la asesora no puede escribir texto libre para encolar. O se hace esto, o se hace la Pieza 1, no las dos sobre el mismo inbox. **Recomiendo empezar por las piezas 1 a 3**, que son configuración más una rama, y evaluar la Pieza 4 después de ver cuántas conversaciones vencidas llegan cada lunes.

### Pieza 5: qué plantillas aprobar

Se crean y se someten desde YCloud. La aprobación de Meta toma de minutos a 24 horas. Empezamos con dos.

**`reapertura_asesor`** (la que usa el puente de forma automática)

> Hola {{1}}, te escribe {{2}} de Constructora Aurora. Quedó pendiente tu consulta sobre {{3}} y te tengo la información. Responde a este mensaje y seguimos por aquí.

Variables: `1` nombre del cliente, `2` nombre de la asesora, `3` proyecto de interés.
Botón de respuesta rápida: **"Sí, cuéntame"**. Tocarlo cuenta como mensaje entrante y reabre la ventana con un toque, sin que el cliente tenga que escribir.

**`seguimiento_lead`** (seguimiento comercial por iniciativa de la asesora)

> Hola {{1}}, soy {{2}}, asesora de Constructora Aurora. Te contacto por tu interés en {{3}}. ¿Quieres que te comparta disponibilidad y la estructura de pago?

Reglas de contenido, no negociables:

- **Nada de precios, porcentajes ni condiciones de pago dentro de la plantilla.** El texto de plantilla es fijo y no pasa por el catálogo, así que meter cifras ahí rompe la regla de "nunca inventes ni calcules un dato" del prompt del agente.
- Nada que suene a que la constructora financia. Mismo motivo que la sección de financiación del prompt.
- El nombre correcto es siempre "Constructora Aurora" o "Constructora Bahia", escritos exactamente así.
- Meta rechaza plantillas que terminan en variable o que tienen dos variables seguidas. Los dos borradores cumplen.

### Pieza 6: red de seguridad por código de error

El nodo `¿Falló el envío?` se queda. Si el error de YCloud trae **131047**, la nota privada debe decir con claridad que la ventana venció y que hay que usar una plantilla, no el texto genérico de hoy. Cubre el caso de que `agent_reply_time_window` y Meta no coincidan por un borde de minutos.

## Cómo se ve para la asesora

Paso a paso, con las piezas 1 a 3 puestas:

1. Abre una conversación de WhatsApp cuyo último mensaje del cliente es de hace más de 23 horas.
2. El cuadro de respuesta aparece **deshabilitado**, con el aviso de que solo puede responder con una plantilla. No hay forma de equivocarse ni de escribir algo que no salga.
3. Toca el botón de plantillas en la barra del composer.
4. Ve la lista de plantillas aprobadas, cada una con su idioma, categoría y el texto del cuerpo.
5. Elige una y llena un formulario con las variables: nombre del cliente, su propio nombre, proyecto.
6. Ve la vista previa con las variables ya reemplazadas y envía.
7. El mensaje sale, queda en el hilo como cualquier otro, y el cliente lo recibe.
8. Cuando el cliente responde, la ventana se reabre sola y el editor se habilita otra vez.

No hay pantalla nueva, ni botón custom, ni entrenamiento más allá de "cuando aparezca deshabilitado, usa plantilla".

## Cambios concretos en el puente

Sobre [n8n-workflows/D2C — Puente YCloud ↔ Chatwoot.json](n8n-workflows/D2C%20—%20Puente%20YCloud%20↔%20Chatwoot.json), 21 nodos hoy. Para las piezas 1 a 3 basta con esto:

| Nodo | Cambio |
|---|---|
| `Separar adjuntos` | pasar `template_params` al item de salida, leyéndolo de `body.additional_attributes` |
| `¿Es plantilla?` (nuevo) | IF sobre `template_params.name` |
| `Enviar por YCloud` | segunda variante del `jsonBody` con `type: "template"`. Mismo `from`, misma credencial, misma URL |

Cuerpo del envío de plantilla, verificado contra la documentación de YCloud:

```json
{
  "from": "+57XXXXXXXXXX",
  "to": "{{telefono}}",
  "type": "template",
  "template": {
    "name": "reapertura_asesor",
    "language": { "code": "es", "policy": "deterministic" },
    "components": [
      { "type": "body", "parameters": [
        { "type": "text", "text": "{{processed_params.1}}" },
        { "type": "text", "text": "{{processed_params.2}}" },
        { "type": "text", "text": "{{processed_params.3}}" }
      ]}
    ]
  }
}
```

La Pieza 4, si se hace, agrega los nodos de cola: lectura de la conversación, escritura de `mensaje_en_cola` y `ultimo_template_at`, entrega diferida en la rama entrante y asignación automática.

## Costo

Colombia es de los mercados más baratos de Meta. Tarifas por mensaje entregado:

| Categoría | Colombia |
|---|---|
| Utility / Authentication | ~$0,0008 USD |
| Marketing | ~$0,02 USD |
| Servicio (respuesta libre dentro de la ventana) | gratis hasta el 30 de septiembre de 2026 |

Meta clasifica la categoría, no nosotros. `reapertura_asesor` puede caer en marketing porque retoma una conversación comercial. Aun en el peor caso, 200 reaperturas al mes cuestan **4 dólares**. No es un factor de decisión.

**Dos cambios de tarifa a vigilar este año:**

- **1 de octubre de 2026:** los mensajes de servicio dentro de la ventana dejan de ser gratis y pasan a cobrarse a la tarifa de utility. Eso empieza a costar plata por cada respuesta del bot, no solo por las plantillas. A ~$0,0008 por mensaje sigue siendo del orden de centavos al mes con el volumen actual, pero conviene medirlo.
- Meta publica las tarifas exactas por mercado antes del 1 de septiembre de 2026. Confirmar contra el rate card de YCloud antes de dar cifras al cliente.

## Riesgos y límites

| Riesgo | Mitigación |
|---|---|
| Si Meta clasifica la plantilla como marketing, aplica el límite de plantillas de marketing por usuario y algunos mensajes se descartan **en silencio** | Ingerir el webhook de estado de mensaje de YCloud (`sent`, `delivered`, `failed`) y reflejarlo en la conversación. El puente hoy no lo hace, es trabajo aparte pero necesario |
| El quality rating de la plantilla baja si la gente bloquea o reporta, y Meta la pausa | Enviar reapertura solo cuando hay un mensaje real de asesora pendiente. Nunca en masa, nunca sin motivo |
| La plantilla se aprueba con un texto y el negocio quiere otro | Cada cambio de texto exige nueva aprobación. Dejar el texto quieto y variar por las variables |
| Adjuntos fuera de ventana | La cola de v1 es solo texto. Si la asesora manda una foto fuera de ventana, se envía la plantilla y la nota privada le dice que la reenvíe cuando el cliente conteste |
| El servidor es un portátil que se duerme | Si la rama entrante no corre, `ultimo_inbound` queda viejo y todo se ve "cerrado". La Pieza 4 (error 131047) es lo que evita que eso rompa el flujo |

## Plan de implementación

1. Crear `reapertura_asesor` y `seguimiento_lead` en YCloud y someterlas a Meta → verificar: las dos quedan en estado `APPROVED` y se registra la categoría que Meta les asignó.
2. `PATCH` del inbox 1 con `agent_reply_time_window: "23"` → verificar: en una conversación con último entrante de hace más de 23 horas, el editor aparece deshabilitado con el aviso de plantilla.
3. `PATCH` del inbox 1 con `message_templates` → verificar: el botón de plantillas aparece y lista las dos, con idioma y categoría correctos.
4. Rama de plantilla en el puente → verificar: enviar `reapertura_asesor` desde la UI a un teléfono de prueba fuera de ventana y que llegue con las variables bien reemplazadas.
5. Afinar la nota privada del error 131047 → verificar: forzando el fallo, la nota dice que hace falta plantilla.
6. Workflow diario de sincronización de plantillas desde YCloud → verificar: pausar una plantilla en Meta la saca del selector en menos de 24 horas.
7. Medir cuántas conversaciones llegan vencidas cada lunes → decidir si la Pieza 4 vale la pena.
8. Actualizar la tabla de limitaciones en [chatwoot-deployment.md](chatwoot-deployment.md) y el README de workflows.

Los pasos 1 a 4 son el mínimo que resuelve el problema, y son configuración más una rama en un nodo. La primera versión de este documento estimaba mucho más trabajo del necesario.

---

# Instagram y Messenger

## No hay plantillas. Hay un tag, y Chatwoot ya lo trae hecho

Meta no ofrece plantillas fuera de la ventana en Instagram ni en Messenger. El mecanismo es otro: el **tag `HUMAN_AGENT`**, que extiende la ventana de 24 horas a **7 días** cuando responde una persona.

Comparación de los dos canales:

| | WhatsApp | Instagram / Messenger |
|---|---|---|
| Ventana base | 24 horas | 24 horas |
| Extensión | ninguna | 7 días con tag `HUMAN_AGENT` |
| Reapertura fuera de ventana | plantilla aprobada, se paga por mensaje | no existe |
| Costo | ~$0,0008 a $0,02 USD por plantilla | gratis |
| Trabajo de desarrollo | el puente de n8n (secciones de arriba) | **cero** |
| Qué lo bloquea | nada, se puede hacer ya | App Review de Meta |

**El trabajo de desarrollo es cero.** Chatwoot v4.16 ya implementa el tag de forma nativa en `Instagram::SendOnInstagramService` y `Facebook::SendOnFacebookService`:

```ruby
def merge_human_agent_tag(params)
  global_config = GlobalConfig.get('ENABLE_INSTAGRAM_CHANNEL_HUMAN_AGENT')
  return params unless global_config['ENABLE_INSTAGRAM_CHANNEL_HUMAN_AGENT']

  params[:messaging_type] = 'MESSAGE_TAG'
  params[:tag] = 'HUMAN_AGENT'
  params
end
```

Son dos flags, uno por canal: `ENABLE_INSTAGRAM_CHANNEL_HUMAN_AGENT` y `ENABLE_MESSENGER_CHANNEL_HUMAN_AGENT`.

Segunda buena noticia: en estos canales, cuando Meta rechaza el envío, Chatwoot llama a `Messages::StatusUpdateService` y **el mensaje queda marcado como fallido en la interfaz**. No pasa lo del puente de WhatsApp, que muestra como enviado algo que nunca salió. La asesora se entera sola.

## El candado real es de permisos, no de código

Encender el flag sin el permiso aprobado hace que Meta rechace **todos** los mensajes, no solo los de fuera de ventana. El orden importa:

1. **Verificación de negocio de Meta** (cámara de comercio + RUT). Es el pendiente 6 de [chatwoot-deployment.md](chatwoot-deployment.md) y ya es el camino crítico del proyecto.
2. **App Review para Advanced Access de mensajería de Instagram.** Sin esto, Instagram solo responde a cuentas con rol en la app. O sea, hoy la ventana de 24 horas ni siquiera es el problema: el canal no atiende clientes reales.
3. **App Review para el feature Human Agent.** Es una solicitud aparte, y Meta solo la aprueba a apps que ya pasaron el review de mensajería. Va después, no en paralelo.
4. **Encender el flag** en la instancia.

Conclusión práctica: **la ventana de Instagram no se arregla antes de que salga la verificación de negocio.** No hay nada que construir mientras tanto, y sí hay algo que empujar.

## Trampa al encender el flag

`GlobalConfig.get` lee de la tabla **`installation_configs`**, no de `ENV`. Es exactamente la trampa que ya costó las variables `IG_*` y `FB_*` (ver "las variables de Meta NO se leen del `.env`" en [chatwoot-deployment.md](chatwoot-deployment.md)). Agregarlo al `.env` y reiniciar **no hace nada**.

Se enciende por Super Admin, en Settings → Configuration, o con el mismo runner de Rails que ya se usó:

```ruby
%w[ENABLE_INSTAGRAM_CHANNEL_HUMAN_AGENT ENABLE_MESSENGER_CHANNEL_HUMAN_AGENT].each do |k|
  ic = InstallationConfig.find_or_initialize_by(name: k)
  ic.value = true
  ic.save!
end
GlobalConfig.clear_cache
```

## Riesgo de cumplimiento: el flag es global y el bot también queda marcado

Este es el punto que hay que decidir con la cabeza fría.

El flag no distingue quién escribe. Cuando está encendido, **cada mensaje saliente** del inbox lleva `tag: HUMAN_AGENT`, incluidos los del agente de IA. Y Meta prohíbe explícitamente ese tag para mensajes automatizados, dice que detecta el abuso, y lo limita a soporte, no a promoción.

Lo que juega a favor: el bot solo responde a mensajes entrantes, en segundos, así que **sus mensajes siempre caen dentro de las 24 horas** y se habrían entregado igual sin el tag. El tag sobra en ellos, no los habilita a nada. El uso real de la extensión sigue siendo humano.

Lo que no se puede hacer en v4.16: controlarlo por mensaje. La API de Chatwoot no expone `messaging_type`, así que o está para todos o para nadie. Un parche al servicio para saltarse el tag cuando el remitente es el bot es viable pero cuesta mantener un fork de Chatwoot. No lo recomiendo por ahora.

Decisión sugerida: encenderlo cuando el permiso esté aprobado, y dejar registrado el motivo. Si Meta llegara a objetar, el argumento es verificable: el tag solo cambia el resultado en mensajes escritos por una asesora.

## Más allá de 7 días no hay nada

Pasados los 7 días, la conversación de Instagram se cierra sin salida técnica. Las opciones reales:

- **Marketing Messages API.** Existe, pero exige un opt-in explícito capturado **dentro** de la ventana. Es otro producto y otro flujo, y no está en alcance.
- **Esperar a que el cliente escriba.**
- **Mover la conversación a WhatsApp.** Es lo que sirve hoy.

## La solución operativa: sacar el teléfono temprano

Para una operación de venta, la respuesta más sólida a la ventana de Instagram no es técnica. Es que el lead termine en WhatsApp, que es el único canal con un mecanismo pagado de reapertura y el que las asesoras usan de todos modos.

El prompt del agente ya lo hace: por Instagram y Messenger pide un teléfono de contacto junto con el nombre (ver "Captura de datos (lead)" en [system-prompt-v3.md](system-prompt-v3.md)). Vale la pena reforzar dos cosas ahí:

- Pedir el teléfono **antes** de que la conversación se enfríe, no al final. Con la ventana de 7 días hay margen, pero el margen se acaba.
- Cuando el lead da el teléfono y muestra interés real, que la asesora abra el hilo de WhatsApp escribiendo desde el número de la constructora dentro de esos 7 días. Ahí el cliente responde, se abre la ventana de WhatsApp, y a partir de ese punto aplica todo el mecanismo de plantillas de la primera mitad de este documento.

Eso convierte el problema de Instagram en un problema de WhatsApp, que sí está resuelto.

## Plan para Instagram y Messenger

1. Empujar la verificación de negocio de Meta (cámara de comercio + RUT del cliente) → verificar: el Business Manager muestra el negocio como verificado.
2. App Review de mensajería de Instagram, para Advanced Access → verificar: una cuenta de Instagram sin rol en la app logra conversar con el inbox.
3. Solicitar el feature Human Agent en la misma app → verificar: aparece aprobado en el panel de la app.
4. Encender los dos flags por `installation_configs` y limpiar la caché → verificar: `GlobalConfig.get('ENABLE_INSTAGRAM_CHANNEL_HUMAN_AGENT')` devuelve true en la consola de Rails.
5. Prueba real: conversación de Instagram con último mensaje del cliente de hace más de 24 horas y menos de 7 días, respuesta de asesora → verificar: llega al teléfono y no queda marcada como fallida en Chatwoot.
6. Reforzar en el prompt la captura temprana de teléfono en Instagram y Messenger → verificar: en 10 conversaciones de Instagram con interés real, el teléfono queda registrado en HubSpot.

El paso 6 es el único que se puede hacer hoy. Del 1 al 5 dependen de Meta.


---

---

## Lo que esto no resuelve

- **Contacto en frío a listas.** Escribirle a gente que nunca ha hablado con el número es otra cosa: exige opt-in registrado y un flujo de campaña aparte. No está en esta propuesta.
- **Instagram y Messenger más allá de los 7 días.** No hay mecanismo. Ver la sección de Instagram arriba.
- **Acuses de entrega y lectura.** Siguen sin reflejarse. La primera fila de la tabla de riesgos empieza a atacarlo.

## Fuentes

- [Send a message directly, YCloud](https://docs.ycloud.com/reference/whatsapp_message-send-directly)
- [WhatsApp Messaging Examples, YCloud](https://docs.ycloud.com/reference/whatsapp-messaging-examples)
- [Update Custom Attributes, Chatwoot](https://developers.chatwoot.com/api-reference/conversations/update-custom-attributes)
- [Error 131047 re-engagement message](https://www.heltar.com/blogs/troubleshooting-whatsapp-api-error-131047-re-engagement-message-required-cm5fctgfx000mkg77rgedcamx)
- [Cambio de tarifas del 1 de octubre de 2026](https://www.hello-charles.com/blog/whatsapp-service-message-pricing-what-changes-in-2026)
- [Tarifas por país](https://formbeep.com/whatsapp-api-pricing/)
- [Human Agent tag en Instagram y Messenger, Chatwoot](https://www.chatwoot.com/hc/user-guide/articles/1745225158-what-is-human-agent-tag-in-instagram-messenger-channel)
- [Instagram App Review, Chatwoot self-hosted](https://developers.chatwoot.com/self-hosted/instagram-app-review)
- [Messenger Platform e IG Messaging API policy, Meta](https://developers.facebook.com/documentation/business-messaging/messenger-platform/policy)
- Código: `app/services/instagram/send_on_instagram_service.rb`, `app/services/facebook/send_on_facebook_service.rb`, `app/services/conversations/message_window_service.rb`, `app/models/channel/api.rb`, `app/models/message.rb` y `app/javascript/dashboard/components/widgets/conversation/ReplyBox.vue`, tag `v4.16.0`
