---
name: bbc-skill-tool
description: "Builds, manages, and troubleshoots WhatsApp bots using the BuilderBot Cloud (BBC) MCP Tool v2.5. Covers bot creation for businesses needing appointment booking, citas, escalation, and conversational AI. Enforces verification after every mutation, destructive action gates, error recovery, and pre-deploy validation. USE FOR: BuilderBot, BBC, WhatsApp bot, bot para WhatsApp, MCP tool, builderbot, flow, deploy bot, QR code, crear bot, chatbot, bot creation for businesses (restaurant, salon, store, services, rental, creators, forum, tech support), debugging bot behavior, voice/image/document handling, notifications, structured data capture, managing BBC projects, project settings, zona horaria, add_chatpdf routing rules, AI-to-flow routing, outbound messaging API, Google Calendar appointments, deploy status diagnostics, builderbot_docs, builderbot_sanity_check, builderbot_blacklist, block spam numbers, lista negra, webhook, URL de eventos, plantillas de WhatsApp, WhatsApp template, plantilla Meta, HSM, WhatsApp Flows, formulario de WhatsApp, builderbot_webhook, builderbot_whatsapp_template, builderbot_whatsapp_flow. Pattern 2 (AI-powered) is the DEFAULT for 80% of real-world cases."
activation:
  # Charged when the body is at most twice this (≤7020 tokens, agentSkillBudget.spec).
  # 3510 + bbc-behavior 1750 fits the 6000 turn budget. Do not lower it.
  max_context_tokens: 3510
---

# BBC MCP Tool v2.5 — Safe-by-Default WhatsApp Bot Builder

> **v2.5 changelog (vs v2.4):** Three new tools: `builderbot_webhook`, `builderbot_whatsapp_template`, `builderbot_whatsapp_flow` (see "Webhook, WhatsApp templates and Flows"). Publishing a Flow and deleting a template are gated: the tool answers `confirmation_required` until `confirm: true`, which you set only after the person's explicit yes.

> **v2.4 changelog (vs v2.3):** Inputs are now **strict**: a wrong shape returns `code: "validation_error"` with the failing paths and a hint per mistake — it is no longer dropped silently. Shapes fixed in this version: routing `rules` live at the **top level** of the answer (never under `plugins.openai`); `options.media` and `options.gotoFlow` are **strings**; Google Calendar is `plugins.scheduleGoogleCalendar` + top-level outcome rules; `add_intent` rules need a real operator. `add_chatpdf` is created in **one call** (the platform provisions the assistant). New `builderbot_project action='settings'` (time zone, channel). `EVENTS.ACTION` and `EVENTS.IG_COMMENT` may repeat. Deploy statuses are `INITIALIZATION | READY_TO_SCAN | ONLINE | FAILED`.

## CORE PHILOSOPHY

**Three pillars:**

1. **VERIFY** — After every mutating operation, call the matching list (`builderbot_project action='list'`, `builderbot_list_flows`, `builderbot_list_answers`, `builderbot_blacklist action='list'`) to confirm.
2. **GATE** — Before every destructive operation, show impact and wait for an explicit "yes".
3. **RECOVER** — When something fails, read `code` + `hint`, fix the payload or propose a fix. Never retry the same payload unchanged, never just stop.

---

## TOOL REFERENCE (Quick)

