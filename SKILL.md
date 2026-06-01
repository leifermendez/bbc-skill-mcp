---
name: bbc-skill-tool
description: "Builds, manages, and troubleshoots WhatsApp bots using the BuilderBot Cloud (BBC) MCP Tool v2.1. Covers bot creation for businesses needing appointment booking, citas, escalation, and conversational AI. Enforces verification after every mutation, destructive action gates, error recovery, and pre-deploy validation. USE FOR: BuilderBot, BBC, WhatsApp bot, bot para WhatsApp, MCP tool, builderbot, flow, deploy bot, QR code, crear bot, chatbot, bot creation for businesses (restaurant, salon, store, services, rental, creators, forum, tech support), debugging bot behavior, voice/image/document handling, notifications, structured data capture, managing BBC projects, BBC scaffolding files, add_chatpdf routing rules, AI-to-flow routing, outbound messaging API, Google Calendar appointments, deploy status diagnostics. Pattern 2 (AI-powered) is the DEFAULT for 80% of real-world cases."
---

# BBC MCP Tool v2.1 — Safe-by-Default WhatsApp Bot Builder

> **v2.1 changelog (vs v2.0):** Corrected `EVENTS.*` uniqueness rule (it's only the literal system event that must be unique, not "action-style" flows). Replaced the simplified `add_chatpdf` `assistant` config with the real `plugins.openai.*` structure. Added new sections: **Routing from add_chatpdf rules**, **System Variables**, **Deploy Status Diagnostics**, **EVENTS.MEDIA with vision**, **Google Calendar native answer**, **Outbound Messaging REST API**, **`_capture_conditional_` trigger**. See `references/learned-patterns.md` for the practical patterns these unlock.

## CORE PHILOSOPHY

v2.x = v1 + Safety. Same tools, same BBC API, but with **mandatory patterns** that prevent
the silent failures, accidental deletions, and unvalidated deploys that plagued v1.

**Three pillars:**

1. **VERIFY** — After every mutating operation, call the corresponding `list_*` to confirm
2. **GATE** — Before every destructive operation, show impact and require explicit "yes"
3. **RECOVER** — When something fails, diagnose why and propose a fix, never just stop

---

## TOOL REFERENCE (Quick)

| Tool | Purpose | Mutating? |
| --- | --- | --- |
| `builderbot_list_projects` | List all projects | No |
| `builderbot_create_project` | Create project | Yes |
| `builderbot_list_flows` | List flows in project | No |
| `builderbot_create_flow` | Create flow with keywords | Yes |
| `builderbot_update_flow` | Update flow config | Yes |
| `builderbot_delete_flow` | Delete flow + all answers | Yes (DESTRUCTIVE) |
| `builderbot_list_answers` | List answers in flow | No |
| `builderbot_create_answer` | Add answer to flow | Yes |
| `builderbot_update_answer` | Update answer content | Yes |
| `builderbot_delete_answer` | Delete single answer | Yes (DESTRUCTIVE) |
| `builderbot_validate_bot` | Health check before deploy | No |
| `builderbot_deploy` | Deploy/status/QR/reboot/delete | Yes (some actions) |

---

## MANDATORY PATTERNS

### Pattern 1: VERIFY after every mutation

After ANY create/update/delete, call the corresponding `list_*` tool to confirm:

```
create_flow(projectId, ...) → list_flows(projectId) → search for the new flow
create_answer(...)          → list_answers(projectId, flowId) → search for new answer
update_answer(...)          → list_answers(projectId, flowId) → confirm content changed
delete_flow(...)            → list_flows(projectId) → confirm flow is gone
```

If verification fails: STOP, report the discrepancy, diagnose, and propose recovery.

### Pattern 2: GATE before destructive actions

Before `delete_flow`, `delete_answer`, or `deploy(action='delete')`:

1. Show what will be affected (flow name, answer count, references)
2. Show a clear warning: "⚠️ This cannot be undone"
3. Wait for explicit user confirmation
4. After execution, VERIFY the deletion
5. Run `validate_bot` to check for broken references

### Pattern 3: RECOVER on failure

When any operation fails or verification shows a discrepancy:

1. **Diagnose**: Check limits (flow count), conflicts (duplicate keywords), permissions
2. **Report**: Tell user exactly what went wrong
3. **Propose**: Offer concrete fix (delete unused flow, rename keyword, retry)
4. **Execute**: Fix with user approval, then VERIFY

---

## FLOW CREATION RULES

### create_flow parameters

```
projectId:        UUID (from list_projects)
name:             "Human Readable Name" (Title Case, user's language)
label:            "snake_case_slug" (lowercase, no spaces)
keywords:         ["keyword1", "keyword2"] or ["EVENTS.WELCOME"]
listenKeywords:   false (ALWAYS false unless EVENTS.VOICE_NOTE)
transcribeAudio:  true only for EVENTS.VOICE_NOTE
interpretImage:   true only for EVENTS.MEDIA
analyzeDocument:  true only for EVENTS.DOCUMENT
```

### CRITICAL RULES

* `listenKeywords` MUST be `false` for ALL flows EXCEPT `EVENTS.VOICE_NOTE`
* A new flow has 0 answers — the bot will NOT respond until you add answers
* Always add answers IMMEDIATELY after creating a flow
* Text keywords must be UNIQUE across all flows
* **The literal system event `EVENTS.ACTION` (and every other `EVENTS.*`) can appear in only ONE flow per project.** BBC routes the event to exactly one destination; duplicates cause silent conflicts where only one flow fires and the other is ignored.
* **However, you can have UNLIMITED "action-style" flows** — flows triggered by `add_chatpdf` routing rules (see [Routing from add_chatpdf rules](#routing-from-add_chatpdf-rules)). Those flows use custom labels like `_save_order_`, `_save_payment_`, `_book_appointment_`, NOT the literal `EVENTS.ACTION`.

### System Events Reference

| Event | Use case | Required flags |
| --- | --- | --- |
| `EVENTS.WELCOME` | Catch-all / fallback | listenKeywords: false |
| `EVENTS.VOICE_NOTE` | Voice message handler | listenKeywords: true, transcribeAudio: true |
| `EVENTS.MEDIA` | Image handler (vision) | interpretImage: true (see vision pattern below) |
| `EVENTS.DOCUMENT` | Document handler | analyzeDocument: true |
| `EVENTS.LOCATION` | Location shared | — |
| `EVENTS.ACTION` | Single generic "action" sink | unique per project |

> **Note:** `EVENTS.ACTION` is rarely the right tool. For multi-action bots, prefer **custom-label flows triggered from `add_chatpdf` rules**, not `EVENTS.ACTION`. See the routing section below.

---

## ANSWER TYPES REFERENCE

### add_text — Simple text message

```
type: "add_text"
message: "Your text here"  ← MAX 160 chars for WhatsApp
```

### add_chatpdf — AI-powered assistant (Pattern 2, DEFAULT)

When creating, pass an empty `message` and the assistant config inside `plugins.openai`:

```
type: "add_chatpdf"
message: ""
plugins: {
  openai: {
    assistantName: "Nora",
    assistantInstructions: "You are a helpful assistant for [business]...",
    assistantInterpretMultimedia: true,   // optional, enables vision on EVENTS.MEDIA flows
    rules: [                              // optional, AI-response → flow routing (see below)
      {
        conditionRule: "PEDIDO CONFIRMADO",
        conditionFlowId: "<save-order-flow-uuid>",
        condition: "contains",
        conditionValue: ""
      }
    ]
  }
}
```

**Key facts:**

* `message: ""` is REQUIRED — text content lives nowhere for AI answers; the assistant is the response engine.
* `plugins.openai.assistantInstructions` is the full system prompt. Treat it like a normal LLM system prompt.
* `plugins.openai.rules` is what makes the AI "agentic" — it routes to other flows based on what the AI's reply contains. This is the foundation of multi-flow bots. See [Routing from add_chatpdf rules](#routing-from-add_chatpdf-rules).
* When recreating a deleted assistant, the platform assigns a fresh `assistant_id` server-side — that's normal, not an error.

**CRITICAL — single-answer rule:** NEVER mix `add_chatpdf` with `add_text` (or any other answer type) in the same flow. Both fire on every message → broken double-response loop. A flow that contains an `add_chatpdf` must contain ONLY that one answer.

### add_image / add_video / add_doc / add_voice_note — Media

```
type: "add_image"
message: ""
options: { media: { url: "https://public-url.com/image.jpg" }, gotoFlow: {} }
```

### add_http — HTTP webhook/API call

```
type: "add_http"
message: ""
plugins: {
  http: {
    url: "https://api.example.com/endpoint",
    method: "GET" | "POST",
    headers: { "Content-Type": "application/json" },  // optional
    body: { key: "value" },  // optional, for POST; can reference {aiResponse}, {from}, {time}
    rules: [                  // optional, route based on response payload
      { conditionRule: "open", conditionFlowId: "<closed-flow-uuid>", condition: "equals", conditionValue: "false" }
    ]
  }
}
```

**CRITICAL**: `plugins.http.rules` is REQUIRED to be present (even as `[]`). Omitting it causes backend rejection.

### add_mute — Mute contact (human handoff)

```
type: "add_mute"
plugins: { mute: { status: true, gapTime: 60 } }
options: { gotoFlow: { flowId: "farewell-flow-uuid" } }
```

Mutes the INDIVIDUAL contact, not the whole bot. `gapTime` = minutes until auto-unmute.

### add_intent — AI semantic routing (lighter-weight than add_chatpdf rules)

```
type: "add_intent"
plugins: {
  intent: {
    rules: [{
      conditionRule: "The user wants to talk to a human",
      conditionFlowId: "<target-flow-uuid>",
      condition: "",
      conditionValue: ""
    }]
  }
}
```

### add_google_calendar — Native appointment scheduling

BBC Cloud has a first-class Google Calendar answer that checks availability in real time and routes conditionally — no Apps Script bridge needed for the calendar lookup itself.

```
type: "add_google_calendar"
plugins: {
  calendar: {
    calendarId: "your-calendar-id@group.calendar.google.com",
    durationMinutes: 60,
    rules: {
      available:    { flowId: "<confirm-appointment-flow-uuid>" },
      unavailable:  { flowId: "<next-available-slot-flow-uuid>" },
      missingInfo:  { flowId: "<ask-for-date-flow-uuid>" }
    }
  }
}
```

Use case: the AI captures a desired date/time → this answer queries the calendar → routes to one of three follow-up flows depending on the outcome.

### Capture rule

`options.capture = true` → bot waits for user reply, passes it to NEXT answer.
**NEVER set capture=true on the LAST answer** of a flow — the value is silently lost.

For structured multi-field capture (e.g. name → phone → email), use the `_capture_conditional_` trigger pattern documented in `references/learned-patterns.md`.

### Redirect

`options.gotoFlow = { flowId: "<target-flow-uuid>" }` → redirect after this answer.

---

## ROUTING FROM `add_chatpdf` RULES {#routing-from-add_chatpdf-rules}

**This is the pattern that unlocks multi-action AI bots and replaces the old `EVENTS.ACTION` workaround.**

The AI in `add_chatpdf` produces a normal conversational reply. While generating that reply, BBC scans `plugins.openai.rules`: if the AI's text matches any rule, the bot **goes to that flow** after sending the reply, carrying the AI's text in `{aiResponse}`.

```
EVENTS.WELCOME flow
       │
       ▼
  add_chatpdf  ← has plugins.openai.rules: [
       │           { contains "PEDIDO CONFIRMADO" → flow "Save Order" },
       │           { contains "COMPROBANTE RECIBIDO" → flow "Save Payment" },
       │           { contains "AGENDAR CITA" → flow "Calendar Check" },
       │         ]
       │
       │ AI naturally says: "Tu pedido quedó así: ... PEDIDO CONFIRMADO ✅"
       ▼
  Match → goto "Save Order" flow
       │
       ▼
  add_http POST https://script.google.com/.../exec
           body: { pedido: "{aiResponse}", numero: "{from}" }
```

**Design principles:**

1. Teach the AI in its `assistantInstructions` to emit a specific marker phrase whenever an action should fire ("PEDIDO CONFIRMADO", "DERIVAR A SOPORTE", "COMPROBANTE RECIBIDO"). Keep markers in uppercase and uncommon enough to never collide with natural conversation.
2. Each action gets its OWN flow with a custom label (e.g. `_save_order_`, `_save_payment_`) — these labels never collide with `EVENTS.*`.
3. The destination flow typically does an `add_http` to a backend (Apps Script, your API) using `{aiResponse}` as the payload.
4. There is no hard limit on the number of rules. The Sorelle/Freddy production pattern routinely uses 3–6 rules from a single `add_chatpdf`.

**Why not just use `EVENTS.ACTION`?** Because only ONE flow can carry the literal `EVENTS.ACTION`. The moment you need two actions (save order + save payment), the second collides silently. Routing from `add_chatpdf` rules is the supported way to scale.

---

## SYSTEM VARIABLES (Available in messages, HTTP bodies, and prompts)

| Variable | Meaning | Where it works |
| --- | --- | --- |
| `{from}` | The contact's WhatsApp number | All answer types |
| `{aiResponse}` | The full text the AI just generated in the preceding `add_chatpdf` | Flows reached via `plugins.openai.rules` routing — typically as the payload of an `add_http` |
| `{name}` | Contact's WhatsApp display name (if available) | All answer types |
| `{time}` | Current server time, 24h `HH:mm` | `assistantInstructions`, message bodies |
| `{date}` | Current server date | `assistantInstructions`, message bodies |
| `{fullDate}` | Full timestamp | `assistantInstructions`, message bodies |

**Warning on time-zone correctness:** `{time}`, `{date}`, and `{fullDate}` use the BBC server's clock, not the business's local timezone. For schedule-sensitive logic (open/closed, after-hours pricing rules), prefer an `add_http` GET to an Apps Script endpoint that does `Utilities.formatDate(new Date(), "America/Santiago", ...)` and returns `{ open: true/false, message: "..." }`. See `references/learned-patterns.md` → "Time-window validation" for the full Sorelle pattern.

---

## OUTBOUND MESSAGING REST API (Bot → contact, server-initiated)

The MCP tools cover **building** bots. To send a message FROM your backend INTO an existing conversation (order updates, payment confirmations, proactive notifications), use the BBC Cloud REST API:

```
POST https://app.builderbot.cloud/api/v2/{projectId}/messages
Headers:
  Content-Type: application/json
  x-api-builderbot: <your-api-key>
Body:
{
  "messages": { "content": "Your message text here" },
  "number": "584246743207",       // E.164 without "+"
  "checkIfExists": false
}
```

Typical callers: an Apps Script web app receiving a webhook from the bot, or a separate panel sending broadcasts. This is NOT a tool in this MCP — it's a plain HTTP call your backend makes.

> **Documented gap:** BBC Cloud has NO managed blacklist REST endpoint comparable to `x-api-builderbot` for adding/removing numbers. The blacklist API exposed in BuilderBot docs (`bot.blacklist.add/remove`) belongs to the self-hosted `@builderbot/bot` framework. If you need blacklist for a Cloud project, manage it inside the panel manually or maintain your own list in Apps Script and pre-filter before sending.

---

## DECISION TREE: Which Pattern to Use

```
User request → What type of business?
├── Needs appointments/citas/availability → Pattern 2 (AI) + add_google_calendar ★ RECOMMENDED
├── Needs Q&A about products/services    → Pattern 2 (AI)
├── Needs order tracking/status          → Pattern 2 (AI) + HTTP
├── Multi-step transactional (order + pay + confirm) → Pattern 2 + chatpdf rules + multiple action flows
├── Simple info (hours, location, menu)  → Pattern 1 (keyword-based) OK
├── Lead generation / qualification      → Pattern 2 (AI) + _capture_conditional_
└── Human handoff / escalation           → Pattern 2 (AI) + add_mute
```

**DEFAULT**: Pattern 2 (AI-powered with `add_chatpdf`) for 80% of real cases.
Only use pure keyword → text flows for very simple, static information.

---

## PATTERN TEMPLATES BY VERTICAL

### Salon / Beauty / Services

* WELCOME: `add_chatpdf` with services, prices, hours, booking guidance
* `plugins.openai.rules`: marker "AGENDAR CITA" → calendar verification flow
* Calendar flow: `add_google_calendar` with available/unavailable/missingInfo branches
* Escalation: marker "DERIVAR A AGENTE" → `add_mute` flow

### Restaurant / Food (Pizzería pattern)

* PRE-WELCOME: `add_http` GET to GAS `/horario` (returns `{open, message}` in the local TZ) — branches with `rules`
* WELCOME (only if open): `add_chatpdf` with menu, combos, delivery info
* `plugins.openai.rules`: markers "PEDIDO CONFIRMADO" → save-order flow; "COMPROBANTE RECIBIDO" → save-payment flow
* Image flow: `EVENTS.MEDIA` with `interpretImage: true` + `assistantInterpretMultimedia: true` for payment-receipt OCR

### E-commerce / Store

* WELCOME: `add_chatpdf` with catalog, pricing, shipping
* Order status: marker "CONSULTAR ESTADO" → flow that does `add_http` GET to order API
* Image: vision-based product identification from a photo (catalog must be loaded in `assistantInstructions`)

### Content Creator / Digital Products

* WELCOME: `add_chatpdf` with product catalog, pricing
* Purchase: marker "GENERAR LINK DE PAGO" → flow that returns checkout URL
* Support: AI handles FAQ; marker "ESCALAR" triggers human handoff

### Forum / Event

* WELCOME: `add_chatpdf` with event details, schedule, registration
* Registration: `_capture_conditional_` for structured data + `add_http` POST to registration API

---

## COMPLETE WORKFLOW: Building a Bot

### Step 1: Get or Create Project

```
list_projects()
├── Project exists → use its projectId
└── Need new → create_project(name) → list_projects() → VERIFY
```

### Step 2: Plan Flows

Before creating anything, plan all flows and share the plan with user:

```
📋 Bot Plan: [Business Name]
├── welcome (EVENTS.WELCOME) — AI assistant with business context + N routing rules
├── _action_save_order_  — triggered by AI marker "PEDIDO CONFIRMADO"
├── _action_save_payment_ — triggered by AI marker "COMPROBANTE RECIBIDO"
├── escalation (keyword: "agente", "humano") — human handoff
└── Total: N flows
```

### Step 3: Create Flows (one at a time)

For EACH flow:

```
1. create_flow(projectId, name, label, keywords, ...)
2. list_flows(projectId) → VERIFY flow exists, get flow_id
3. create_answer(projectId, flowId, type, message, ...)
4. list_answers(projectId, flowId) → VERIFY answer exists
5. [If add_chatpdf] update_answer with plugins.openai.{assistantName, assistantInstructions, rules}
6. list_answers(projectId, flowId) → VERIFY instructions and rules set
```

**NEVER create all flows first then add answers. Create flow → add answers → verify → next flow.**

**For `add_chatpdf` rules:** the rules reference `conditionFlowId` of OTHER flows. Build the destination flows FIRST so you have their UUIDs, then add the rules to the chatpdf last via `update_answer`.

### Step 4: Validate

```
validate_bot(projectId)
├── criticalCount = 0 → Ready to deploy
└── criticalCount > 0 → Fix issues first
    ├── Dead ends → add answers to empty flows
    ├── Long messages → shorten to ≤160 chars
    ├── Dangling capture → remove capture from last answer
    └── Missing WELCOME → create welcome flow
```

### Step 5: Deploy (with GATE)

```
1. validate_bot(projectId) → MUST pass (criticalCount = 0)
2. Show deployment summary to user:
   ┌─────────────────────────────┐
   │ 📤 DEPLOY SUMMARY          │
   │ Project: [name]            │
   │ Flows: N                    │
   │ Answers: M                  │
   │ Validation: ✅ PASS        │
   │ Confirm? [yes/no]          │
   └─────────────────────────────┘
3. Wait for user confirmation
4. deploy(projectId, action: "create")
5. deploy(projectId, action: "status") → check status
6. If READY_TO_SCAN → deploy(projectId, action: "qr") → show QR
```

---

## DEPLOY STATUS DIAGNOSTICS

`deploy(projectId, action: "status")` returns one of:

| Status | Meaning | Action |
| --- | --- | --- |
| `CONNECTED` | Bot is live and receiving messages | Nothing — this is "prendido" ✅ |
| `READY_TO_SCAN` | Deployed, waiting for WhatsApp QR scan | Call `deploy(action: "qr")` and present the QR |
| `INITIALIZATION` | Spinning up | Wait and re-check status in ~30s |
| `FAILED` | Deploy crashed | Inspect logs in panel; redeploy or contact support |

**Auditing "how many bots are on?":** list all projects, then call `deploy(action: "status")` for each. Only `CONNECTED` counts as live.

---

## ERROR HANDLING FRAMEWORK

### Common Failures & Recovery

| Error | Diagnosis | Recovery |
| --- | --- | --- |
| Flow created but not in list | Silent failure / API lag | Retry create, verify again |
| "Limit reached" | 50/50 flows used | List flows, find unused, offer to delete |
| Duplicate keyword | Keyword conflict | list_flows to find conflict, rename to a custom label |
| Two flows with `EVENTS.ACTION` | Event collision — only one fires | Change one to a custom label `_action_xxx_`, route to it via add_chatpdf rule |
| Answer > 160 chars | WhatsApp truncation | Shorten message, verify |
| Deploy fails | Validation issues | Run validate_bot, fix all criticals |
| add_chatpdf + add_text in same flow | Double-response bug | Delete the add_text answer |
| capture=true on last answer | Dangling capture | Remove capture or add follow-up answer |
| Flow exists but has 0 answers | Assistant was deleted (panel mishap) | Recreate the `add_chatpdf` with full `plugins.openai.assistantInstructions` |
| `BuilderBot API key required` | MCP connector lost its key | User reconnects BBC MCP TOOL in Claude → Settings → Connectors |

### Batch Operations: Sequential with Verification

When creating multiple flows, ALWAYS do them sequentially:

```
FOR EACH flow in plan:
  1. create_flow → VERIFY
  2. create_answer(s) → VERIFY each
  3. [configure AI if needed] → VERIFY
  THEN next flow
```

NEVER batch-create all flows then batch-add answers. This hides partial failures.

---

## FINAL REPORT FORMAT

After completing bot creation, always provide:

```
📊 BOT REPORT: [Project Name]
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Project ID: [uuid]

✅ Flows Created:
  1. [flow_name] (id: [uuid]) — [N] answers — [type summary]
  2. ...

🔀 add_chatpdf routing rules:
  • "MARKER_1" → [destination_flow_name]
  • "MARKER_2" → [destination_flow_name]

⚠️ Issues Fixed:
  • [description of any issues encountered and resolved]

🔍 Validation: [PASS/FAIL]
  • Criticals: 0
  • Warnings: [N]

📤 Deploy Status: [CONNECTED / READY_TO_SCAN / INITIALIZATION / FAILED / NOT YET]

🔗 Tool Call Trace:
  [list of all tool calls made in order]
```

---

## ANTI-PATTERNS (Never Do These)

1. ❌ Creating a flow without immediately adding answers
2. ❌ Reporting success without verification (`list_*`)
3. ❌ Deleting without showing impact and getting confirmation
4. ❌ Deploying without running `validate_bot` first
5. ❌ Mixing `add_chatpdf` and `add_text` in the same flow
6. ❌ Setting `listenKeywords: true` on non-VOICE_NOTE flows
7. ❌ Setting `capture: true` on the last answer of a flow
8. ❌ Batch-creating all flows before adding answers
9. ❌ Ignoring validation warnings about message length (>160 chars)
10. ❌ Creating `add_http` answers without `plugins.http.rules` (even `[]` is fine)
11. ❌ Putting `EVENTS.ACTION` on more than one flow — use custom labels routed from `add_chatpdf` rules instead
12. ❌ Trusting `{time}`/`{date}` for business-hours logic when the business isn't in the BBC server timezone — use an `add_http` to a TZ-aware backend
13. ❌ Putting assistant instructions in `message` instead of `plugins.openai.assistantInstructions`

---

## REFERENCES

For deeper patterns and worked examples, read:

* `references/verticals.md` — Detailed bot templates per business type
* `references/advanced-patterns.md` — HTTP integrations, multi-flow routing, knowledge base setup
* `references/learned-patterns.md` — **NEW in v2.1.** Production patterns learned from real deployments: chatpdf rules as router, `{aiResponse}` pipe to backend, time-window validation via TZ-aware GAS, image-as-payment-proof workflow, deleted-assistant recovery.
