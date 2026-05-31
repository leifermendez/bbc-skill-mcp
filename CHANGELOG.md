# Changelog

## v2.1 — Real-world corrections (zabdielna fork)

This release patches v2.0 with corrections learned from production deployments.
None of these change tool signatures or break v2.0 bots; they're documentation
and pattern fixes that prevent bugs the upstream skill was actively causing.

### Fixed

- **`EVENTS.*` uniqueness rule was overstated.** The original line "Each EVENTS.\*
  can appear in only ONE flow" was correct only for the literal system event
  (e.g. `EVENTS.ACTION`). It led the assistant to refuse multi-action bot
  designs. v2.1 clarifies: the literal event is unique per project, but you can
  have unlimited "action-style" flows triggered by `add_chatpdf` routing rules
  via custom labels.

- **`add_chatpdf` configuration shape was wrong.** v2.0 said to configure via
  `update_answer` with a top-level `assistant` field. The real BBC API nests
  instructions under `plugins.openai.assistantInstructions` and
  `plugins.openai.assistantName`. v2.1 documents the actual shape, including
  `plugins.openai.rules` and `plugins.openai.assistantInterpretMultimedia`.

### Added

- **"Routing from add_chatpdf rules" section.** Documents the marker-phrase
  pattern that lets the AI trigger arbitrary downstream flows. This is the
  foundation of every multi-action production bot built on BBC Cloud.

- **System Variables table.** `{from}`, `{aiResponse}`, `{name}`, `{time}`,
  `{date}`, `{fullDate}` — where each works and the timezone gotcha for
  schedule-sensitive logic.

- **Outbound Messaging REST API.** The `POST /api/v2/{projectId}/messages`
  endpoint that server-side code uses to push messages into a conversation
  (order updates, broadcasts). Not an MCP tool — a plain HTTP call.

- **`add_google_calendar` answer type.** First-class appointment scheduling
  with available / unavailable / missingInfo conditional routing. Replaces the
  BBC + GAS hybrid that v2.0 implied was necessary.

- **Deploy Status Diagnostics.** Meaning of `CONNECTED`, `READY_TO_SCAN`,
  `INITIALIZATION`, `FAILED`, and how to audit "how many bots are live?"

- **EVENTS.MEDIA + vision pattern.** Receipt OCR for payment proofs,
  product identification from photos, with the multimedia flag and precision
  caveats.

- **`_capture_conditional_` trigger.** Structured lead capture via AI marker
  + GAS parser, instead of brittle multi-step `capture: true` chains.

- **New file `references/learned-patterns.md`.** Ten production patterns
  with worked examples: chatpdf-rules-as-router, `{aiResponse}` pipe to GAS,
  TZ-aware schedule validation, image-as-payment-proof, deleted-assistant
  recovery, live-bot auditing, outbound notifications, and known limitations
  (no Cloud blacklist endpoint, chatpdf can't call HTTP directly, etc.).

### Known limitations documented (not new behavior, just newly written down)

- BBC Cloud has no managed blacklist REST endpoint.
- `add_chatpdf` cannot directly trigger HTTP — must route via rules.
- Vision OCR is best-effort; never auto-credit money from parsed receipts.
- `{time}`/`{date}` are BBC-server-local, not business-local.