| Tool | Purpose | Mutating? |
| --- | --- | --- |
| `builderbot_project` | Projects: `list` \| `get` \| `create` \| `update` \| `delete` \| `duplicate` \| `settings`. Start with `action='list'`; the returned `uuid` is `projectId` | `create`/`update`/`delete`/`duplicate`/`settings` yes; `delete` is DESTRUCTIVE (`confirm=true`) |
| `builderbot_list_flows` | List flows in project | No |
| `builderbot_create_flow` | Create flow with keywords | Yes |
| `builderbot_update_flow` | Update flow config | Yes |
| `builderbot_delete_flow` | Delete flow + all answers | Yes (DESTRUCTIVE) |
| `builderbot_list_answers` | List answers in flow | No |
| `builderbot_create_answer` | Add answer to flow | Yes |
| `builderbot_update_answer` | Update answer content; `plugins.openai` merges with what is stored | Yes |
| `builderbot_delete_answer` | Delete single answer | Yes (DESTRUCTIVE) |
| `builderbot_validate_bot` | Pre-publish structural health check (14 checks) | No |
| `builderbot_deploy` | Deploy: `create` \| `status` \| `qr` \| `reboot` \| `delete` | `create`/`reboot`/`delete` yes |
| `builderbot_blacklist` | Blocked phones: `list` \| `add` \| `remove`. `phones` required for add/remove; stored as-is (no E.164) | `add`/`remove` yes |
| `builderbot_sanity_check` | Runtime health of a deployed bot | No |
| `builderbot_read_logs` | Container logs; `lines` 1–500 (default 200), returns `returned`/`total`/`truncated` | No |
| `builderbot_webhook` | Event webhook of the project: `get` \| `set` \| `disable`. `set` needs `url` | `set`/`disable` yes |
| `builderbot_whatsapp_template` | Meta templates: `list` \| `get` \| `create` \| `delete`. `create` goes to Meta review | `create`/`delete` yes; `delete` is DESTRUCTIVE (`confirm=true`) |
| `builderbot_whatsapp_flow` | WhatsApp Flows: `list` \| `get` \| `create` (DRAFT) \| `publish` \| `retire` | `create`/`publish`/`retire` yes; `publish` is IRREVERSIBLE (`confirm=true`) |
| `builderbot_docs` | Official API reference: `search` \| `get` \| `list`. Honor `coverage`: only call the named MCP tool; `not_exposed` is REST-only | No |

**Retired names (do not call):** `builderbot_list_projects` → `builderbot_project action='list'`. `builderbot_create_project` → `builderbot_project action='create'`. `builderbot_get_qr` → `builderbot_deploy action='qr'`. If a call returns `code: "unknown_tool"`, retry once with the `hint`.

### Project settings

`builderbot_project(action='settings', projectId)` returns `{ provider, timeZone, phoneNumber, rejectCalls, showTypingEvents, showRecordingEvents }`. Pass `settings: {...}` with the fields to change. `timeZone` is an IANA name (`America/Mexico_City`); `provider` ∈ `baileys | baileys_code | telegram | meta | gohighlevel | gohighlevel_bridge | facebook | instagram | webchat`. API keys are never returned. **Set the time zone before building calendar or reminder flows** — appointment slots and reminders are computed in it.

### Webhook, WhatsApp templates and Flows

All three take `projectId` + `action`. Templates and Flows need the project connected through Meta (Conexión Rápida); if the call is refused for that, tell the person to connect first.

* **`builderbot_webhook`**: `set` with `url` (https) sends every bot event as a POST with a JSON body. `headers: [{ key, value }]` are sent with each event; omit `headers` to **keep** the stored ones, `[]` removes them. Values come back as `[hidden]`. `disable` stops events. VERIFY with `get`.
* **`builderbot_whatsapp_template`**: `create` submits to **Meta review**; it is not usable until `APPROVED`, so never promise it. After `create`, run `get` and report the real status (`PENDING`, `APPROVED`, `REJECTED`). `name` is lowercase letters, digits and `_`; `language` defaults to `es`; `category` is `MARKETING | UTILITY | AUTHENTICATION`. Exactly one `BODY` (≤1024 chars; header and footer ≤60; button text ≤25). Every `{{n}}` variable needs example values or Meta rejects it. A `FLOW` button needs the `flow_id` of a **published** Flow.
* **`builderbot_whatsapp_flow`**: `create` uploads a **DRAFT** (`name`, `categories`, `flowJson` as an object, not text). `publish` freezes it: it can never be edited or deleted again, only deprecated with `retire`. Use `version: "7.3"`; the simplest valid Flow is one terminal screen whose `Footer` runs `complete`, so the answers reach the bot without an endpoint. Do not set `publish: true` on `create` unless the person asked to publish.

