# Learned Patterns — v2.1

Production patterns from real deployments. None of these are documented in the BBC
official docs or the upstream v2.0 skill; they were reverse-engineered from working
projects (Freddy, Sorelle, Nora) and validated against the BBC MCP tool.

---

## 1. The `add_chatpdf` rules-as-router pattern

**Problem solved:** A bot needs to do several different "actions" (save order to Sheets, save payment, book appointment, escalate to human) based on what the user wants. The naive approach — putting `EVENTS.ACTION` on every action flow — fails because only ONE flow per project can carry the literal `EVENTS.ACTION`.

**The pattern:**

1. The `add_chatpdf` answer holds `plugins.openai.rules`, an array of `{ conditionRule, conditionFlowId, condition, conditionValue }`.
2. Each rule listens for a **marker phrase** in the AI's reply.
3. The `assistantInstructions` teaches the AI to emit that marker whenever the corresponding action should fire.
4. After the AI replies, BBC scans the reply against the rules. On match, it goes to `conditionFlowId`, carrying `{aiResponse}` (the full reply text).

**Example rule:**

```json
{
  "conditionRule": "PEDIDO CONFIRMADO",
  "conditionFlowId": "<save-order-flow-uuid>",
  "condition": "contains",
  "conditionValue": ""
}
```

**Marker conventions:**

* UPPERCASE, two or more words: `PEDIDO CONFIRMADO`, `COMPROBANTE RECIBIDO`, `AGENDAR CITA`, `DERIVAR A SOPORTE`.
* Distinctive enough never to appear in normal conversation.
* The AI mentions them in its visible reply ("Tu pedido quedó así: ... **PEDIDO CONFIRMADO**"). The customer sees the marker, which is fine and even desirable — it doubles as a visual confirmation.

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

## 3. Time-window validation via TZ-aware GAS (Sorelle pattern)

**Problem solved:** `{time}` and `{date}` use the BBC server's clock, which may not match the business's local timezone. Business-hours logic that uses these variables breaks silently on DST changes or server moves.

**The pattern:** Put a TZ-aware Apps Script web app in front of the WELCOME flow.

```
EVENTS.WELCOME (custom-label flow, not the chatpdf)
       │
       ▼
  add_http GET https://script.google.com/.../exec?route=schedule
       │
       │  Apps Script returns: { "open": true|false, "mensaje": "..." }
       │
       │  add_http.plugins.http.rules:
       │    { conditionRule: "open", condition: "equals", conditionValue: "false",
       │      conditionFlowId: "<closed-flow-uuid>" }
       │
       ▼ (only if open=true falls through)
   gotoFlow → "AI Welcome" flow with the add_chatpdf
```

Apps Script side:

```javascript
function doGet(e) {
  const tz = "America/Santiago";
  const now = new Date();
  const day  = Utilities.formatDate(now, tz, "EEEE").toLowerCase();
  const hhmm = Utilities.formatDate(now, tz, "HH:mm");
  const open = isOpen(day, hhmm);  // your business-hours table
  return ContentService
    .createTextOutput(JSON.stringify({ open, mensaje: open ? "" : "Estamos cerrados. Horarios: ..." }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

**The AI never sees a message sent outside business hours.** The closed-flow ends the conversation cleanly with the right hours. No prompt-engineering of `{time}` needed.

---

## 4. Image-as-payment-proof workflow

For pizzerías, food vendors, or any flow that accepts a screenshot of a bank
transfer as proof of payment:

```
EVENTS.MEDIA (interpretImage: true)
       │
       ▼
  add_chatpdf with:
    plugins.openai.assistantInterpretMultimedia: true
    plugins.openai.assistantInstructions: "...You will receive an image of a
        payment receipt. Extract: amount, sender bank, reference number, timestamp.
        Reply with: '✅ Comprobante recibido. Monto: X | Banco: Y | Ref: Z'.
        When all fields are clear, end the reply with COMPROBANTE RECIBIDO."
    plugins.openai.rules:
      { conditionRule: "COMPROBANTE RECIBIDO" → flow "_save_payment_" }
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
triggered by an `add_chatpdf` routing rule, e.g. marker `LEAD COMPLETO`. The AI
holds the full lead in its conversation memory and emits a structured block right
before the marker:

```
Nombre: Pedro Gómez
Empresa: Acme SRL
Email: pedro@acme.com
Necesidad: bot para restaurante
LEAD COMPLETO
```

The destination flow runs an `add_http` POST with `body: { lead: "{aiResponse}" }` to a GAS endpoint that parses the labeled lines. The AI gates emission on actually having all fields, which it does much more reliably than a rigid capture chain.

---

## 6. Deleted-assistant recovery

**Symptom:** A flow that used to work returns nothing. `builderbot_list_answers(flowId)` shows `[]` (zero answers). The panel may show a flow card with no preview.

**Cause:** Someone deleted the `add_chatpdf` answer through the panel UI. The flow shell remains.

**Recovery:**

1. Recreate the answer via `builderbot_create_answer` with `type: "add_chatpdf"` and `message: ""`.
2. Immediately follow up with `builderbot_update_answer` carrying the full `plugins.openai.{assistantName, assistantInstructions, assistantInterpretMultimedia?, rules}`.
3. The platform issues a fresh `assistant_id` — that's expected, not a bug. Old chat history in the panel will reference the old ID; new conversations use the new one.
4. Verify with `builderbot_list_answers` and `builderbot_validate_bot`.

---

## 7. Vision-based product identification

For e-commerce bots that should answer "do you have something like this?" from a
photo:

* `EVENTS.MEDIA` flow with `interpretImage: true`.
* The `add_chatpdf` answer in that flow has `assistantInterpretMultimedia: true`.
* The product catalog MUST be loaded into `assistantInstructions` (or attached via the `scrapeUrl` knowledge-base path) — without that, the AI describes the photo accurately but suggests products you don't sell.

Precision is best-effort per category and product family, not per SKU. Acceptable for "show me alternatives", risky for "is this exactly the same item".

---

## 8. Auditing live bots ("how many bots are prendidos?")

```
1. builderbot_list_projects() → get all project IDs
2. For each: builderbot_deploy({projectId, action: "status"})
3. Count where status == "CONNECTED"
```

`READY_TO_SCAN` projects are deployed but no one has linked a WhatsApp account yet — those are NOT live for incoming messages.

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

* **No managed blacklist endpoint on BBC Cloud.** The `bot.blacklist.add/remove` API in BuilderBot docs is for the self-hosted framework only. Cloud projects have to manage blocked numbers manually in the panel, or you filter on your side before calling the outbound messaging API.
* **`add_chatpdf` cannot trigger HTTP by itself.** It can only emit text. To call an external endpoint after the AI replies, you MUST route via `plugins.openai.rules` to a separate flow that contains the `add_http`. There is no "tool calling" inside the chatpdf.
* **Vision OCR is best-effort.** Don't auto-credit money based on a parsed receipt without human approval.
* **`{time}`/`{date}` are BBC-server-local, not business-local.** Use a TZ-aware GAS endpoint for hours logic.
