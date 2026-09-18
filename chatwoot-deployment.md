# Chatwoot — despliegue Aurora

Instancia self-hosted para el agente de ventas D2C. Estado al **21 de julio de 2026**.

**URL:** https://chatwoot.example.com — en línea, HTTPS con certificado válido.

---

## Arquitectura

```
Instagram / Messenger                    WhatsApp
   (Meta, nativo)                     (YCloud, vía puente)
        ↓ webhooks                          ↓ webhook
  Cloudflare (TLS)                    n8n Cloud (puente)
        ↓ túnel QUIC                        ↓ API pública
  cloudflared → 127.0.0.1:3000  ←───────────┘
        ↓
  Chatwoot (rails + sidekiq + postgres + redis)
        ↓ webhook message_created
  n8n Cloud → Sales Agent → respuesta por API de Chatwoot
        ↓ (si el inbox es API) webhook_url
  n8n Cloud → YCloud sendDirectly → WhatsApp
```

**Los tres canales llegan a la misma bandeja, pero por dos caminos distintos.** Instagram y Messenger entran nativos desde Meta. WhatsApp entra por un canal API genérico con un puente en n8n, porque Chatwoot solo soporta 360dialog y Meta Cloud API como proveedores de WhatsApp — YCloud no está entre ellos. Ver la sección del puente más abajo.

**No hay puertos abiertos en el firewall del cliente.** Todo el tráfico entrante llega por el túnel saliente de Cloudflare. Esto se eligió porque el 80/443 público de esa IP ya está tomado por otra máquina (Windows + XAMPP) y no se podía disputar.

## Servidor

| | |
|---|---|
| Acceso | `ssh -p <PUERTO> <usuario>@<IP_DEL_SERVIDOR>` |
| Hostname / IP privada | `<hostname>` / `192.168.1.x` (NAT) |
| OS | Ubuntu 22.04.1 LTS |
| CPU / RAM / Disco | 2 núcleos AMD A4-9125 / 7.2 GB / 98 GB |
| Usuario | `<usuario>` (en grupo `sudo`) |

⚠️ **Es un portátil**, no un servidor. Chip de laptop, wifi, sesión gráfica y usuario logueado en consola física. Antes del reinicio del 21/07 llevaba 82 días encendido.

## Instalado

| Componente | Versión | Ruta |
|---|---|---|
| Docker CE + Compose | 29.6.2 / v5.3.1 | — |
| Chatwoot | **v4.16.0-ce** (fijada, no `latest`) | `/opt/chatwoot` |
| Postgres | pgvector/pgvector:pg16 | volumen `chatwoot_postgres_data` |
| Redis | redis:alpine | volumen `chatwoot_redis_data` |
| cloudflared | 2026.7.2 | servicio systemd |

Todos los contenedores con `restart: always` y escuchando **solo en `127.0.0.1`**.
`chatwoot-base-1` aparece como `Exited (0)` — es normal, es el ancla YAML del compose, no un servicio real.

## Configuración

`/opt/chatwoot/.env` (permisos `600`). Secretos generados en la máquina con `openssl`.

```
FRONTEND_URL=https://chatwoot.example.com
FORCE_SSL=true
DEFAULT_LOCALE=es
ENABLE_ACCOUNT_SIGNUP=false
INSTALLATION_ENV=docker
```

Verificado que `FORCE_SSL=true` no genera bucle de redirección: el túnel pasa correctamente `X-Forwarded-Proto` y la cookie de sesión sale con flag `secure`.

## Canales sociales — configuración Meta

### ⚠️ Trampa: las variables de Meta NO se leen del `.env`

Los controladores de webhooks usan `GlobalConfigService.load('IG_VERIFY_TOKEN', '')`, que lee de la tabla **`installation_configs` en la base de datos**, no de `ENV`. Esa tabla se siembra durante `db:chatwoot_prepare`, así que **cualquier variable agregada al `.env` después de la instalación inicial queda ignorada, incluso reiniciando los contenedores.**

Síntoma: la verificación de webhook de Meta falla con `401 wrong verify token` aunque `printenv` dentro del contenedor muestre el valor correcto.

Tras editar `FB_*` / `IG_*` en el `.env`, hay que sincronizar:

```ruby
# sudo docker compose exec rails bundle exec rails runner /tmp/sync.rb
%w[IG_VERIFY_TOKEN FB_VERIFY_TOKEN FB_APP_ID FB_APP_SECRET].each do |k|
  v = ENV[k].to_s
  next if v.empty?
  ic = InstallationConfig.find_or_initialize_by(name: k)
  ic.value = v
  ic.save!
end
GlobalConfig.clear_cache
```

