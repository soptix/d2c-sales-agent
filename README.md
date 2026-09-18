# D2C sales agent: WhatsApp, Instagram and Messenger into one CRM

An AI sales agent that answers homebuyer enquiries on three channels, qualifies them,
and writes every lead into HubSpot without an advisor touching a keyboard.

Built for a residential homebuilder running two companies off a single WhatsApp number,
Facebook page and Instagram account. In production it created 152 CRM contacts from
conversations that previously lived only in a phone.

## What it does

- Answers WhatsApp, Instagram DM and Messenger in seconds, in Colombian Spanish, from one shared inbox.
- Works out which company and which construction project the buyer is asking about, then routes the lead to the advisors who handle it.
- Captures name, phone, channel, project, house type, bedrooms and financing signals, and writes them to HubSpot as structured properties.
- Escalates to a human advisor and keeps assignment in sync both ways between the inbox and the CRM.
- Never quotes a price the catalog does not confirm, and refuses to accept customer claims as product data.

## Repository

| File | What's in it |
|---|---|
| [customer-service-ai-agent-architecture.md](customer-service-ai-agent-architecture.md) | The full system design: channels, catalog model, lead capture, routing, failure modes. |
| [system-prompt-v3.md](system-prompt-v3.md) | The production agent prompt, including the guardrails that stop it inventing prices or claiming other builders' projects. |
| [hubspot-crm-setup-guide.md](hubspot-crm-setup-guide.md) | Every CRM property, pipeline and view the agent writes to, in English and Spanish. |
| [chatwoot-deployment.md](chatwoot-deployment.md) | The deployment log: what was installed, what broke, and how each problem was solved. |
| [whatsapp-ventana-24h.md](whatsapp-ventana-24h.md) | How the WhatsApp 24 hour messaging window was handled, with template copy. |
| [n8n-workflows/](n8n-workflows/) | The five n8n workflows, exported as JSON, plus a node by node README. |

## A note on the contents

This is real production work, published with the client's identity removed.

Company names, project names, advisor names, the city, phone numbers, email addresses,
hostnames, server details, portal IDs and every credential are replaced with placeholders.
Anything in angle brackets, such as `<CHATWOOT_ADMIN_TOKEN>` or `<IP_DEL_SERVIDOR>`,
was a real value. Unit prices and lot areas are rounded example figures.

The architecture, the workflows, the prompt and the problem solving are unchanged.

Some documents are in Spanish, some in English, some in both. That is how they were written
for the client's team, and they are published as they were used.
