# HubSpot CRM — Components & Setup Guide

**Companion to:** `customer-service-ai-agent-architecture.md`
**Scope:** What to build in HubSpot so the n8n agent can capture, segment, route and escalate leads for **Constructora Aurora** (Altos del Rio) and **Constructora Bahia** (Villa Serena), per Sections 4.5 and 7 of the architecture.
**Tier assumption:** HubSpot free CRM to start, per the architecture's cost guidance. A few components below note where a paid tier (Starter/Professional) becomes necessary — flagged so you can decide, not assumed.

---

## 1. Component overview

| Component | Purpose | Free tier? |
|---|---|---|
| Custom contact properties | Store project, company, channel, financing signals, consent | Yes |
| Custom deal properties | Mirror project/company on the deal for pipeline reporting | Yes |
| One deal pipeline, shared | Track lead → advisor → close, segmented by `company` + `project` properties (not one pipeline per project/company) | Yes |
| Lead Status values | Track where a lead is without needing extra pipelines | Yes |
| Active lists (per project) | Feed advisor routing and reporting | Yes (list limits apply) |
| Owners + Teams | Assign leads to the right advisor per project | Teams require Starter+; owners work on free |
| Workflows | Auto-assign by project, alert on hot leads | Requires Starter+ (free tier workflow support is very limited) |
| Private App (API) | Lets n8n create/update contacts & deals | Yes |
| Conversations inbox | Human handoff surface | Yes, but native WhatsApp/IG inbox channel needs Starter+ or an app-marketplace connector |

The two paid-tier flags (Teams, Workflows) are the main place the "free to start" plan will hit a wall. Section 8 below gives a free-tier workaround for each.

---

## 2. Pipeline structure

**Decision: one shared pipeline, with `company` and `project` as required properties on every deal.** This matches Section 7's explicit rule against a pipeline per project ("would multiply and go stale"), works on the free tier (multi-pipeline is typically gated to paid tiers), and lets you filter/report by company or project with saved views instead of maintaining parallel pipelines. If hard separation is ever needed (e.g., an advisor must never see the other company's deals), that's a **Teams/permissions** problem, not a pipeline problem — solve it with Section 8.2 instead of splitting pipelines.

---

## 3. Contact properties to create

Go to **Settings → Properties → Contact properties → Create property** for each. Group them under a new property group, e.g. `Lead Capture (Bot)`.

| Property name (internal) | Label | Type | Values / notes |
|---|---|---|---|
| `lead_company` | Empresa | Dropdown | `aurora`, `bahia`. **Internal name is `lead_company`, NOT `company`** — HubSpot's native "Company Name" property already owns the internal name `company` and can't be deleted or reused |
| `project` | Proyecto de interés | **Single-line text** | Holds the catalog's `project_id` (e.g. `altos-del-rio-vi`, `villa-serena-3`). **Deliberately text, not a dropdown:** n8n writes this straight from the catalog, so a dropdown would (a) need a second manual edit per new project and (b) hard-fail the API write with `400 – not one of the allowed options` whenever the catalog has a project the dropdown doesn't. Text keeps "add a project = edit a row" true (architecture §4) |
| `lead_channel` | Canal | Dropdown | `whatsapp`, `messenger`, `instagram_dm` |
| `whatsapp_id` | WhatsApp ID | Single-line text | Raw sender ID from the BSP, in case it differs from `phone` |
| `unit_type_interest` | Tipo de vivienda de interés | Single-line text or dropdown | Free text is safer while unit names vary by project (e.g. "Medianera 3 alcobas") |
| `bedrooms_wanted` | Alcobas deseadas | Number | |
| `financing_bank_preference` | Banco de interés | Dropdown | `BBVA`, `AV Villas`, `Bancolombia`, `FNA`, `otro`, `no_sabe` |
| `needs_credit` | Necesita crédito | Dropdown (Yes/No/Unknown) | |
| `conversation_transcript_url` | Enlace a transcripción | Single-line text (or URL) | Link to the stored chat log |
| `price_confirmation_pending` | Precio pendiente de confirmar | Yes/No | Set `true` whenever the agent hit a missing catalog value (Section 4.2 gap case) — this is your signal that a human must follow up with a number |
| `data_consent_given` | Autorización tratamiento de datos | Yes/No | Habeas Data (Ley 1581) consent capture — Section 10 |
| `data_consent_date` | Fecha de autorización | Date picker | |
| `data_controller_company` | Empresa responsable del dato | Dropdown | Only needed if the two companies end up as separate data controllers per the open legal question in Section 11.2 — leave blank until that's resolved |