### Valores registrados en la app de Meta

| Campo | Valor |
|---|---|
| Webhook Instagram | `https://chatwoot.example.com/webhooks/instagram` |
| OAuth redirect Instagram | `https://chatwoot.example.com/instagram/callback` |
| Webhook Messenger | `https://chatwoot.example.com/bot` |
| Campo a suscribir | `messages` |

Verify tokens generados y ya sincronizados a la base. Están en `/opt/chatwoot/.env`.

Verificación del endpoint (token correcto devuelve el challenge, incorrecto da 401):

```bash
curl "https://chatwoot.example.com/webhooks/instagram?hub.mode=subscribe&hub.verify_token=<TOKEN>&hub.challenge=PRUEBA"
```

### Advanced Access

El permiso de mensajería de Instagram requiere **Advanced Access** vía App Review de Meta, que normalmente exige verificación de negocio. Sin eso, solo funcionan las cuentas con rol en la app (admin/developer/tester) — suficiente para probar el circuito completo, insuficiente para clientes reales.

---

## WhatsApp — puente YCloud vía canal API

### Por qué no es un canal nativo

`Channel::Whatsapp::PROVIDERS` en v4.16 es `%w[default whatsapp_cloud]` — 360dialog y Meta Cloud API. **YCloud no es soportado.** Como el cliente ya tenía conexión operativa con YCloud y Meta Cloud API exigía verificación de negocio (que no está), se optó por un canal **API genérico** con un puente en n8n.

### Recursos creados en Chatwoot

| | |
|---|---|
| Inbox | id `1`, nombre "WhatsApp (YCloud)", tipo `Channel::Api` |
| `inbox_identifier` | `<CHATWOOT_INBOX_IDENTIFIER>` |
| `hmac_token` | `<CHATWOOT_HMAC_TOKEN>` |
| `webhook_url` | `https://example.app.n8n.cloud/webhook/chatwoot-outbound` |
| Agent bot | id `1`, "Sales Agent Bridge", token `<CHATWOOT_AGENT_BOT_TOKEN>` |
| Token admin (en uso) | `<CHATWOOT_ADMIN_TOKEN>` |

### Workflow

[n8n-workflows/D2C — Puente YCloud ↔ Chatwoot.json](n8n-workflows/D2C%20—%20Puente%20YCloud%20↔%20Chatwoot.json) — 21 nodos, dos ramas:

```
Entrante:  webhook ycloud-inbound → buscar/crear contacto → buscar/crear conversación → mensaje incoming
Saliente:  webhook chatwoot-outbound → YCloud sendDirectly → si falla → nota privada
```

Webhook de YCloud debe apuntar a `https://example.app.n8n.cloud/webhook/ycloud-inbound`.

### ⚠️ Ediciones requeridas en `D2C Sales Agent — Chatwoot (WhatsApp + IG + Messenger).json`

> Nota: estas tres ediciones **ya están aplicadas** en la versión actual del workflow (ver §4). Se dejan documentadas por si se reimporta desde una versión previa.

Sin estas tres, el agente **descarta silenciosamente** todos los WhatsApp, porque el canal API se identifica como `Channel::Api` y no coincide con el filtro existente:

| Nodo | Cambio |
|---|---|
| `Is Chatwoot inbound text?` | regex → `Whatsapp\|Instagram\|FacebookPage\|Api` |
| `Normalize` → `sender_id` | `channel.includes('Whatsapp')` → `(channel.includes('Whatsapp') \|\| channel.includes('Api'))` |
| `Normalize` → `session_id` y `channel` | mismo cambio en ambas expresiones |

### Limitaciones conocidas del puente

| Limitación | Consecuencia |
|---|---|
| **Ventana de 24h de WhatsApp** | Respuesta de asesor pasadas 24h del último mensaje del cliente → Meta la rechaza y Chatwoot la muestra como enviada. Mitigado con nota privada automática, **no resuelto**. |
| Plantillas aprobadas | Sin soporte. No se puede iniciar conversación en frío. |
| Adjuntos, audio, imágenes | Solo texto. |
| Acuses de entrega y lectura | No se reflejan. |

La ventana de 24h afecta específicamente la **entrega a asesor humano** y el **seguimiento comercial** — los dos momentos que más importan en venta inmobiliaria. El bot nunca la incumple porque responde en segundos.

### Estado de verificación

Cada endpoint fue probado por separado con `curl` (contacto, conversación, mensaje entrante: los tres creados correctamente).