```
builderbot_whatsapp_template { projectId, action: "create", name: "recordatorio_cita", language: "es", category: "UTILITY",
  components: [
    { type: "BODY", text: "Hola {{1}}, tu cita es el {{2}}.", example: { body_text: [["Ana", "lunes 10:00"]] } },
    { type: "BUTTONS", buttons: [{ type: "QUICK_REPLY", text: "Confirmar" }] } ] }

builderbot_whatsapp_flow { projectId, action: "create", name: "contacto", categories: ["CONTACT_US"],
  flowJson: { version: "7.3", screens: [{ id: "CONTACTO", title: "Contacto", terminal: true, success: true, data: {},
    layout: { type: "SingleColumnLayout", children: [
      { type: "TextInput", name: "nombre", label: "Nombre", "input-type": "text", required: true },
      { type: "Footer", label: "Enviar", "on-click-action": { name: "complete", payload: { nombre: "${form.nombre}" } } } ] } }] } }
```

**Order for a template with a Flow button:** `flow create` → `get` and show it → person says yes → `publish` with `confirm: true` → `template create` with `flow_id` → `template get` for the status. If `create` fails on the Flow JSON, the error lists what Meta rejected: fix those paths and resend, never the same JSON.

Sending a template to a contact is **not** an MCP tool (REST, see below).

---

## MANDATORY PATTERNS

### Pattern 1: VERIFY after every mutation

```
builderbot_project(action='create') → builderbot_project(action='list') → find the new uuid
create_flow(projectId, ...)         → list_flows(projectId) → find the flow, keep its uuid
create_answer(...)                  → list_answers(projectId, flowId) → find the answer
update_answer(...)                  → list_answers(projectId, flowId) → confirm the change
delete_flow(...)                    → list_flows(projectId) → confirm it is gone
builderbot_blacklist(add|remove)    → builderbot_blacklist(action='list') → confirm phones
builderbot_webhook(set|disable)     → builderbot_webhook(action='get') → confirm url / headers
whatsapp_template(create|delete)    → whatsapp_template(action='get'|'list') → report Meta's status
whatsapp_flow(create|publish|retire)→ whatsapp_flow(action='get') → confirm status
```

If verification fails: STOP, report the discrepancy, diagnose, propose recovery.

### Pattern 2: GATE before destructive actions

Before `delete_flow`, `delete_answer`, `deploy(action='delete')`, `builderbot_project(action='delete')`, `builderbot_whatsapp_template(action='delete')`, `builderbot_whatsapp_flow(action='publish'|'retire')`: show what is affected (flow name, answer count, references), say it cannot be undone, wait for the person's yes, then VERIFY and run `validate_bot` for broken references.

### Pattern 3: RECOVER on failure

| `code` | Meaning | Do |
| --- | --- | --- |
| `validation_error` | Wrong shape; `message` lists the paths, `hint` says the right shape | Fix the payload per the hint and resend. Never resend unchanged |
| `assistant_provision_failed` | The `add_chatpdf` answer exists but its AI assistant does not | `builderbot_update_answer { answerId, assistant: { instructions } }`. **Do not create the answer again** |
| `builderbot_api_conflict` | 409: duplicate event keyword (lists `events`) or a deploy already exists | Rename / reuse the flow; for deploy use `action='status'` |
| `builderbot_api_payment_required` | 402: plan limit; message carries the billing URL | Tell the person the limit and the URL. Do not retry |
| `deploy_qr_not_ready` | Status is not `READY_TO_SCAN` | Follow the hint (`ONLINE` = already linked; `INITIALIZATION` = poll status) |
| `unknown_tool` | Retired name or cached tool list | Retry once with the `hint` |
| `confirmation_required` | A destructive or irreversible action ran without `confirm: true` | Show what changes, ask for the person's yes, and only then resend with `confirm: true`. Never add it on your own to get past the error |
| `BuilderBot API key required` | MCP connector lost its key | Person reconnects BBC MCP TOOL in Claude → Settings → Connectors |