Reuse native properties instead of duplicating them: `firstname`, `lastname`, `phone`, `hs_lead_status` (see Section 5), and `hubspot_owner_id` (see Section 8.1).

---

## 4. Deal properties to create

Deals give you pipeline/stage tracking and revenue reporting that contacts alone don't. Create a deal every time a lead reaches "qualified" (not on first contact — see Section 6 workflow), so the pipeline stays a signal of real intent rather than every inbound message.

| Property name | Label | Type | Notes |
|---|---|---|---|
| `lead_company` | Empresa | **Single-line text** | Values `aurora` / `bahia` (internal name `lead_company`). Text on the deal — field type is irrelevant to n8n as long as the value strings match the contact property |
| `project` | Proyecto | **Single-line text** | Same rationale as the contact property — text, not dropdown, so catalog-driven project_ids never fail the write |
| `unit_type_interest` | Tipo de vivienda | Single-line text | Copied from the contact at deal creation |
| `amount` | Monto | Currency (native) | Populate from the catalog's `price_cop` when known; leave blank if `price_confirmation_pending` |
| `financing_bank_preference` | Banco de interés | Dropdown | Same values as contact |

Deal name convention: `{project} — {firstname lastname}` (e.g. `altos-del-rio-vi — Juan Pérez`) so advisors can scan the pipeline without opening every record.

---

## 5. Lead status & pipeline stages

**`hs_lead_status`** (native property) — customize its dropdown options to match the agent's decision points from Section 3.2/6:

`Nuevo` → `Contactado por bot` → `Calificado` → `Precio pendiente – Escalado` → `Asignado a asesor` → `Cerrado ganado` → `Cerrado perdido`

**Deal pipeline stages** (single shared pipeline, per Section 2):

1. `Nuevo lead` (deal created on qualification)
2. `Contactado por asesor`
3. `Cita / visita agendada`
4. `Propuesta enviada`
5. `Negociación`
6. `Cerrado ganado`
7. `Cerrado perdido`

Set stage 6 as the "Closed Won" and stage 7 as "Closed Lost" probability markers in the pipeline editor.

---

## 6. Lists — segmentation for routing & reporting

Create **Active lists** (Contacts → Lists) filtered on the `project` property, one per active project:

- `Leads — Altos del Rio VI` (filter: `project = altos-del-rio-vi`)
- `Leads — Villa Serena 3ra Etapa` (filter: `project = villa-serena-3`)
- `Leads — Precio pendiente` (filter: `price_confirmation_pending = true`) — this is your daily "advisor must call back with a number" queue

Add a project value → add a list. This is the property-driven segmentation Section 7 calls for instead of a pipeline-per-project.

---

## 7. API access for n8n — Service Key (preferred) or Private App

n8n needs a token to create/update contacts and deals. Do **not** use a personal API key (deprecated).

**Preferred: a Service Key.** As of 2026 HubSpot marks UI Private Apps as *legacy* and points API-only, system-to-system integrations (exactly n8n's use) to **Service Keys** (public beta). They take the **same scopes**, are used as a **Bearer token** (drop-in for n8n's App Token field), and aren't tied to an individual employee. Their one limitation — **no webhook support** — does not affect this build: n8n receives inbound messages from the BSP/Meta and only *writes* outbound to HubSpot, so no HubSpot→n8n webhook is needed.

1. **Development (Desarrollo) → Keys → Service keys → Create service key.**
2. Name it `n8n-agent-integration`.
3. Add scopes (minimum needed):
   - `crm.objects.contacts.read` / `.write`
   - `crm.objects.deals.read` / `.write`
   - `tickets` (read + write) — **required for the escalation tool**, which creates a HubSpot ticket (Section 9). Missing this yields `400 – Must have scope [tickets-write]` at runtime even though contacts/deals work.
   - `crm.schemas.contacts.read`
   - `crm.schemas.deals.read`
   - `crm.objects.owners.read` (needed for the assignment logic in Section 8.1)
4. Create, then **Show → Copy** the key into n8n's HubSpot credential (**App Token** auth — the key is sent as `Authorization: Bearer <key>`, identical to a private-app token). Store it in n8n's credential vault, not in a workflow node.
5. Test with a single "create contact" call before wiring the full agent flow.