**22/07 — puente validado de punta a punta (rama entrante).** Con un payload simulado de YCloud (`whatsapp.inbound_message.received`) empujado al webhook self-hosted, la ejecución recorrió los 13 nodos sin error y **creó contacto + conversación + mensaje entrante en Chatwoot** (último nodo `Publicar mensaje entrante` devolvió message id). Valida: IF, normalización, credencial del usuario de servicio de Chatwoot, base URL pública y las 5 llamadas a la API. Quedó un contacto de prueba "Cliente Simulado" / `573009998877` en Chatwoot.

La rama de salida (`conversation.meta.sender.phone_number` en `Enviar por YCloud`) sigue sin verificar contra payload real.

### ✅ Rama entrante WhatsApp — RESUELTA de punta a punta (23/07)

Un WhatsApp real (`+57XXXXXXXXXX`, "bot fight mode prueba") recorrió los 13 nodos del puente y **creó contacto + conversación + mensaje en Chatwoot**. El flujo entrante YCloud→puente→Chatwoot funciona.

El bug fue una **cadena de dos problemas**, no uno:

**1. Binding viejo de YCloud tras cambiar de URL.** Al repuntar el webhook de n8n Cloud (`ycloud-whatsapp`) al self-hosted, YCloud dejó de entregar. Se resolvió recreando **todo fresco**: nodo webhook nuevo con **path distinto `ycloud-inbound-2`**, workflow reactivado con `n8n update:workflow --active=true` + reinicio (¡`import:workflow` fuerza `active=false`!), y **endpoint YCloud nuevo creado por API** (`<YCLOUD_ENDPOINT_ID>`) apuntando al path nuevo.

**2. Cloudflare Bot Fight Mode bloqueaba a YCloud.** Una vez YCloud entregaba, Cloudflare respondía **403** a su petición (bot-protection por fingerprint). Síntoma diagnóstico: `curl` → 200, pero `python-urllib` y YCloud → 403. En **plan Free**, la acción **Skip** de una WAF Custom Rule **NO cubre Bot Fight Mode** (solo Pro+ con Super Bot Fight Mode). 

**IP real de entrega de YCloud: `47.236.160.49`** (red Alibaba `47.236.0.0/16`, User-Agent `Go-http-client/2.0` — **no Svix**, por eso las IPs de Svix no aplican). Fix permanente scoped que no toca la web corporativa: **IP Access Rules → Allow** para esa IP/rango (un Allow de IP bypassa Bot Fight Mode), y Bot Fight Mode se puede dejar encendido.

Notas de infra aprendidas:
- `import:workflow` **siempre** deja `active=false`; para activar y registrar el webhook: `n8n update:workflow --id=<id> --active=true` **+ reiniciar n8n**. Verificar en tabla `webhook_entity`.
- n8n corre en contenedor: `127.0.0.1:3000` desde adentro NO alcanza Chatwoot (loopback del contenedor). Se usó la URL pública `https://chatwoot.example.com` para la pata de respuesta.
- Endpoint YCloud nuevo secret: `<YCLOUD_WEBHOOK_SECRET>` (para verificación de firma futura).

---

## n8n self-hosted — instalado 21/07, sin exponer todavía

Corriendo en `/opt/n8n`, **solo en `127.0.0.1:5678`**. Estrategia acordada: **en paralelo con n8n Cloud**, no migración directa. Cloud sigue siendo el que atiende tráfico real hasta que el puente se pruebe end-to-end.

| Componente | Versión | Notas |
|---|---|---|
| n8n | **2.32.0** (fijada) | había `v3-nightly`, descartada |
| Postgres | `postgres:16-alpine` | **contenedor propio**, volumen `n8n_n8n_postgres_data`. No comparte el de Chatwoot |
| Volumen de datos | `n8n_n8n_data` | `/home/node/.n8n` |

Ambos con `restart: always`. El Postgres de n8n **no publica puerto al host** (solo red interna del compose), a diferencia del de Chatwoot que está en `127.0.0.1:5432`.

`/opt/n8n/.env` (permisos `600`): `POSTGRES_PASSWORD` y `N8N_ENCRYPTION_KEY`, generados con `openssl` en la máquina.

Variables relevantes en el compose: `WEBHOOK_URL=https://n8n.example.com/`, `N8N_PROXY_HOPS=1` (va detrás del túnel), `TZ=America/Bogota` — el host está en `Etc/UTC`.

### Consumo medido (2 min idle tras arrancar)

| | |
|---|---|
| n8n | 334 MB |
| Postgres de n8n | 49 MB |
| **Total añadido** | **~380 MB** |
| RAM disponible | 5.3 GB de 7.2 |
| Carga | 0.36 en 2 núcleos |

