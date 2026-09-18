# D2C Sales Agent — n8n workflows

The live build is **three workflows** running on n8n, with **Chatwoot** as the single channel layer for all three channels (WhatsApp, Instagram, Facebook Messenger) and **HubSpot** as the CRM. WhatsApp reaches Chatwoot through a YCloud bridge; Instagram and Messenger are native Chatwoot channels.

| File | n8n workflow id | Role |
|---|---|---|
| `D2C Sales Agent — Chatwoot (WhatsApp + IG + Messenger).json` | `tF4UU5eskdvrPmoC` | **The agent brain.** Chatwoot inbound → Claude agent (catalog + CRM tools) → Chatwoot reply. |
| `D2C — CRM Registrar Lead (sub-workflow).json` | `FlAO2qRZUzGWFmfE` | **CRM writer.** One call upserts the contact and, optionally, creates the deal and/or the escalation ticket, with associations. Called by the agent as a tool. |
| `D2C — Puente YCloud ↔ Chatwoot.json` | `rt9rQXfzHW3XXNpw` | **WhatsApp bridge.** YCloud ↔ Chatwoot in both directions, because Chatwoot has no native YCloud provider. |
| `D2C — Sync asignación Chatwoot → HubSpot.json` | `pTvpKQtFgq6PN64K` | **Assignment sync, inbound.** Advisor takes a chat in Chatwoot → writes her name into the contact's `agente` property in HubSpot. |
| `D2C — Sync asignación HubSpot → Chatwoot.json` | `NtcfpSYzUWyNITwt` | **Assignment sync, outbound.** Advisor sets `agente` on a HubSpot contact → assigns the Chatwoot conversation to her. HubSpot Trigger, ~2s, with a 15-min poll as backup. |

```
                     WhatsApp ──(YCloud)──┐
                                          ├─→  D2C — Puente YCloud ↔ Chatwoot  ──┐
                     Instagram ─(native)──┤                                      │
                     Messenger ─(native)──┘                                      ▼
                                                                             CHATWOOT
                                                                                 │  message_created webhook
                                                                                 ▼
                                     D2C Sales Agent — Chatwoot  ──(tool)──→  D2C — CRM Registrar Lead  ──→  HubSpot
                                                 │
                                                 └──→ reply back through the same Chatwoot conversation
```

For the server/infra side of Chatwoot, the YCloud bridge and the self-hosted n8n, see [`../chatwoot-deployment.md`](../chatwoot-deployment.md). This README covers the workflows themselves.

---

# 1. `D2C Sales Agent — Chatwoot (WhatsApp + IG + Messenger)` — the agent

All three channels enter and leave through **Chatwoot**. n8n never talks to a channel provider from this workflow: Chatwoot owns the provider integrations (WhatsApp via the API-inbox bridge, IG/FB natively), and n8n sees one uniform Chatwoot message shape.

## Flow

```
Chatwoot Webhook (all inboxes) → Is Chatwoot inbound text? → Normalize Chatwoot Message → Normalized Message
                                                                                                  ↓
                             Log Conversation (Sheets) ← Send Chatwoot Reply ← Sales Agent ←──────┘
```