**Fallback: legacy Private App.** If the *Development → Service keys* area isn't in your portal yet (beta rolls out in phases), a Private App still works and requires no action for the foreseeable future: **Settings → Integrations → Private Apps → Create a private app**, same scopes, copy the access token into the same n8n App Token field. Migrating later is just swapping the token in n8n — no other change.

---

## 8. Assignment & automation — free-tier workaround

The architecture calls for routing each project's leads to the advisors who handle it (Section 4.5, 7). Native HubSpot **Workflows** (if-this-then-that automation) are the natural tool but are gated behind Starter/Professional on most current plans. Two paths:

### 8.1 If staying on free tier
Do the routing logic **in n8n, not HubSpot**: maintain a small lookup table in n8n (or a tab in the same Sheets/Airtable catalog) mapping `project → advisor HubSpot owner ID`. When the agent's lead-capture tool creates/updates the contact and deal, it also sets `hubspot_owner_id` directly via the API call — this reproduces "assignment rules" without needing HubSpot Workflows. This is a small addition to the same tool that already writes the lead (Section 3.2, step 4), not new infrastructure.

For the hot-lead alert (Section 6: "define escalation triggers... routing to a shared human inbox"), have n8n send the Slack/WhatsApp/email notification to the assigned advisor directly when the agent's handoff tool fires, instead of relying on a HubSpot workflow trigger.

### 8.2 If upgrading to Sales Hub Starter (~low monthly cost)
Build native HubSpot workflows instead:
- **Workflow: Auto-assign by project** — trigger on contact property `project` is known → set `hubspot_owner_id` based on a branching action per project value.
- **Workflow: Hot lead alert** — trigger on `hs_lead_status = Precio pendiente – Escalado` OR a deal entering `Nuevo lead` → internal notification/email to the assigned owner.
- **Teams**: create a HubSpot Team per company (`Equipo Aurora`, `Equipo Bahia`) and optionally per-project sub-teams, then restrict record visibility so an advisor only sees their company/project's leads — this is what actually delivers the "per-company separation" Section 10 gestures at, cleanly, without splitting pipelines.

Given the project is starting on the free tier, **default to 8.1** and revisit 8.2 once volume justifies the upgrade (same logic the architecture uses for the CRM tier decision itself, Section 8 of the architecture doc).

---

## 9. Human handoff — Conversations inbox

Section 6 of the architecture names "a shared human inbox (HubSpot or Chatwoot)" for escalations. On HubSpot:

- The free **Conversations inbox** works for email/live chat out of the box.
- Native **WhatsApp/Instagram DM channel integration** inside HubSpot Conversations generally requires Sales/Service Hub Starter or higher, or a marketplace connector tied to your BSP (360dialog/Twilio both list HubSpot connectors).
- Since the agent brain already lives in n8n and channel messaging is already routed through Chatwoot (see `n8n-workflows/README.md` and `chatwoot-deployment.md`), you don't strictly need HubSpot's native channel connection — n8n's escalation path creates a **HubSpot ticket** on the contact and you notify the advisor separately (Slack/WhatsApp/email) with a link to the Chatwoot conversation. This avoids paying for the native inbox channel just for escalation visibility.

If the team wants advisors replying *inside* HubSpot rather than in Chatwoot/WhatsApp directly, that's when the native channel connector becomes worth the upgrade — flag it as a decision for the client, not a default.

### 9.1 Escalation ticket — how it's implemented

Escalation is no longer a standalone tool. The agent calls the **`D2C — CRM Registrar Lead` sub-workflow** with `escalate = true` (and `ticket_name` / `ticket_description`), and that sub-workflow's `Create Ticket (HubSpot)` node creates a `HIGH`-priority ticket **associated to the contact**. Pick its `Pipeline` and `Stage` in the node after import (currently `pipelineId: 0`, `stageId: 1`).

> **The live ticket node writes only native fields** — `subject` (from `ticket_name`), `description` (from `ticket_description`), `priority: HIGH`, and the contact association. It does **not** write the custom ticket properties below. When escalation moved into the CRM sub-workflow, those custom properties were dropped to keep the write simple (and to sidestep the truncated-internal-name issue recorded in `chatwoot-deployment.md`). The motive/channel/phone are already captured on the **contact** record and folded into `ticket_description` in prose.

If you later want structured ticket fields (e.g. to build ticket views/reports), create these on the **Ticket** object (Settings → Objects → Tickets → Manage ticket properties) and add them to the `Create Ticket` node — they are **optional / not currently populated**:

| Property name | Label | Type | Options / Notes |
|---|---|---|---|
| `motivo_escalamiento` | Motivo de escalamiento | Dropdown | `listo_para_comprar`, `pregunta_credito`, `precio_faltante`, `queja`, `pidio_humano`, `otro` |
| `project` | Proyecto | **Single-line text** | Same rationale as the contact/deal property — holds the catalog `project_id`, text so catalog-driven values never fail the write |
| `lead_channel` | Canal | Dropdown | `whatsapp`, `messenger`, `instagram_dm` — same values as the contact property |
| `telefono_contacto` | Teléfono de contacto | Single-line text | Phone captured in-chat (WhatsApp number, or the one the customer shares on Instagram/Messenger) |

### 9.2 Associations are automatic

The sub-workflow links records by id in the same run, so nothing is left dangling:

- **Contact ↔ Deal** — the deal is created with `associatedVids = [contact vid]`.
- **Deal ↔ Ticket** — if a run both creates a deal and a ticket, a HubSpot v4 API call associates them so the ticket shows on the deal.

The earlier "contact/deal linking is manual, do it in Phase 2" caveat no longer applies.

### 9.3 `agente` on tickets

The **Ticket** object has its own `agente` property: dropdown, internal name `agente`, same four options as the contact one (Ana Gomez, Beatriz Ruiz, Carolina Diaz, Diana Mejia). Verified against the API on 21/08, no truncation this time.

It is kept in step with the contact's `agente` by the assignment sync, not by the ticket creation node. `Create Ticket` still writes only native fields, and a new escalation ticket is born with `agente` empty because no advisor has taken the chat yet. The first time the contact's `agente` changes, the sync stamps every open ticket of that contact. The reverse also holds: an advisor who sets `agente` on a ticket has it pushed to the contact, which then assigns the Chatwoot conversation. See `n8n-workflows/README.md` §6.1.

---

## 10. Setup checklist (execution order)

1. [ ] Create HubSpot account (or confirm existing one) — free tier.
2. [ ] Create Private App, grant scopes, save token in n8n (Section 7).
3. [ ] Create contact property group `Lead Capture (Bot)` and all properties in Section 3.
4. [ ] Create deal properties in Section 4.
5. [ ] Customize `hs_lead_status` dropdown values (Section 5).
6. [ ] Build the shared pipeline with the 7 stages (Section 5).
7. [ ] Create the per-project active lists + the "Precio pendiente" list (Section 6).
8. [ ] Wire n8n's lead-capture tool to write contact + deal fields via the Private App token — test with one manual conversation end-to-end.
9. [ ] Implement owner assignment in n8n per Section 8.1 (project → owner lookup) and confirm a test lead lands with the right `hubspot_owner_id`.
10. [ ] Confirm the escalation path: the `D2C — CRM Registrar Lead` sub-workflow creates the ticket (associated to the contact) when the agent passes `escalate = true`, and pick the ticket `Pipeline`/`Stage` in its `Create Ticket` node. The 4 custom ticket properties (Section 9.1) are **optional** — the live node writes only native fields. Notify the advisor out-of-band (Slack/WhatsApp/email) with a link to the Chatwoot conversation.
11. [ ] ~~Add `project` values to the dropdown~~ — no longer needed: `project` is a free-text field holding the catalog `project_id`, so adding a project needs **no** HubSpot change (this is the point of making it text). Only remember to add a new **active list** (Section 6) if you want per-project segmentation for the new project.
12. [ ] Revisit Section 8.2 (Teams + native Workflows) once conversation volume justifies a paid tier.

---

## 12. Guía paso a paso (interfaz de HubSpot en español)

Versión operativa de los pasos 2–7 del checklist de la Sección 10, con las rutas de clics para seguir directamente en la interfaz en español de HubSpot. Los nombres de menú en español pueden variar levemente según la versión/región de HubSpot — se incluye el término en inglés entre paréntesis por si el menú no coincide exactamente.

### Paso 1: Propiedades de contacto

1. Clic en el ícono de **engranaje** (arriba a la derecha) → **Configuración** (Settings).
2. En el menú izquierdo: **Objetos** (Objects) → **Contactos** (Contacts).
3. Pestaña **Propiedades de contactos** (Contact properties).
4. Clic en **Crear propiedad** (Create property), arriba a la derecha.
5. En la primera pantalla, **Objeto = Contacto**, y en **Grupo** clic en **+ Añadir grupo** (+ Add group) para crear `Lead Capture (Bot)` — así todo queda agrupado en vez de disperso entre los grupos por defecto de HubSpot.
6. Crea cada una de estas propiedades (Crear propiedad → completar campos → Siguiente → si es lista desplegable, agregar las opciones → Crear):