Chatwoot no se vio afectado (público sigue respondiendo 200). **La duda de CPU sigue abierta**: estas cifras son sin tráfico. El pico de arranque llegó a 2.01 de carga durante las migraciones.

### ✅ Ruta en el túnel — resuelta 21/07

**Accesible en https://n8n.example.com** (200, TLS válido, owner reclamado).

El túnel es de **gestión remota** (`cloudflared --token-file /etc/cloudflared/token`, sin config local), así que la ruta se crea en el dashboard, no en el servidor: Zero Trust → Tunnels → *Public Hostnames* → Add `n8n` / `example.com` / `HTTP` → `localhost:5678`.

⚠️ **Trampa que costó varias vueltas:** hay (había) **dos túneles en la cuenta**. El que sostiene la operación y ya servía Chatwoot es **`<TUNNEL_ID>`** — es el único con conector corriendo en el servidor. El token que se compartió por chat era de **otro** túnel, `<OTRO_TUNNEL_ID>`, sin conector. Agregar la ruta en el túnel equivocado da **530 / error 1033** y el conector del servidor no registra ni una petición. Regla: **el túnel correcto es el que muestra `chatwoot` en su lista de hostnames.** El `<OTRO_TUNNEL_ID>` se borró.

Al reasignar el hostname al túnel bueno, Cloudflare no sobrescribe el registro DNS viejo → error *"A DNS record with this name already exists"*. Hay que **borrar a mano** el `CNAME n8n` en DNS → Records y reintentar; entonces recrea el CNAME correcto.

Config final del conector (verificada en log): `chatwoot → :3000`, `n8n → :5678`, resto 404.

### Owner de n8n — reclamado 21/07

Creado apenas la URL fue pública, para cerrar la ventana en que cualquiera podía reclamarlo (`showSetupOnFirstLoad` pasó a `false`).
- Email `<correo-del-operador>`, contraseña generada por Claude: **rotar al primer login** (ver higiene de credenciales).

### Lo que falta

1. ~~Crear owner~~ / ~~rotar contraseña~~ — hecho (usuario cambió credenciales 21/07).
2. ~~Importar los workflows **desactivados**~~ — hecho 21/07. Importados **Chatwoot-Multichannel** (`tF4UU5eskdvrPmoC`), **Chat-Test** (`JG09DGSlZl30zqX9`) y **Puente YCloud↔Chatwoot** (`rt9rQXfzHW3XXNpw`), los tres `active=false`, en el proyecto personal del owner. Los otros tres (YCloud-Multichannel, YCloud-Chatwoot, WhatsApp-Phase1) no.
3. **Recrear credenciales a mano** — el `N8N_ENCRYPTION_KEY` es nuevo, las de n8n Cloud no se pueden exportar descifradas. Anthropic, Google Sheets OAuth, HubSpot, y el Header Auth de Chatwoot (`api_access_token`).
4. Solo al final, y de uno en uno: reapuntar YCloud, el `webhook_url` del inbox API de Chatwoot y Meta si aplica.
5. Ventaja del self-hosted: la pata de respuesta puede ir a `http://127.0.0.1:3000` en vez de salir a internet y volver por el túnel.

### ⚠️ Propiedades de HubSpot — nombres truncados en Tickets

Verificado contra la API de HubSpot (22/07): las propiedades de Contacto (11) y Negocio (4) existen con tipos y opciones correctos. En **Tickets**, dos propiedades quedaron creadas con el **nombre interno truncado** (les falta la primera letra) y se decidió usarlas tal cual, porque HubSpot no permite renombrar un nombre interno:

| Nodo escribe | Nombre interno real en HubSpot |
|---|---|
| `otivo_escalamiento` | `otivo_escalamiento` (era `motivo_escalamiento`) |
| `ead_channel` | `ead_channel` (era `lead_channel`) |

- El nodo **Escalate to Human Advisor** fue editado para escribir esos nombres truncados. **Save Contact NO se tocó** — su `lead_channel` apunta al objeto Contacto, donde el nombre sí quedó bien.
  - *Estado actual: el escalamiento se movió al sub-workflow `D2C — CRM Registrar Lead`, cuyo nodo `Create Ticket` **ya no escribe ninguna propiedad custom** (solo asunto/descripción/prioridad + asociación al contacto). Los nombres truncados del ticket quedan sin uso.*
- Las opciones de `ead_channel` (ticket) se crearon como `whatsapp/facebook/instagram` y se **corrigieron por API** a `whatsapp/messenger/instagram_dm` para coincidir con lo que escribe el agente y con el `lead_channel` del contacto.
- `lead_company` y `project` son **texto** (no dropdown) a propósito: evita mantener listas de proyectos/empresas a mano y que un valor nuevo del catálogo falle el write.