When the body is unclear, `builderbot_docs(action='search', query=...)` before guessing.

---

## FLOW CREATION RULES

```
projectId:        UUID (from builderbot_project action='list')
name:             "Human Readable Name" (Title Case, person's language)
label:            "snake_case_slug"
keywords:         ["keyword1", "keyword2"] or ["EVENTS.WELCOME"]
listenKeywords:   false (ALWAYS false unless EVENTS.VOICE_NOTE)
transcribeAudio:  true only for EVENTS.VOICE_NOTE
interpretImage:   true only for EVENTS.MEDIA
analyzeDocument:  true only for EVENTS.DOCUMENT
```

* A new flow has 0 answers — the bot will NOT respond until you add answers. Add them IMMEDIATELY.
* Never mix `EVENTS.*` and text keywords in one flow. Text keywords must be unique across flows.
* **Each `EVENTS.*` event except `EVENTS.ACTION` and `EVENTS.IG_COMMENT` can exist in only ONE flow per project.** `EVENTS.ACTION` and `EVENTS.IG_COMMENT` may repeat (the platform exempts them; `validate_bot` ignores those duplicates).
* Flows reached only by routing rules (`_save_order_`, `_book_appointment_`) use a custom text label as keyword, not an event.

| Event | Use case | Required flags |
| --- | --- | --- |
| `EVENTS.WELCOME` | Catch-all / fallback | listenKeywords: false |
| `EVENTS.VOICE_NOTE` | Voice message handler | listenKeywords: true, transcribeAudio: true |
| `EVENTS.MEDIA` | Image handler (vision) | interpretImage: true |
| `EVENTS.DOCUMENT` | Document handler | analyzeDocument: true |
| `EVENTS.LOCATION` | Location shared | — |
| `EVENTS.ACTION` | Generic action sink | may repeat |

---

## ANSWER SHAPES (strict — unknown keys are rejected)

`type` must be one of: `add_text`, `add_image`, `add_video`, `add_doc`, `add_voice_note`, `add_http`, `add_calendar`, `add_mute`, `add_intent`, `add_reminder`, `add_follow_up`, `add_forward_message`, `add_voice_ai`, `add_structured_output`, `add_google_calendar`, `add_chatpdf`, `add_shopify`. Bare names (`text`, `image`) are rejected.

### Routing rules (top level, any type)

```
rules: [{ conditionRule: "PEDIDO CONFIRMADO", condition: "includes", conditionValue: "", conditionFlowId: "<flow uuid>" }]
```

`condition` ∈ `=== | !== | > | < | >= | <= | startsWith | includes | notIncludes | regex | notRegex | length> | length< | length=`. `rules` is a top-level field of the answer — **never** inside `plugins.openai`. `plugins.http.rules` and `plugins.intent.rules` are the only plugin-level rules.

### add_text

```
type: "add_text", message: "Your text here"
options: { capture: true }   // optional: wait for the reply and pass it to the NEXT answer
```

Keep messages short (~160 chars) for readability; WhatsApp does **not** truncate them. **Never `capture: true` on the LAST answer** — the value is lost.

### add_chatpdf — AI-powered assistant (Pattern 2, DEFAULT)

One call. The platform creates the assistant and returns `assistantId`.

```
type: "add_chatpdf"
message: ""
plugins: { openai: {
  assistantName: "Nora",
  assistantInstructions: "You are a helpful assistant for [business]...",   // REQUIRED
  assistantModel: "gpt-5.4-nano",                                            // default; gpt-5.4-mini for harder tasks
  assistantInterpretMultimedia: true                                         // optional: vision on EVENTS.MEDIA flows
} }
rules: [ { conditionRule: "PEDIDO CONFIRMADO", condition: "includes", conditionValue: "", conditionFlowId: "<save-order uuid>" } ]
```