| Etiqueta (label) | Nombre interno | Tipo de campo | Opciones |
|---|---|---|---|
| Empresa | `lead_company` | Lista desplegable de selección única | `aurora`, `bahia` — usar nombre interno `lead_company`, NO `company` (ese ya existe de forma nativa) |
| Proyecto de interés | `project` | **Texto de una línea** | guarda el `project_id` del catálogo; texto (no lista) para que n8n escriba cualquier proyecto nuevo sin fallar |
| Canal | `lead_channel` | Lista desplegable de selección única | `whatsapp`, `messenger`, `instagram_dm` |
| WhatsApp ID | `whatsapp_id` | Texto de una línea | — |
| Tipo de vivienda de interés | `unit_type_interest` | Texto de una línea | — |
| Alcobas deseadas | `bedrooms_wanted` | Número | — |
| Banco de interés | `financing_bank_preference` | Lista desplegable de selección única | `BBVA`, `AV Villas`, `Bancolombia`, `FNA`, `otro`, `no_sabe` |
| Necesita crédito | `needs_credit` | Lista desplegable de selección única | `si`, `no`, `no_sabe` |
| Enlace a transcripción | `conversation_transcript_url` | Texto de una línea | — |
| Precio pendiente de confirmar | `price_confirmation_pending` | Casilla única (Sí/No) | — |
| Autorización tratamiento de datos | `data_consent_given` | Casilla única (Sí/No) | — |
| Fecha de autorización | `data_consent_date` | Selector de fecha | — |

Tip: al escribir la Etiqueta, HubSpot genera el nombre interno automáticamente — haz clic en "Editar" junto al nombre interno para forzarlo a que coincida exactamente con los nombres de la tabla (n8n va a referenciarlos directamente, así que deben quedar igual).

### Paso 2: Propiedades de negocio

Mismo procedimiento, pero en **Objetos** (Objects) → **Negocios** (Deals) → **Propiedades de negocios** (Deal properties) → **Crear propiedad**. Solo necesitas estas — `company`, `project` y `financing_bank_preference` pueden reutilizar las mismas opciones de la tabla anterior:

| Etiqueta | Nombre interno | Tipo de campo | Opciones |
|---|---|---|---|
| Empresa | `lead_company` | **Texto de una línea** | valores `aurora` / `bahia` (nombre interno `lead_company`) |
| Proyecto | `project` | **Texto de una línea** | igual que en contactos: texto, no lista |
| Tipo de vivienda | `unit_type_interest` | Texto de una línea | — |
| Banco de interés | `financing_bank_preference` | Lista desplegable de selección única | igual que arriba |

(La propiedad **Monto** / Amount ya existe por defecto — no hace falta crearla.)

### Paso 2b: Propiedades de ticket (opcional)

**Opcional — el workflow actual no las escribe.** El escalamiento vive ahora dentro del sub-workflow `D2C — CRM Registrar Lead`, cuyo nodo `Create Ticket (HubSpot)` crea el ticket con solo asunto, descripción, prioridad y la asociación al contacto (ver Sección 9.1). Crea estas propiedades solo si más adelante quieres campos estructurados en el ticket (para vistas/reportes); si las creas, agrégalas al nodo `Create Ticket`. Mismo procedimiento, en **Objetos** (Objects) → **Tickets** → **Propiedades de tickets** (Ticket properties) → **Crear propiedad**.

| Etiqueta | Nombre interno | Tipo de campo | Opciones |
|---|---|---|---|
| Motivo de escalamiento | `motivo_escalamiento` | Lista desplegable de selección única | `listo_para_comprar`, `pregunta_credito`, `precio_faltante`, `queja`, `pidio_humano`, `otro` |
| Proyecto | `project` | **Texto de una línea** | guarda el `project_id` del catálogo; texto, no lista |
| Canal | `lead_channel` | Lista desplegable de selección única | `whatsapp`, `messenger`, `instagram_dm` (mismos valores que en contactos) |
| Teléfono de contacto | `telefono_contacto` | Texto de una línea | teléfono capturado en el chat (número de WhatsApp, o el que comparta en Instagram/Messenger) |

Recuerda: al importar el workflow, elige manualmente el **Pipeline** y la **fase (Stage)** del ticket en el nodo `Escalate to Human Advisor`.