Método de verificación/edición de HubSpot desde fuera: `n8n export:credentials --id=<id> --decrypted` en el contenedor para obtener el token, usarlo transitoriamente contra `api.hubapi.com`, y borrar el archivo descifrado. El token nunca se imprime.

### ⚠️ Activar un evento de ticket en el HubSpot Trigger desactiva el workflow (21/08)

Al agregar `ticket.propertyChange` sobre `agente` al trigger de `D2C — Sync asignación HubSpot → Chatwoot`, la publicación falló con `Bad request` y **n8n dejó el workflow en `active: false`**. La sincronización de contactos quedó caída unos 20 segundos, hasta revertir el evento y volver a publicar.

Causa: la app pública `<APP_ID>` no tiene el scope `tickets`, así que HubSpot rechaza crear la suscripción. Confirmado aparte: `GET /webhooks/v3/<APP_ID>/subscriptions` con la credencial de developer responde `403 MISSING_SCOPES`.

Reglas que salen de esto:

- Un cambio en el nodo trigger de HubSpot **puede desactivar un workflow en producción**. Hacerlo como paso aparte, nunca junto con otros cambios, y verificar `active` inmediatamente después de publicar.
- El resto de la lógica se puede dejar construida y desconectada del trigger. Se publica sin riesgo y se conecta cuando el scope esté.
- Los dos workflows de sync comparten la misma app pública, que tiene **una sola URL de webhook**. Por eso todos los eventos van en el mismo nodo trigger y no en workflows separados.
- La dirección ticket → contacto **no se quedó esperando el scope**: corre por polling cada 15 min leyendo el historial de la propiedad `agente` (`propertiesWithHistory`), que dice cuándo cambió y quién la cambió. El Service Key sí puede leer y escribir tickets, el scope solo hace falta para el webhook. Con el scope, la latencia baja de 15 min a unos 2 s.

### ⚠️ Trampas del import por CLI (21/07)

- El JSON exportado **no trae campo `id`** → `import:workflow` falla con `null value in column "id" of relation "workflow_entity"`. Hay que inyectar un `id` (16 chars alfanuméricos) y `active:false` antes de importar.
- `docker compose cp carpeta n8n:/tmp/dest` **anida** si `/tmp/dest` ya existe (`/tmp/dest/carpeta/...`) → se importaban los archivos viejos. Copiar **por ruta de archivo explícita** (`cp x.json n8n:/tmp/import/x.json`), no la carpeta.

### Capacidad medida (21/07, con Chatwoot corriendo e idle)

| | |
|---|---|
| RAM | 1.2 GB en uso de 7.2 — **5.7 GB disponibles** |
| Carga | 0.04 en 2 núcleos — prácticamente ocioso |
| Disco | 12 GB de 98 — **82 GB libres** |
| Chatwoot completo | ~1.05 GB RAM (sidekiq 527 MB, rails 437 MB, postgres 88 MB, redis 4 MB) |

**RAM y disco sobran.** La duda real es CPU: esas cifras son sin tráfico. Bajo carga, Sidekiq + el agente de IA + los workflows de n8n compiten por 2 núcleos de un AMD A4 de 2017.

### Consideraciones antes de migrar desde n8n Cloud

- **Hostname propio en el mismo túnel.** Cloudflare Tunnel admite varias rutas: agregar `n8n.example.com → localhost:5678` en *Public Hostnames*. No hace falta otro túnel ni tocar el firewall.
- **Cambian todas las URLs de webhook.** Las de YCloud, las de Chatwoot (`webhook_url` del inbox API) y las de Meta si aplica. Hay que actualizarlas en los tres lados.
- **`WEBHOOK_URL` de n8n** debe coincidir con el hostname público, o las URLs que muestra el editor salen mal.
- **Postgres propio para n8n.** No compartir la instancia de Chatwoot; levantar un contenedor aparte o usar SQLite si el volumen es bajo.
- **Se hereda el riesgo del portátil.** Hoy n8n Cloud es lo único del sistema que no depende de que esa laptop siga encendida. Migrarlo concentra todo en un solo punto de fallo sin backups.

---

## Sincronización de asignaciones Chatwoot ↔ HubSpot (13/08)

Las cuatro asesoras comparten un solo usuario de HubSpot, así que el dueño del registro (`hubspot_owner_id`) no sirve para saber quién atiende a cada lead. Se usa la propiedad de contacto **`agente`** (desplegable con los cuatro nombres) y dos workflows la mantienen en sincronía con el asignado de la conversación en Chatwoot, en ambos sentidos.