- **Chatwoot Webhook (all inboxes)** — generic n8n Webhook (`POST /webhook/chatwoot-inbound`). Chatwoot's account-level webhook fires for every inbox, so the filter does the narrowing.
- **Is Chatwoot inbound text?** — passes only when **all** of these hold:
  - `event = message_created`
  - `message_type = incoming` (drops the bot's own outgoing replies, so no echo loop)
  - `content` is not empty (drops pure attachments)
  - `private = false` (drops agent private notes)
  - `conversation.channel` matches `Whatsapp|Instagram|FacebookPage|Api` — the `Api` inbox is the WhatsApp/YCloud bridge inbox
  - **`conversation.meta.assignee` is empty** — the handoff pause: if a human advisor is assigned to the conversation, the agent stays quiet.
- **Normalize Chatwoot Message** — derives a uniform shape (see table). WhatsApp arrives on either a `Whatsapp` or `Api` channel and is treated as `whatsapp` in both cases; Instagram → `instagram`; anything else → `messenger`. Keeps `conversation_id` + `account_id` for the reply and logging.
- **Sales Agent** — `@n8n/n8n-nodes-langchain.agent` v3.1, `maxIterations: 15`. Its system message is mirrored in [`../system-prompt.md`](../system-prompt.md). Attached:
  - **Claude (reasoning)** — `lmChatAnthropic` v1.5, model **`claude-sonnet-5`**, `temperature: 0`.
  - **Conversation Memory** — `memoryBufferWindow` v1.3, keyed on `session_id`, `contextWindowLength: 50`.
  - Tools: the three catalog tools and the CRM sub-workflow tool (see §1.2).
- **Send Chatwoot Reply** — `POST https://chatwoot.example.com/api/v1/accounts/{account_id}/conversations/{conversation_id}/messages` with `message_type: outgoing`. Chatwoot then delivers over whichever provider that inbox uses (for WhatsApp that means the bridge picks it up and sends via YCloud — see §3).
- **Log Conversation (Sheets)** — appends every turn to the `Conversaciones` tab of the catalog sheet. Runs *after* the reply is sent and is `onError: continueRegularOutput`, so a Sheets failure never blocks the customer's answer.

## 1.1 Normalized message shape

| Field | WhatsApp (`Whatsapp`/`Api`) | Instagram / Messenger |
|---|---|---|
| `channel` | `whatsapp` | `instagram` / `messenger` |
| `sender_id` | `sender.phone_number` (falls back to `sender.identifier`) | Chatwoot contact `sender.id` |
| `message_text` | `content` | `content` |
| `contact_name` | `sender.name` | `sender.name` (Chatwoot resolves IG/FB display names) |
| `session_id` | `whatsapp-<phone>` | `<channel>-<contact_id>` |
| `conversation_id`, `account_id` | carried for reply + logging | carried for reply + logging |

> **Memory is keyed per contact, not per Chatwoot conversation.** If an advisor resolves a conversation and the customer writes again, Chatwoot opens a *new* conversation but the same contact — keying on the contact (`session_id`) keeps the agent's history intact across that boundary. `conversation_id` is carried separately so the reply lands in the right thread.

## 1.2 Tools wired into the agent

The catalog lives in one Google Sheet (`documentId` `<GOOGLE_SHEET_ID>`, "Catálogo de Proyectos — Agente de IA").

| Tool | Node type | Source | Purpose |
|---|---|---|---|
| **List Active Projects** | `googleSheetsTool` | `Proyectos` tab | Lists active projects (`project_id`, name, company, city/sector, delivery state, financing banks) when the customer asks generally what's available |
| **Get Project Info** | `googleSheetsTool` (filtered by `project_id`) | `Proyectos` tab | Descriptive answers for one project: finishes/materials, location/sector, what's included, financing banks, FAQ |
| **Look Up Unit Price & Details** | `googleSheetsTool` (returns the full tab) | `Tipos de Unidad` tab | Every unit type across all projects — `project_id, tipo_unidad, alcobas, area_m2, posicion, precio_cop, cuota_inicial_30, saldo_70, disponibilidad`. The agent filters itself and is told never to quote a price without this |
| **Call `D2C — CRM Registrar Lead (sub-workflow)`** | `toolWorkflow` | → workflow `FlAO2qRZUzGWFmfE` | Single call that saves the lead and, via flags, creates the deal and/or escalates (see §2) |

> **The CRM is now one tool, not three.** Earlier builds gave the agent three separate HubSpot tool nodes (`Save Contact`, `Create Deal`, `Escalate to Human Advisor`). Those nodes still exist in this workflow but are **`disabled`** — they've been superseded by the single sub-workflow call, which does contact + deal + ticket + associations in one deterministic pass. Leave them disabled; they're kept only for reference.

---

# 2. `D2C — CRM Registrar Lead (sub-workflow)` — the CRM writer

Called by the agent's `Call D2C — CRM Registrar Lead` tool. One invocation does everything the lead needs, gated by two booleans (`create_deal`, `escalate`), so the agent decides *what* to record and the sub-workflow decides *how*.

## Flow

```
Input → Save Contact (HubSpot) → IF create deal ─true→ Create Deal (HubSpot) ─┐
                                                └false──────────────────────────┴→ After Deal → IF escalate ─true→ Create Ticket (HubSpot) → IF deal exists ─true→ Associate Deal↔Ticket ─┐
                                                                                                          └false───────────────────────────────────────────────────────────────────────────┴→ Result
```

- **Input** — `executeWorkflowTrigger` with the typed fields the agent passes: identity (`sender_id`, `channel`, `contact_name`, `account_id`, `conversation_id`), lead data (`firstname`, `lastname`, `email`, `phone`, `project_id`, `company_slug`, `needs_credit`, `financing_bank`, `bedrooms_wanted`, `price_confirmation_pending`, `data_consent_given`), deal data (`create_deal`, `unit_type`, `amount`) and escalation data (`escalate`, `ticket_name`, `ticket_description`).
- **Save Contact (HubSpot)** — always runs. Upserts by email; if the customer gave none, synthesizes `sin-correo-<channel>-<sender_id>@leads.example.com` (a real TLD — HubSpot rejects `.lead`/`.invalid`). Writes the custom properties `whatsapp_id`, `lead_channel` (`instagram` → `instagram_dm`), `project`, `lead_company`, `needs_credit`, `financing_bank_preference`, `bedrooms_wanted`, `price_confirmation_pending`, `conversation_transcript_url`, `data_consent_given`, plus `firstName`/`lastName`/`phoneNumber` (phone falls back to `sender_id` on WhatsApp).
- **IF create deal → Create Deal (HubSpot)** — when `create_deal = true`. Creates the deal at stage `1399089990`, **associated to the contact** (`associatedVids`), with `amount` and the custom properties `project`, `lead_company`, `unit_type_interest`, `financing_bank_preference`. Deal name: `{contact_name} — {project_id} (unit_type)`.
- **IF escalate → Create Ticket (HubSpot)** — when `escalate = true`. Creates a `HIGH`-priority ticket associated to the contact, with `ticket_name` as subject and `ticket_description` as body. *(This node currently writes only the native subject/description/priority + contact association — no custom ticket properties; see the note in `hubspot-crm-setup-guide.md §9.1`.)*
- **IF deal exists → Associate Deal↔Ticket** — if the run both created a deal and a ticket, a HubSpot v4 API `PUT` associates them so the advisor sees the ticket on the deal.
- **Result** — returns `status: ok`, `contact_vid`, `deal_created`, `ticket_created` to the agent.

> **Associations are automatic now.** Contact↔Deal and Deal↔Ticket are linked by id inside this sub-workflow — the earlier "linking is manual, Phase 2" caveat no longer applies.

---

# 3. `D2C — Puente YCloud ↔ Chatwoot` — the WhatsApp bridge

Chatwoot v4.16's `Channel::Whatsapp::PROVIDERS` is only `default` + `whatsapp_cloud` (360dialog / Meta Cloud API); **YCloud is not supported**. So WhatsApp runs through a Chatwoot **API inbox** (`inbox_id = 1`) with this workflow bridging YCloud ↔ Chatwoot in both directions. 21 nodes, two independent branches.

## Inbound: YCloud → Chatwoot

```
Webhook YCloud (entrante) → ¿Es texto entrante? → Normalizar WhatsApp → Buscar contacto → ¿Contacto existe?
   → (yes) Contacto existente ┐                                                              → (no) Crear contacto → Contacto nuevo ┐
                              └→ Datos del contacto → Buscar conversación → ¿Conversación abierta? ──────────────────────────────────┘
                                    → (yes) Conversación existente ┐
                                    → (no)  Crear conversación → Conversación nueva ┴→ Publicar mensaje entrante (incoming)
```

- **Webhook YCloud (entrante)** — `POST /webhook/ycloud-inbound-2`. Filters to `whatsapp.inbound_message.received` of `type: text`.
- Normalizes the phone to E.164, the text, and the profile name, then **finds-or-creates** the Chatwoot contact and an open conversation on inbox `1`, and posts the customer's message as `incoming`. From there the agent workflow (§1) picks it up via the `message_created` webhook.

## Outbound: Chatwoot → YCloud

```
Webhook Chatwoot (saliente) → ¿Saliente y público? → Enviar por YCloud → ¿Falló el envío?
                                                                              → (yes) Nota privada: no entregado
                                                                              → (no)  Entregado
```

- **Webhook Chatwoot (saliente)** — `POST /webhook/chatwoot-outbound`. This is the API inbox's `webhook_url`. Passes only `message_type = outgoing`, `private = false`, non-empty `content`.
- **Enviar por YCloud** — `POST https://api.ycloud.com/v2/whatsapp/messages/sendDirectly`, `from: +57XXXXXXXXXX`, `to` the customer's number, `onError: continueRegularOutput`.
- **¿Falló el envío? → Nota privada: no entregado** — if YCloud returns an error or no message id, posts a **private note** into the Chatwoot conversation explaining the likely cause (the WhatsApp 24-hour window expired) so the advisor knows the reply didn't reach the customer. This is a mitigation, not a fix — see the known limitations in `chatwoot-deployment.md`.

---

# 4. Before it runs — setup

## 4.1 Credentials (in n8n)

- **Chatwoot API** — Header Auth credential: header `api_access_token`, value = a Chatwoot agent-bot or admin access token. Used by **Send Chatwoot Reply** (agent) and by all the bridge's Chatwoot calls.
- **YCloud API** — Header Auth credential: header `X-API-Key`, value = the YCloud API key. Used by **Enviar por YCloud**.
- **Anthropic** — for Claude Sonnet 5.
- **Google Sheets (OAuth2)** — for the four catalog/log nodes.
- **HubSpot** — App Token (Service Key), used by the sub-workflow's contact/deal/ticket nodes and the association call. See `hubspot-crm-setup-guide.md`.

## 4.2 Catalog sheet

One Google Sheet, "Catálogo de Proyectos — Agente de IA", with three tabs:

- **`Proyectos`** — one row per project. Fact columns (`project_id`, name, company, city/sector, delivery state, financing banks, price validity year) *and* description columns (`acabados`, `ubicacion_descripcion`, `incluye`, `faq`). `List Active Projects` filters on active status; `Get Project Info` reads the description columns.
- **`Tipos de Unidad`** — one row per sellable unit type, linked by `project_id`: `tipo_unidad, alcobas, area_m2, posicion, precio_cop, cuota_inicial_30, saldo_70, disponibilidad`. Leave `precio_cop` blank where not confirmed (e.g. Villa Serena) — the agent is told to say so and capture the lead rather than guess.
- **`Conversaciones`** — append-only transcript log (one row per turn): `timestamp, conversation_id, channel, contact_name, sender_id, user_message, agent_response`. `conversation_id` here is the same `session_id` used as the memory key, so the transcript and the agent's actual context line up. Create it with those headers before the first run — `Log Conversation` resolves the tab by name and errors if it's missing.

## 4.3 HubSpot

Custom properties and the pipeline/ticket setup are in [`../hubspot-crm-setup-guide.md`](../hubspot-crm-setup-guide.md). In short, the sub-workflow's nodes need, on the **Contact** object: `whatsapp_id`, `lead_channel`, `project`, `lead_company`, `needs_credit`, `financing_bank_preference`, `bedrooms_wanted`, `price_confirmation_pending`, `conversation_transcript_url`, `data_consent_given`; and on the **Deal** object: `project`, `lead_company`, `unit_type_interest`, `financing_bank_preference`. The deal stage (`1399089990`) and the ticket pipeline/stage (`0`/`1`) are set in the nodes — confirm they match your portal after import.

## 4.4 Chatwoot + bridge wiring

- **Chatwoot account webhook** → `.../webhook/chatwoot-inbound`, subscribed to `message_created` (drives the agent).
- **API inbox `webhook_url`** → `.../webhook/chatwoot-outbound` (drives the outbound bridge to YCloud).
- **YCloud inbound webhook** → `.../webhook/ycloud-inbound-2`.
- IG/Messenger are connected as native Chatwoot channels (Meta app) — no n8n involvement on the channel side.

See `chatwoot-deployment.md` for the concrete tokens, inbox ids and the Cloudflare-tunnel / Bot-Fight-Mode gotchas learned during setup.

---

# 5. Known limitations / notes

- **Text only.** Images, audio and attachments are dropped by the filter (`content` is empty for a pure attachment) on both the agent and the bridge.
- **WhatsApp 24-hour window.** An advisor reply sent more than 24h after the customer's last message is rejected by Meta; the bridge posts a private note but cannot deliver it. The bot itself never hits this (it answers in seconds). Cold-start templates are not supported.
- **Handoff pause is inbound-only per message.** The agent skips a conversation while a human is *assigned* (the `assignee` check). If no one is assigned, the agent answers. **Since the assignment sync went live, setting `agente` on a HubSpot contact also pauses the bot on that conversation**, because the sync assigns it in Chatwoot. That is usually what you want, but it is a side effect worth knowing.
- **Unknown channels** matching none of `Whatsapp|Instagram|FacebookPage|Api` are dropped, so a web-widget inbox on the same account won't get bot replies.

## Node types / versions (verified against the live workflows)

- AI Agent: `@n8n/n8n-nodes-langchain.agent` v3.1
- Anthropic Chat Model: `@n8n/n8n-nodes-langchain.lmChatAnthropic` v1.5 — model `claude-sonnet-5`
- Simple Memory: `@n8n/n8n-nodes-langchain.memoryBufferWindow` v1.3
- Google Sheets tool / regular: `n8n-nodes-base.googleSheetsTool` / `n8n-nodes-base.googleSheets` v4.7
- HubSpot tool / regular: `n8n-nodes-base.hubspotTool` / `n8n-nodes-base.hubspot` v2.2
- Call sub-workflow tool: `@n8n/n8n-nodes-langchain.toolWorkflow` v2.2
- Sub-workflow trigger: `n8n-nodes-base.executeWorkflowTrigger` v1.1

---

# 6. `D2C — Sync asignación Chatwoot ↔ HubSpot` — who owns the lead

The four advisors share one HubSpot user, so `hubspot_owner_id` says nothing about who is handling a lead. The contact property **`agente`** (dropdown: Ana Gomez, Beatriz Ruiz, Carolina Diaz, Diana Mejia) carries that, and two workflows keep it in sync with the Chatwoot assignee in both directions.

| | Chatwoot → HubSpot | HubSpot → Chatwoot |
|---|---|---|
| Trigger | Account webhook, `conversation_updated` only, at `/webhook/chatwoot-asignacion` | HubSpot Trigger on `contact.propertyChange` / `agente`, plus a 15-min poll as backup |
| Scope | Events whose `changed_attributes` include `assignee_id` | The contact in the event; the poll takes contacts with `agente` set and `lastmodifieddate` in the last 20 min |
| Write | `PATCH /crm/v3/objects/contacts/{id}` with `agente` | `POST /conversations/{id}/assignments` with `assignee_id` |
| Skips when | The contact already has that `agente` | The conversation already has that assignee |

Measured latency on the HubSpot side: **2 seconds** from the property change to the Chatwoot assignment. The poll stays as a safety net because the server is a laptop that sleeps; events HubSpot cannot deliver are recovered on the next run.

The HubSpot Trigger needs a `hubspotDeveloperApi` credential, which is **not** the Service Key: it requires a HubSpot developer account, a public app inside it (App ID `<APP_ID>`), and that app installed in portal `<PORTAL_ID>`. Service Keys do not support webhooks, and the HubSpot Workflow "send webhook" action needs Operations Hub Professional.

**The join key is `conversation_transcript_url`, not the phone.** The sub-workflow writes that property on every lead it captures, and it contains the Chatwoot conversation id. Phone data is unreliable (nulls, trailing `\n`, `+` inconsistencies, and Instagram contacts have none). Consequence: a Chatwoot conversation with no HubSpot contact has nowhere to write, and is skipped without error.

Two gotchas, both found the hard way and written up in [`../chatwoot-deployment.md`](../chatwoot-deployment.md):

- **`hs_lastmodifieddate` is `null` on contacts.** Use `lastmodifieddate`. With the wrong one the poll always returns `total: 0`.
- **`conversation_updated` fires on any conversation change**, not just assignment. The webhook node's `onlyRunIf` drops the rest before an execution is created.
- **The HubSpot Trigger node renames `objectId` to `contactId`.** Filtering on `objectId` drops the event silently: the execution shows as successful and takes 50 ms. The Code node accepts both names.

The agent-to-agent map (`3` Ana, `4` Beatriz, `6` Carolina, `7` Diana) lives in a Code node in each workflow. Changing the team means editing both, plus the dropdown options in HubSpot.

## 6.1 The same `agente` field on tickets

Tickets carry their own `agente` property (internal name `agente`, verified against the API on 21/08, same four dropdown options as the contact one). Three branches inside `D2C — Sync asignación HubSpot → Chatwoot` keep it in step with the contact's, and with Chatwoot.

**The contact is the source of truth.** An advisor owns the lead, not a single ticket. Whatever direction a change comes from, it lands on the contact first and everything else follows from there.

```
contact `agente` changes ──→ assign Chatwoot conversation
                         └─→ stamp every open ticket of that contact

ticket `agente` changes ──→ write the contact ──→ back into the line above
```

| | Contact → tickets | Ticket → contact |
|---|---|---|
| Entry | The HubSpot contact event, and the 15-min contact poll | Its own 15-min poll on ticket property history. The webhook entry is built but not enabled, see below |
| Lookup | `POST /crm/v3/objects/tickets/search` filtered by `associations.contact` and `hs_pipeline_stage != 4` | `GET /crm/v3/objects/tickets/{id}?properties=agente&associations=contacts` |
| Write | `PATCH /crm/v3/objects/tickets/{id}` | `PATCH /crm/v3/objects/contacts/{id}`, then `Contacto a resincronizar` feeds the contact back into `Leer contactos (HubSpot)` |
| Skips when | The ticket already has that value | The last change to `agente` came from our own integration, is older than 20 min, is not one of the four names, or the ticket has no associated contact |

**Open ticket means pipeline `0`, stage other than `4` (Closed).** That is the only ticket pipeline in the portal. A closed ticket keeps whoever handled it.

### Why the ticket poll reads property history

A ticket's `hs_lastmodifieddate` moves on any change: stage, subject, a note. Polling on it alone, a ticket touched for an unrelated reason would push its stale `agente` onto the contact and overwrite the correct owner.

So the poll takes tickets modified in the last 20 minutes, then calls `batch/read` with `propertiesWithHistory: ["agente"]` and keeps only those whose **most recent** `agente` change is inside the window. The history entry also carries `sourceId`, which is how the branch ignores its own echo: a change written by the Service Key (`<SERVICE_KEY_APP_ID>`) is the mirror writing, not an advisor deciding. Verified on 21/08 against a real write.

`Contacto a resincronizar` closes the loop back into `Leer contactos (HubSpot)`, so the ticket branch reaches Chatwoot through the chain that was already working rather than calling Chatwoot itself, and without depending on the HubSpot contact event firing.

**The ticket poll has its own schedule trigger on purpose.** If it hung off the contacts' `Respaldo cada 15 minutos`, both branches would converge on the same nodes inside a single execution and the positional pairing in `Decidir tickets` would stop being reliable.

### Pending: the ticket webhook

`Extraer tickets del evento` is built and disconnected. Adding `ticket.propertyChange` to the trigger on 21/08 failed to register and **n8n deactivated the workflow** (about 20 seconds of downtime, then reverted). The public app `<APP_ID>` does not have the `tickets` scope, so HubSpot rejects the subscription. Confirmed separately: `GET /webhooks/v3/<APP_ID>/subscriptions` returns `403 MISSING_SCOPES`.

This is a latency upgrade, not a missing feature. The poll already covers the direction, at up to 15 minutes instead of about 2 seconds. To finish it:

1. In the HubSpot developer account that owns app `<APP_ID>`, add the `tickets` scope and update the install in portal `<PORTAL_ID>`. The scope lives in the developer UI (App → Auth → Scopes) unless the app has been migrated to the projects framework, in which case it is `requiredScopes` in `public-app.json` plus `hs project upload && hs project deploy`.
2. Add a second event to the trigger node: `ticket.propertyChange` on property `agente`.
3. Connect the trigger to `Extraer tickets del evento`.
4. Publish, then confirm the workflow is still `active`. If it deactivates again, the scope did not take.

Leave the poll in place either way. It is the safety net for a server that sleeps.