* `assistantInstructions` is the full system prompt. Leave `message` empty: the AI writes the reply.
* While generating, BBC scans the top-level `rules` against the AI's text; a match jumps to `conditionFlowId` after the reply, with the text in `{aiResponse}`. Teach the AI to emit uppercase markers ("PEDIDO CONFIRMADO", "AGENDAR CITA") only when an action should fire.
* Build destination flows FIRST (you need their uuids), then create the chatpdf with its `rules`, or add them later with `update_answer { rules }`.
* **Single-answer rule:** a flow with `add_chatpdf` contains ONLY that answer. `validate_bot` marks anything else critical.
* Editing: `update_answer { answerId, assistant: { instructions, model } }` or `plugins: { openai: {...} }` — partial patches merge with what is stored.
* Knowledge: `update_answer { answerId, assistant: { scrapeUrl } }` to ingest a page; `{ listFiles: true }` / `{ deleteFileId }` to manage files.

### add_image / add_video / add_doc / add_voice_note

```
type: "add_image", message: "", options: { media: "https://public-url.com/image.jpg" }
```

`media` is a **string URL**, not `{ url }`. `validate_bot` flags a media answer without it.

### add_http

```
type: "add_http", message: ""
plugins: { http: {
  url: "https://api.example.com/endpoint", method: "GET" | "POST",
  headers: { "Content-Type": "application/json" },                       // optional
  body: { pedido: "{aiResponse}", numero: "{from}" },                     // optional (POST)
  rules: []                                                               // ALWAYS present; route on the response payload
} }
```

### add_mute — human handoff (mutes ONE contact)

```
type: "add_mute", plugins: { mute: { status: true, gapTime: 60 } }, options: { gotoFlow: "<farewell flow uuid>" }
```

`gotoFlow` is a **string uuid**, not `{ flowId }`. `gapTime` = minutes until auto-unmute.

### add_intent — AI semantic routing

```
type: "add_intent"
plugins: { intent: { rules: [{ conditionRule: "The user wants to talk to a human", condition: "===", conditionValue: "", conditionFlowId: "<uuid>" }] } }
```

`condition` must be a real operator (`"==="`), never `""`.

### add_structured_output — silent typed extraction

```
type: "add_structured_output", ref: "lead"
plugins: { structuredOutput: { properties: [
  { key: "nombre", type: "string", promptKey: "Nombre del cliente" },
  { key: "fecha",  type: "string", promptKey: "Fecha deseada" } ] } }
```

Sends no bubble. Put it first in the flow a rule routes to; each value is stored as `{ref}_{clave}` → `{lead_nombre}` for the next steps.

### add_google_calendar — native appointment scheduling

```
type: "add_google_calendar", message: ""
plugins: { scheduleGoogleCalendar: { action: "create", calendarId: "xyz@group.calendar.google.com", durationMeeting: 60 } }
rules: [
  { conditionRule: "available",     condition: "===", conditionValue: "", conditionFlowId: "<confirm uuid>" },
  { conditionRule: "not_available", condition: "===", conditionValue: "", conditionFlowId: "<offer-slot uuid>" },
  { conditionRule: "misunderstood", condition: "===", conditionValue: "", conditionFlowId: "<ask-date uuid>" } ]
```

There is **no** `plugins.calendar`. Outcomes are `available | not_available | misunderstood`, routed with top-level rules. Set the project `timeZone` first.

### Redirect

`options.gotoFlow: "<target flow uuid>"` → jump after this answer. `validate_bot` marks a jump to a missing flow critical.

---

## SYSTEM VARIABLES

| Variable | Meaning | Where |
| --- | --- | --- |
| `{from}` | Contact's WhatsApp number | All answer types |
| `{name}` | Contact's display name (if available) | All answer types |
| `{aiResponse}` | Text the AI just generated in the preceding `add_chatpdf` | Flows reached via top-level `rules`, typically an `add_http` body |
| `{time}` `{date}` `{fullDate}` | Current server time/date (24h `HH:mm`). Calendar and reminder steps use the project `timeZone`; for open/closed logic in a message, prefer an `add_http` to a TZ-aware backend | `assistantInstructions`, messages |
| `{ref}_{clave}` | Fields an `add_structured_output` extracted (`{lead_nombre}`) | Steps after the extraction |