### Piezas

| | |
|---|---|
| Propiedad HubSpot | `agente`, tipo `enumeration`/`select`, grupo `captura_de_leads_(bot)`. Valores exactos: `Ana Gomez`, `Beatriz Ruiz`, `Carolina Diaz`, `Diana Mejia` |
| Workflow Chatwoot → HubSpot | `pTvpKQtFgq6PN64K`, activo |
| Workflow HubSpot → Chatwoot | `NtcfpSYzUWyNITwt`, activo |
| Webhook de cuenta en Chatwoot | id `2`, solo evento `conversation_updated`, apunta a `https://n8n.example.com/webhook/chatwoot-asignacion` |
| App pública de HubSpot | id `<APP_ID>`, en una cuenta de desarrollador aparte, instalada en el portal `<PORTAL_ID>`. Solo sostiene el webhook |
| Credencial n8n del trigger | `HubSpot Developer account` (`hubspotDeveloperApi`, id `r27St1NAT3o0PPM7`) |

### Las dos entradas de HubSpot → Chatwoot

La vía principal es el nodo **HubSpot Trigger** suscrito a `contact.propertyChange` sobre `agente`. Medido el 13/08: **2 segundos** desde que se cambia el campo hasta que la conversación queda asignada.

Detrás queda un **respaldo cada 15 minutos** que busca contactos con `agente` modificados en los últimos 20. Existe porque el servidor es un portátil que se duerme: los eventos que HubSpot no logre entregar en esa ventana se recuperan igual. Si el trigger se cae del todo, la sincronía sigue funcionando con 15 minutos de retraso en vez de romperse.

Las dos entradas comparten toda la cadena de abajo. El evento del trigger solo trae el id del contacto, así que se leen con `crm/v3/objects/contacts/batch/read`, que devuelve la misma forma (`results[].properties`) que la búsqueda del respaldo.

⚠️ **El nodo HubSpot Trigger de n8n renombra `objectId` a `contactId`** en la salida. Filtrar por `objectId` hace que el evento se descarte en silencio: la ejecución aparece exitosa y dura 50 ms. El Code acepta los dos nombres.

⚠️ **La credencial del trigger no es la Service Key.** Es `hubspotDeveloperApi`, que exige cuenta de desarrollador de HubSpot, app pública dentro de ella (App ID, Developer API Key, Client Secret) y **esa app instalada en el portal del cliente**. Sin la instalación la suscripción se crea pero no llega ningún evento. Las Service Keys no soportan webhooks, y la acción "send webhook" de los Workflows de HubSpot exige Operations Hub Professional, que el portal no tiene.

**Mapa de agentes** (id de usuario de Chatwoot ↔ valor de `agente`): `3` Ana Gomez, `4` Beatriz Ruiz, `6` Carolina Diaz, `7` Diana Mejia. Está escrito en un nodo Code de cada workflow. Si entra o sale una asesora hay que tocar los dos, más las opciones del desplegable en HubSpot.

### La llave de unión es `conversation_transcript_url`

No es el teléfono. En los 152 contactos que creó el agente, `phone` viene nulo, con `\n` al final o duplicado en `whatsapp_id` con y sin `+`, y en Instagram no hay teléfono del todo. En cambio `conversation_transcript_url` está siempre y contiene el id de la conversación: `https://chatwoot.example.com/app/accounts/1/conversations/50`.

- **Chatwoot → HubSpot** reconstruye esa URL con `account_id` + `conversation_id` del webhook y busca por `EQ` exacto. Como respaldo agrega filtros `OR` por `whatsapp_id` y `phone` cuando el contacto tiene teléfono.
- **HubSpot → Chatwoot** parte la URL y saca el id de la conversación.

Consecuencia: la sincronía solo cubre contactos que el agente haya registrado en HubSpot. Un chat de Chatwoot sin contacto en el CRM no tiene a dónde escribir, y sale en cero sin error.

### Anti-bucle

Cada lado lee el estado del otro antes de escribir y se detiene si ya coinciden:

```
Asesora asigna en Chatwoot  → webhook → ¿HubSpot ya tiene ese agente? → sí: para
Asesora pone agente en HubSpot → poll → ¿Chatwoot ya tiene ese asignado? → sí: para
```

Sin eso, cada escritura dispararía la del otro lado indefinidamente. Verificado en las dos direcciones: la segunda corrida sobre el mismo dato devuelve cero items.

### Dos trampas que costaron el primer intento

