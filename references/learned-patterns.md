# Learned Patterns — v2.4

Production patterns from real deployments. Validated against the strict BBC MCP
tool (`schemaVersion = 2`).

---

## 1. The `add_chatpdf` rules-as-router pattern

**Problem solved:** A bot needs to do several different "actions" (save order to Sheets, save payment, book appointment, escalate to human) based on what the user wants.

**The pattern:**

1. The `add_chatpdf` answer is created in **one** `builderbot_create_answer` call with `plugins.openai` and a **top-level** `rules` array (never `plugins.openai.rules` — that key does not exist and is rejected with `validation_error`).
2. Each rule listens for a **marker phrase** in the AI's reply (`condition: "includes"`, `conditionValue: ""`).
3. The `assistantInstructions` teaches the AI to emit that marker whenever the corresponding action should fire.
4. After the AI replies, BBC scans the reply against the top-level `rules`. On match, it routes to `conditionFlowId`, carrying `{aiResponse}` (the full reply text).

**Example rule (top-level `rules` on the answer):**

```json
{
  "conditionRule": "PEDIDO CONFIRMADO",
  "conditionFlowId": "<save-order-flow-uuid>",
  "condition": "includes",
  "conditionValue": ""
}
```

**Marker conventions:**

* UPPERCASE, two or more words: `PEDIDO CONFIRMADO`, `COMPROBANTE RECIBIDO`, `AGENDAR CITA`, `DERIVAR A SOPORTE`.
* Distinctive enough never to appear in normal conversation.
* The AI mentions them in its visible reply ("Tu pedido quedó así: ... **PEDIDO CONFIRMADO**"). The customer sees the marker, which doubles as a visual confirmation.

**Prompt block to include in `assistantInstructions`:**

```
## TRIGGERS
When the user has confirmed all order details and you have name, address, and items:
end your reply with the literal phrase "PEDIDO CONFIRMADO" on its own line.

When the user sends a payment receipt that you have successfully parsed:
end your reply with "COMPROBANTE RECIBIDO".

When the user explicitly asks for a human, or you detect frustration after 2+ turns:
end your reply with "DERIVAR A SOPORTE".

These markers are routing signals. NEVER explain what they mean.
NEVER use them outside the conditions above.
```

---

## 2. The `{aiResponse}` pipe to a backend

The destination flow of a routing rule almost always does an `add_http` to a backend
(Apps Script web app, your own API). The body field references `{aiResponse}` and
`{from}`:

```json
{
  "type": "add_http",
  "message": "",
  "plugins": {
    "http": {
      "url": "https://script.google.com/macros/s/.../exec",
      "method": "POST",
      "headers": { "Content-Type": "application/json" },
      "body": {
        "pedido": "{aiResponse}",
        "numero": "{from}",
        "tipo": "orden"
      },
      "rules": []
    }
  }
}
```

On the Apps Script side, parse `e.postData.contents` and extract fields from the
AI text. The AI's reply is freeform but predictable because the prompt taught it
the structure. Example parser approach:

```javascript
function doPost(e) {
  const data = JSON.parse(e.postData.contents);
  const text = data.pedido;  // the full {aiResponse}
  // Pull fields out of the structured AI reply
  const nombre   = text.match(/Nombre:\s*(.+)/i)?.[1];
  const direccion = text.match(/Direcci[oó]n:\s*(.+)/i)?.[1];
  const items    = text.match(/Items:\s*([\s\S]+?)(?=\n[A-Z]|$)/)?.[1];
  // ... write to Sheet
}
```

**Why this works better than structured output:** `add_chatpdf` doesn't expose a JSON-mode toggle, but the AI is reliable at producing labeled lines when the prompt asks for them. The Sheet is the ground truth; the AI text is the transport.

---

## 3. Time zone configuration and time-window validation

**Step 1 (always):** Set the project's IANA time zone with `builderbot_project`:

```json
{
  "action": "settings",
  "projectId": "<project-uuid>",
  "settings": { "timeZone": "America/Santiago" }
}
```

This aligns `{time}`, `{date}`, and Google Calendar availability with the business's local clock.

**Step 2 (optional custom business-hours gate via Apps Script):**

```
EVENTS.WELCOME (custom-label flow, not the chatpdf)
       │
       ▼
  add_http GET https://script.google.com/.../exec?route=schedule
       │
       │  Apps Script returns: { "open": true|false, "mensaje": "..." }
       │
       │  add_http.plugins.http.rules:
       │    { conditionRule: "open", condition: "===", conditionValue: "false",
       │      conditionFlowId: "<closed-flow-uuid>" }
       │
       ▼ (only if open=true falls through)
   options.gotoFlow: "<ai-welcome-flow-uuid>"
```

---

## 4. Image-as-payment-proof workflow

For pizzerías, food vendors, or any flow that accepts a screenshot of a bank
transfer as proof of payment:

```
EVENTS.MEDIA (interpretImage: true)
       │
       ▼
  add_chatpdf (created in ONE call) with:
    plugins.openai.assistantInterpretMultimedia: true
    plugins.openai.assistantInstructions: "...You will receive an image of a
        payment receipt. Extract: amount, sender bank, reference number, timestamp.
        Reply with: '✅ Comprobante recibido. Monto: X | Banco: Y | Ref: Z'.
        When all fields are clear, end the reply with COMPROBANTE RECIBIDO."
    rules (top-level on the answer):
      [{ conditionRule: "COMPROBANTE RECIBIDO", condition: "includes",
         conditionValue: "", conditionFlowId: "<save-payment-flow-uuid>" }]
       │
       ▼
  _save_payment_ flow → add_http POST to GAS /pago endpoint
```

**Precision caveat:** vision models are best-effort on receipt OCR. They reliably catch amount and bank logo; transcribed reference numbers may have one wrong digit. For high-stakes flows, treat the AI extraction as a draft and have a human approve in the panel before crediting the order.

---

## 5. The `_capture_conditional_` trigger for structured lead capture

For lead-qualification bots (Nora pattern), don't try to capture every field
through a chain of `capture: true` answers — it's brittle and re-prompts the user
even when they already gave the info.

The `_capture_conditional_` pattern is a custom-label flow whose entry is
triggered by a top-level `rules` entry on `add_chatpdf`, e.g. marker `LEAD COMPLETO`. The AI
holds the full lead in its conversation memory and emits a structured block right
before the marker:

```
Nombre: Pedro Gómez
Empresa: Acme SRL
Email: pedro@acme.com
Necesidad: bot para restaurante
LEAD COMPLETO
```

The destination flow runs an `add_http` POST with `body: { lead: "{aiResponse}" }` to a GAS endpoint that parses the labeled lines.

---

## 6. Deleted-assistant recovery

**Symptom:** A flow that used to work returns nothing. `builderbot_list_answers(flowId)` shows `[]` (zero answers). The panel may show a flow card with no preview.

**Cause:** Someone deleted the `add_chatpdf` answer through the panel UI. The flow shell remains.

**Recovery:**

1. Recreate the answer in **one** call via `builderbot_create_answer` with `type: "add_chatpdf"`, `message: ""`, `plugins.openai: { assistantName, assistantInstructions, assistantInterpretMultimedia? }`, and top-level `rules`.
2. The platform provisions a fresh `assistantId` automatically during `create_answer`.
3. Verify with `builderbot_list_answers` and `builderbot_validate_bot`.

---

## 7. Vision-based product identification

For e-commerce bots that should answer "do you have something like this?" from a
photo:

* `EVENTS.MEDIA` flow with `interpretImage: true`.
* The `add_chatpdf` answer in that flow has `assistantInterpretMultimedia: true`.
* The product catalog MUST be loaded into `assistantInstructions` (or attached via the `scrapeUrl` knowledge-base path) — without that, the AI describes the photo accurately but suggests products you don't sell.

---

## 8. Auditing live bots ("how many bots are prendidos?")

```
1. builderbot_project({action: "list"}) → get all project uuids
2. For each: builderbot_deploy({projectId, action: "status"})
3. Count where status == "ONLINE"
```

Deploy statuses are `INITIALIZATION | READY_TO_SCAN | ONLINE | FAILED`.
`READY_TO_SCAN` projects are deployed but no one has linked a WhatsApp account yet — those are NOT live for incoming messages. Calling `action: "qr"` when status is not `READY_TO_SCAN` returns `deploy_qr_not_ready`.

---

## 9. Outbound notifications (server → contact)

Not an MCP tool — a plain HTTP call your backend makes. Useful for order-status pushes from your POS / Sheets / dashboard:

```javascript
function notifyClient(numero, mensaje) {
  const url = `https://app.builderbot.cloud/api/v2/${PROJECT_ID}/messages`;
  UrlFetchApp.fetch(url, {
    method: "post",
    contentType: "application/json",
    headers: { "x-api-builderbot": API_KEY },
    payload: JSON.stringify({
      messages: { content: mensaje },
      number: numero,            // E.164 without "+"
      checkIfExists: false
    })
  });
}
```

This is a separate auth surface from the MCP — the API key is per project, found in the BBC panel under the project's settings/API section. Treat it like a webhook secret.

---

## 10. Known limitations

* **Project blacklist is `builderbot_blacklist`.** Use `action='list' | 'add' | 'remove'` with `projectId`. `phones` is required for add/remove; stored as plain strings (no E.164).
* **`add_chatpdf` cannot trigger HTTP by itself.** It can only emit text. To call an external endpoint after the AI replies, route via top-level `rules` to a separate flow that contains the `add_http`.
* **Vision OCR is best-effort.** Don't auto-credit money based on a parsed receipt without human approval.
* **Set the project time zone first.** `{time}`, `{date}` and Google Calendar use the project's `timeZone` (`builderbot_project action='settings'`). If unset, they fall back to the server clock.