---

## OUTBOUND MESSAGING REST API (backend → contact)

Not an MCP tool; a plain HTTP call your backend makes: `POST https://app.builderbot.cloud/api/v2/{projectId}/messages` with header `x-api-builderbot: <api key>` and body `{ "messages": { "content": "..." }, "number": "584246743207", "checkIfExists": false }` (E.164 without "+"). Skip numbers in `builderbot_blacklist(action='list')` first. Sending an approved **template** is the same REST family (`builderbot_docs(action='search', query='whatsapp-template')` gives the body).

---

## DECISION TREE

```
Appointments / citas / availability → Pattern 2 (AI) + add_google_calendar ★
Q&A about products or services      → Pattern 2 (AI)
Order tracking / status             → Pattern 2 (AI) + add_http
Order + pay + confirm               → Pattern 2 + top-level rules → one action flow per marker
Simple static info (hours, address) → Pattern 1 (keyword → add_text)
Lead qualification                  → Pattern 2 (AI) + add_structured_output
Human handoff                       → Pattern 2 (AI) + add_mute
```

**DEFAULT**: Pattern 2 (`add_chatpdf`) for 80% of real cases.

### Vertical sketches

* **Salon / services:** WELCOME `add_chatpdf` (services, prices, hours) with rule "AGENDAR CITA" → calendar flow (`add_google_calendar`, three outcome flows); "DERIVAR A AGENTE" → `add_mute` flow.
* **Restaurant / food:** WELCOME `add_chatpdf` (menu, delivery) with rules "PEDIDO CONFIRMADO" → `add_http` POST order; "COMPROBANTE RECIBIDO" → save-payment flow. `EVENTS.MEDIA` flow with `interpretImage: true` + `assistantInterpretMultimedia: true` for receipts.
* **Store / e-commerce:** WELCOME `add_chatpdf` (catalog, shipping); "CONSULTAR ESTADO" → `add_http` GET order API.
* **Creator / event:** WELCOME `add_chatpdf` (schedule, pricing); "GENERAR LINK DE PAGO" → checkout flow; registration = `add_structured_output` + `add_http` POST.

---

## COMPLETE WORKFLOW: Building a Bot

### Step 1: Project and settings

```
builderbot_project(action='list') → existing uuid, or builderbot_project(action='create', name) → list → VERIFY
builderbot_project(action='settings', projectId) → if the business is not in that timeZone (or it is empty), set it
```

### Step 2: Plan

Before creating anything, write the plan in your reply under this exact heading, as plain Markdown (no code block). The panel turns it into a checklist and ticks each row when the matching write finishes — **never tick a box yourself**, leave every row `- [ ]`:

### Plan
- [ ] flow: Bienvenida (EVENTS.WELCOME, IA)
- [ ] answer: Bienvenida — add_chatpdf
- [ ] flow: Guardar pedido (_save_order_)
- [ ] answer: Guardar pedido — add_http

One `flow:` row per flow and one `answer:` row per answer, in the order you will create them.

### Step 3: Create flows (one at a time)

```
1. create_flow(...)               → list_flows → VERIFY, keep the flow uuid
2. create_answer(...)             → list_answers → VERIFY (an add_chatpdf comes back with assistantId)
3. next flow. Destination flows before the chatpdf that routes to them.
```

**NEVER create all flows first and add answers later.** Flow → answers → verify → next flow.

### Step 4: Validate

```
validate_bot(projectId)
├── criticalCount = 0 → ready to deploy
└── criticalCount > 0 → fix first: dead ends (empty flows), dangling capture, missing WELCOME,
    add_chatpdf sharing a flow, event flags, broken gotoFlow/rule targets, media without URL, add_http without url
```

### Step 5: Deploy (with GATE)