**1. `hs_lastmodifieddate` viene `null` en contactos.** El poll filtra por fecha de modificación y con esa propiedad la búsqueda siempre devolvía `total: 0`. En contactos la buena es **`lastmodifieddate`**. La otra existe pero solo se llena en deals, tickets y empresas.

**2. `conversation_updated` se dispara por cualquier cambio.** Prioridad, etiquetas, `updated_at`, no solo por asignación. En una cuenta con 1.029 conversaciones eso son miles de ejecuciones inútiles al día. El payload real trae `changed_attributes`, así que el nodo Webhook usa la opción **`onlyRunIf`**: los eventos que no traen `assignee_id` reciben 200 y **no crean ejecución**. Probado: reasignar creó una ejecución, dos cambios de prioridad crearon cero.

### Ventana del respaldo

Corre cada 15 minutos y mira los contactos modificados en los últimos 20. El solape es a propósito: la API de búsqueda de HubSpot tarda unos segundos en indexar (un contacto recién creado no aparece en la búsqueda inmediata), y como la operación es idempotente, repetir no cuesta nada. Evita tener que guardar el timestamp de la última corrida.

### Qué queda fuera a propósito

- **Asignados que no son las cuatro asesoras** (Daniel, Admin Aurora, Integracion n8n) se ignoran. No se escribe nada en HubSpot.
- **Desasignar en Chatwoot no limpia `agente`** en HubSpot. Si se quiere, hay que decidir qué significa un contacto sin asesora y agregar la rama.
- **Conversaciones resueltas** se asignan igual si alguien cambia `agente` en HubSpot.

---

## Pendientes

### 1. Suspensión del portátil — 🔴 crítico

Sin resolver. Si alguien cierra la tapa o el equipo entra en suspensión, **dejan de entrar mensajes de clientes**. Requiere desactivar `sleep.target` y la acción de cierre de tapa.

### 2. Backups — 🔴 crítico

**No hay ninguno.** Toda la data (conversaciones, contactos) vive en un volumen Docker de una laptop. Si ese disco falla, se pierde todo. Mínimo: `pg_dump` diario por cron, idealmente enviado fuera de la máquina.

### 3. SMTP — 🟡

Sin configurar. Implica que **no funcionan**: recuperación de contraseña, invitaciones a agentes, confirmación de cambio de email. La contraseña del super admin no se puede resetear por correo — guardarla bien.

### 4. Probar el puente de WhatsApp de punta a punta — 🟡 en progreso (23/07)

Entrante ✅ (WhatsApp→Chatwoot probado). Loop del agente cableado y en prueba. Fixes aplicados en esta sesión:

- **Las 3 ediciones de canal `Api`** en Chatwoot-Multichannel — aplicadas (filtro regex `+Api`, y `sender_id`/`session_id`/`channel` del Normalize tratan `Api` como whatsapp).
- **Webhooks de Chatwoot configurados por API:** webhook de cuenta (`message_created`) → `…/webhook/chatwoot-inbound` (dispara agente); `webhook_url` del inbox API → `…/webhook/chatwoot-outbound` (respuesta→YCloud). **Estaba mal:** el `webhook_url` apuntaba a `chatwoot-inbound` y no había webhook de cuenta.
- **Email sintético de HubSpot:** el fallback era `<sender_id>@<channel>.lead` → **HubSpot rechaza todo TLD `.lead`/`.invalid`** con `INVALID_EMAIL` (probado: solo acepta TLDs reales como `example.com`). Cambiado a `<channel>-<sender_id>@example.com`. Esto tumbaba **Save Contact**, y en cascada **Create Deal**/**Escalate** ("No data found", por el `vid` faltante). *(Estado actual: el fallback vive en el sub-workflow `D2C — CRM Registrar Lead` y usa `sin-correo-<channel>-<sender_id>@leads.example.com`.)*
- **Activación en n8n 2.32:** `import:workflow` deja `active=false` SIEMPRE, y `update:workflow --active=true` está deprecado → usar **`publish:workflow --id=<id>` + reiniciar n8n**. Verificar en `webhook_entity` (registro tiene ~5-10s de lag tras el arranque).
- Los 3 webhooks del loop viven en `webhook_entity`: `chatwoot-inbound` (MC), `chatwoot-outbound` + `ycloud-inbound-2` (puente).

Nota Anthropic: durante la prueba, algunas corridas fallaron con *"credit balance is too low"* — vigilar saldo de la API.

### 5. Cambiar al token de agent-bot — 🟡

El puente usa hoy el token del **super admin** (`<CHATWOOT_ADMIN_TOKEN>`). Ya existe un agent bot dedicado (`<CHATWOOT_AGENT_BOT_TOKEN>`); cambiarlo cuando el flujo esté estable, para que el acceso no cuelgue de una cuenta personal.

### 6. Verificación de negocio de Meta — 🔴 camino crítico

**Lo único que bloquea producción real, y no depende de nosotros.**

Confirmado que el cliente **sí tiene**: página de Facebook, Instagram Business y Meta Business Manager.
**No tiene**: verificación de negocio (cámara de comercio + RUT).

Sin ella:

| Canal | Estado |
|---|---|
| Instagram | permisos en Standard Access → solo cuentas con rol en la app (admin/dev/tester) |
| WhatsApp vía Meta Cloud API | inviable — motivo por el que se fue a YCloud |
| WhatsApp vía YCloud | **no afectado**, YCloud es BSP con su propia habilitación |
| Messenger | requiere App Review para uso público |

El puente de YCloud esquiva el bloqueo para WhatsApp. Instagram y Messenger siguen frenados hasta la revisión de Meta, que puede tomar semanas. **Iniciar el trámite es la tarea de mayor camino crítico del proyecto.**

---

## Higiene de credenciales

| Ítem | Estado |
|---|---|
| Contraseña de `<usuario>` | Compartida por chat. **Rotar** — ya hay llave SSH, no hace falta para entrar. |
| Token de túnel Cloudflare | Compartido por chat. Refrescable desde el dashboard (`•••` → refresh) sin afectar DNS ni rutas. |
| App Secret de Meta | Compartido por chat. Rotable en Settings → Basic → *Reset*; hay que recargarlo en `.env` y sincronizarlo a la base. |
| Token de super admin de Chatwoot | En uso por el puente de n8n. Sustituir por el de agent-bot (pendiente 5). |
| Llave SSH `soptix-chatwoot-deploy` | Instalada en `~/.ssh/authorized_keys` de `<usuario>`. Retirar al terminar el contrato. |
| Llave SSH `soptix-n8n-deploy` | Añadida el 21/07 al `authorized_keys` de `<usuario>` (la anterior no estaba en el equipo de trabajo). Retirar igual al terminar. |
| Contraseña de `<usuario>` — 2ª vez | **Vuelta a compartir por chat el 21/07** y usada para `sudo`. Sigue sin rotarse. `sudo` la pide (la llave solo cubre el login), así que rotarla exige tener a mano la nueva. |
| `/opt/n8n/.env` | `N8N_ENCRYPTION_KEY` — **sin esto las credenciales guardadas en n8n son irrecuperables.** No hay backup. |
| Owner de n8n | Credenciales cambiadas por el usuario el 21/07 (la contraseña inicial generada por Claude ya no es válida). **Sin SMTP no hay recuperación por correo** — si se pierde, resetear por CLI en el servidor. |
| Super admin de Chatwoot | Cuenta creada con `<correo-admin-del-cliente>`. **Sin SMTP no hay recuperación de contraseña** — si se pierde, hay que resetearla por consola de Rails. |
| Panel XAMPP del cliente | Expuesto públicamente en el 443. Fuera de alcance, pero avisado. |
| Developer API Key de HubSpot | Nueva el 13/08, para el trigger de asignaciones. Vive en la **cuenta de desarrollador**, no en el portal del cliente. Da control sobre la app `<APP_ID>` y sus webhooks. Rotable desde la cuenta de desarrollador. |
| Client Secret de la app `<APP_ID>` | Guardado en la credencial `HubSpot Developer account` de n8n. Rotarlo obliga a recargarlo ahí. |
| Roles en Chatwoot | Las cuatro asesoras son `administrator`, no `agent`. Con eso pueden ver y rotar los tokens de API que sostienen el puente y los dos syncs. **Bajarlas a `agent`.** |

## Comandos útiles

```bash
# estado
cd /opt/chatwoot && sudo docker compose ps
sudo systemctl status cloudflared

# logs
sudo docker compose logs -f rails
sudo journalctl -u cloudflared -f

# reiniciar tras cambiar .env
sudo docker compose up -d --force-recreate rails sidekiq

# consola de rails (ej. resetear contraseña)
sudo docker compose exec rails bundle exec rails c
```

## Decisiones de fondo

**Se instaló en el equipo del cliente, no en un VPS propio.** La alternativa evaluada fue un VPS de Soptix (~€5-24/mes), que habría eliminado la dependencia de IT y los problemas de suspensión, uptime e IP. Se descartó por decisión de negocio. Si el equipo resulta ser la laptop de trabajo de alguien, conviene retomar esa conversación.

**Versión fijada en v4.16.0-ce** en vez de `latest`, para que una actualización de imagen no dispare migraciones inesperadas en un reinicio.