### Paso 3: Personalizar el Estado del lead

1. **Configuración → Objetos → Contactos → Propiedades de contactos**.
2. Busca **Estado del lead** (Lead Status; nombre interno `hs_lead_status`) — es una propiedad nativa de HubSpot.
3. Ábrela → **Editar** (Edit) → en la lista de opciones, elimina las que no necesites y agrega:
   `Nuevo`, `Contactado por bot`, `Calificado`, `Precio pendiente – Escalado`, `Asignado a asesor`, `Cerrado ganado`, `Cerrado perdido`.
4. Guardar.

### Paso 4: Construir el pipeline

1. **Configuración → Objetos → Negocios → Pipelines de negocios** (Deal pipelines).
2. Edita el **pipeline por defecto** o crea uno nuevo con **Crear pipeline** y nómbralo de forma neutral, por ejemplo `Ventas — Aurora/Bahia` (recuerda: un solo pipeline compartido, no uno por empresa — ver Sección 2 de esta guía).
3. En el editor de fases, reemplaza las fases por defecto con estas siete, en orden:
   1. Nuevo lead
   2. Contactado por asesor
   3. Cita / visita agendada
   4. Propuesta enviada
   5. Negociación
   6. Cerrado ganado
   7. Cerrado perdido
4. En la fase 6, marca la casilla de **Cerrado ganado** (Closed won, probabilidad 100%). En la fase 7, marca **Cerrado perdido** (Closed lost, probabilidad 0%).
5. Guardar.

### Paso 5: Crear las listas activas

1. En el menú superior, ve a **Contactos → Listas** (Lists).
2. **Crear lista** (Create list) → **Basada en contactos** (Contact-based) → **Lista activa** (Active list) — así se actualiza sola cuando cambian las propiedades.
3. Crea estas tres:
   - `Leads — Altos del Rio VI`: filtro `project` es igual a `altos-del-rio-vi`
   - `Leads — Villa Serena 3ra Etapa`: filtro `project` es igual a `villa-serena-3`
   - `Leads — Precio pendiente`: filtro `price_confirmation_pending` es igual a `Sí`
4. Guarda cada una.

### Paso 6: Crear la clave de API para n8n — Service Key (preferido)

HubSpot marca las Aplicaciones privadas de UI como *legacy* y recomienda **Service Keys** (Claves de servicio, beta pública) para integraciones solo-API como n8n. Mismos scopes, se usa como Bearer token (entra en el campo App Token de n8n), y no queda atada a un empleado. Su única limitación —no soporta webhooks— no afecta este build (n8n solo *escribe* hacia HubSpot).

1. **Desarrollo** (Development) → **Claves** (Keys) → **Claves de servicio** (Service keys) → **Crear clave de servicio**.
2. Nómbrala `n8n-agent-integration`.
3. **Añadir scope** y habilita cada uno:
   - `crm.objects.contacts.read`, `crm.objects.contacts.write`
   - `crm.objects.deals.read`, `crm.objects.deals.write`
   - `tickets` (lectura + escritura) — **obligatorio para la herramienta de escalamiento** (crea un ticket). Sin este, el nodo Escalate falla con `400 – Must have scope [tickets-write]` aunque contactos y negocios sí escriban.
   - `crm.schemas.contacts.read`
   - `crm.schemas.deals.read`
   - `crm.objects.owners.read`
   - **Update** para confirmar los scopes.
4. **Create** → confirma. Bajo la clave, **Show → Copy**. Guárdala en la credencial de HubSpot dentro de n8n (autenticación por **App Token**), nunca en un nodo de workflow ni en un archivo versionado.

**Respaldo (si no aparece el área de Service keys):** crea una **Aplicación privada** en **Configuración → Integraciones → Aplicaciones privadas → Crear una aplicación privada**, con los mismos scopes; copia el token de acceso (solo se muestra una vez) al mismo campo App Token de n8n. Migrar después es solo cambiar el token.

---

## 13. Open items to confirm with the client (feeds architecture Section 11)

- Who are the named advisors per project, so owner assignment (Section 8.1/8.2) has real HubSpot users to map to?
- Is the Habeas Data controller question (architecture Section 11.2) resolved? It determines whether `data_controller_company` is needed and whether one or two consent notices are used.
- Is a paid HubSpot tier acceptable early, or must routing/handoff stay entirely in n8n (Section 8.1) for the foreseeable future?