```
1. validate_bot → criticalCount = 0
2. One sentence: what goes live (bot name, N flows, validation passed). Ask for a yes.
3. WhatsApp not connected yet → send the person to [Conexión Rápida](/project/<uuid>/deploy/meta)
   (they sign in with Meta; you cannot do it). Wait until they say it is done.
4. deploy(action='create') → deploy(action='status')
5. READY_TO_SCAN only on a legacy WhatsApp QR project → deploy(action='qr') → show QR
6. ONLINE → builderbot_sanity_check(projectId). Unhealthy → builderbot_read_logs(projectId)
```

**WhatsApp QR is retired** for new bots. New WhatsApp = **Conexión Rápida** (official Meta). Never explain "Dispositivos vinculados → QR"; if asked for the QR, say it was replaced.

---

## DEPLOY STATUS

| Status | Meaning | Action |
| --- | --- | --- |
| `ONLINE` | Bot is live and receiving messages | Nothing — this is "prendido" |
| `READY_TO_SCAN` | Legacy WhatsApp QR project waiting for a scan | `deploy(action='qr')`; recommend Conexión Rápida |
| `INITIALIZATION` | Spinning up | Re-check status in ~30 s |
| `FAILED` | Deploy crashed | `builderbot_read_logs`; `deploy(action='reboot')` or support |

**"How many bots are on?":** `builderbot_project(action='list')`, then `builderbot_deploy(action='status')` per project. Only `ONLINE` counts.

---

## FINAL REPORT FORMAT

Plain Markdown (no code block, no ids, no tool names, no emoji headers):

1. One sentence with the outcome: the bot is ready, or what is still missing.
2. A table `| Flujo | Se activa con | Mensajes |`, triggers in plain words ("al saludar", "palabra «precio»").
3. If an AI answer routes to flows, one bullet per rule: what the customer says → which flow.
4. Validation in one line (passed, or the warnings that matter). Problems you fixed, one bullet each.
5. Next step: WhatsApp connected and live, or link [Conexión Rápida](/project/<uuid>/deploy/meta).

## FORMATTING FOR THE CHAT PANEL (~420px, Markdown)

- Lists or tables of bots: in-app link in the FIRST cell, id from the tool (never invented): `| [BOT VETERINARIA](/project/<id>) | 7 oct 2026 | 0 |`.
- Section title = a line that is only bold text, no final period. ≤8 bullets per group; tables 2–4 columns, ≤10 rows.
- Dozens of results: total, the 8 most recent, a filter offer. No uuids or JSON unless asked. End with at most one question.

---

## ANTI-PATTERNS (Never Do These)

1. Creating a flow without immediately adding answers
2. Reporting success without verification
3. Deleting without showing impact and getting confirmation
4. Deploying without `validate_bot` first
5. Mixing `add_chatpdf` with any other answer in the same flow
6. `listenKeywords: true` on non-VOICE_NOTE flows
7. `capture: true` on the last answer of a flow
8. Batch-creating all flows before adding answers
9. `rules` inside `plugins.openai`; `media: { url }`; `gotoFlow: { flowId }`; `plugins.calendar`; `type: "text"` — all rejected with `validation_error`
10. Creating `add_http` without `plugins.http.rules` (`[]` is fine)
11. Resending a payload that returned `validation_error` without changing it, or creating a second `add_chatpdf` after `assistant_provision_failed`
12. Building calendar or reminder flows before setting the project `timeZone`
13. Putting assistant instructions in `message` instead of `plugins.openai.assistantInstructions`
14. Calling `builderbot_list_projects` / `builderbot_create_project` — removed; use `builderbot_project` with `action`
15. Sending the person to the Cloud panel or a homemade list to block spam — use `builderbot_blacklist`
16. Setting `confirm: true` without a yes from the person in this conversation, or publishing a Flow "to test it" (publishing is irreversible)
17. Saying a template is ready, or using it as a `FLOW` button target, before Meta shows it `APPROVED` / the Flow is published
18. Sending a template with `{{n}}` variables without `example` values, or a Flow `flowJson` as text with comments or trailing commas
