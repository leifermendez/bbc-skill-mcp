# Advanced Patterns — BBC MCP Tool v2.4

## HTTP Integration (add_http)

### GET Request
```
create_answer:
  type: "add_http"
  message: ""
  plugins:
    http:
      url: "https://api.example.com/status?order={capture}"
      method: "GET"
      rules: []
```

### POST Request
```
create_answer:
  type: "add_http"
  message: ""
  plugins:
    http:
      url: "https://api.example.com/webhook"
      method: "POST"
      headers:
        Content-Type: "application/json"
      body:
        name: "{capture}"
        source: "whatsapp"
      rules: []
```

### Capture → HTTP Pattern
Common for collecting user data then sending to API:
```
Flow: registration
  Answer 1: add_text "¿Cuál es tu nombre?" + capture: true
  Answer 2: add_text "¿Tu email?" + capture: true
  Answer 3: add_http → POST captured data to CRM
```

**IMPORTANT**: Each capture only passes to the NEXT answer. Never set `capture: true` on the last answer of a flow. For multi-field capture, chain answers or use AI (`add_chatpdf`) to collect all fields and route via top-level `rules`.

---

## Multi-Flow Routing

### Using gotoFlow
Redirect user from one flow to another after an answer (`options.gotoFlow` is a **UUID string**, never `{ flowId }`):
```
update_answer:
  options:
    gotoFlow: "<target-flow-uuid>"
```
Pass `null` to clear an existing redirect.

### Using add_intent (AI Semantic Router)
Route based on user intent without exact keyword matching (`condition` must be a valid operator from the closed enum such as `"==="`, never `""`):
```
create_answer:
  type: "add_intent"
  message: ""
  plugins:
    intent:
      rules:
        - conditionRule: "The user wants to book an appointment"
          conditionFlowId: "<booking-flow-uuid>"
          condition: "==="
          conditionValue: "book"
        - conditionRule: "The user wants to talk to a human"
          conditionFlowId: "<escalation-flow-uuid>"
          condition: "==="
          conditionValue: "human"
        - conditionRule: "The user wants to know prices"
          conditionFlowId: "<pricing-flow-uuid>"
          condition: "==="
          conditionValue: "prices"
```

### Using Google Calendar (add_google_calendar)
Configure `plugins.scheduleGoogleCalendar` and route outcomes (`available | not_available | misunderstood`) via top-level `rules`:
```
create_answer:
  type: "add_google_calendar"
  message: ""
  plugins:
    scheduleGoogleCalendar:
      action: "create"
      calendarId: "primary"
      durationMeeting: 30
  rules:
    - conditionRule: "available"
      condition: "==="
      conditionValue: ""
      conditionFlowId: "<confirm-flow-uuid>"
    - conditionRule: "not_available"
      condition: "==="
      conditionValue: ""
      conditionFlowId: "<busy-flow-uuid>"
    - conditionRule: "misunderstood"
      condition: "==="
      conditionValue: ""
      conditionFlowId: "<retry-flow-uuid>"
```

---

## Knowledge Base Management

### Creating an AI Assistant Answer (One Call)
`add_chatpdf` is provisioned in a single `create_answer` call:
```
create_answer:
  projectId: "<project-uuid>"
  flowId: "<flow-uuid>"
  type: "add_chatpdf"
  message: ""
  plugins:
    openai:
      assistantModel: "gpt-5.4-nano"
      assistantInstructions: "You are the assistant for [business]..."
  rules: []
```

### Scraping a Website
After creating an `add_chatpdf` answer, feed it website content:
```
update_answer:
  answerId: "<answer-uuid>"
  flowId: "<flow-uuid>"
  projectId: "<project-uuid>"
  assistant:
    scrapeUrl: "https://www.business-website.com"
```

### Updating AI Instructions or Model
```
update_answer:
  answerId: "<answer-uuid>"
  flowId: "<flow-uuid>"
  projectId: "<project-uuid>"
  plugins:
    openai:
      assistantInstructions: "You are the assistant for [business]..."
      assistantModel: "gpt-5.4-nano"
```

### Listing Knowledge Base Files
```
update_answer:
  answerId: "<answer-uuid>"
  flowId: "<flow-uuid>"
  projectId: "<project-uuid>"
  assistant:
    listFiles: true
```

### Deleting a File from Knowledge Base
```
update_answer:
  answerId: "<answer-uuid>"
  flowId: "<flow-uuid>"
  projectId: "<project-uuid>"
  assistant:
    deleteFileId: "<file-uuid>"
```

---

## Human Handoff Pattern

### Basic Handoff
```
Flow: escalation (keywords: "agente", "humano", "persona")
  Answer 1: add_text "Te comunico con un agente. Espera un momento."
  Answer 2: add_mute { status: true, gapTime: 60 }
```

### Handoff with Context (Advanced)
```
Flow: escalation
  Answer 1: add_text "Transfiriendo a un agente..."
  Answer 2: add_http → POST conversation context to agent dashboard
  Answer 3: add_mute { status: true, gapTime: 120 }
```

---

## Media Answers

`options.media` is always a **public URL string** (never `{ url }`), and `options.gotoFlow` is an optional **flow UUID string** (or `null` / omitted, never `{}`).

### Sending an Image
```
create_answer:
  type: "add_image"
  message: "Nuestro menú actualizado"
  options:
    media: "https://example.com/menu.jpg"
```

### Sending a Document (PDF, etc.)
```
create_answer:
  type: "add_doc"
  message: "Catálogo completo"
  options:
    media: "https://example.com/catalog.pdf"
```

### Sending a Video
```
create_answer:
  type: "add_video"
  message: "Tutorial paso a paso"
  options:
    media: "https://example.com/tutorial.mp4"
```

**IMPORTANT**: Media URLs must be publicly accessible. Private/authenticated URLs will fail.

---

## Deployment Lifecycle

### Full Deploy Sequence
```
1. builderbot_validate_bot(projectId)           → Must pass (criticalCount = 0)
2. builderbot_deploy(projectId, action: "create")  → Provision container
3. builderbot_deploy(projectId, action: "status")  → Wait for READY_TO_SCAN
4. builderbot_deploy(projectId, action: "qr")      → Get QR code image (only in READY_TO_SCAN)
5. User scans QR with WhatsApp
6. builderbot_deploy(projectId, action: "status")  → Shows ONLINE
```

### Status Values
| Status | Meaning |
|--------|---------|
| `INITIALIZATION` | Container starting up (`action='qr'` returns `deploy_qr_not_ready`) |
| `READY_TO_SCAN` | QR available, waiting for scan |
| `ONLINE` | Bot is live and responding (no QR needed) |
| `FAILED` | Something went wrong — inspect with `builderbot_read_logs` |

### Reboot
```
builderbot_deploy(projectId, action: "reboot")  → Restart bot container
```

### Delete Deployment (GATE REQUIRED)
```
builderbot_deploy(projectId, action: "delete")  → Tear down container
```
Does NOT delete project data — just stops the running bot.

---

## WhatsApp Message Limits

- **Text messages**: ≤ 160 characters recommended
- **AI responses** (`add_chatpdf`): Include "keep responses under 150 characters" in instructions
- **Button labels**: ≤ 20 characters
- **List items**: ≤ 24 characters for title, ≤ 72 for description

Always run `builderbot_validate_bot` before deploying.
