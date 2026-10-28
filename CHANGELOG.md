# Changelog

## v2.5 — Webhook, WhatsApp templates and Flows

Adds the three tools `builderbot-mcp-http` registers for webhook and Meta assets (the server now exposes 19 tools).
**Publish after that MCP is deployed.** Before then the agent would call tools the server does not have (`unknown_tool`).

### Added

- **`builderbot_webhook`** (`get` | `set` | `disable`): event webhook URL and headers. Omitting `headers` keeps the stored
  ones, `[]` clears them; values are returned as `[hidden]`.
- **`builderbot_whatsapp_template`** (`list` | `get` | `create` | `delete`): Meta templates. `create` goes to Meta review and
  is usable only once `APPROVED`; `{{n}}` variables need `example` values; a `FLOW` button needs the `flow_id` of a published Flow.
- **`builderbot_whatsapp_flow`** (`list` | `get` | `create` | `publish` | `retire`): WhatsApp Flows. `create` uploads a DRAFT
  (`flowJson` as an object, version `7.3`); `publish` is irreversible.
- **`confirmation_required`** in the recovery table: deleting a template and publishing or retiring a Flow need `confirm: true`,
  set only after the person's explicit yes.
- Frontmatter keywords so the skill activates on webhook, plantilla de WhatsApp, HSM and WhatsApp Flow requests.
- Verify rows, gate entries and three anti-patterns for the new tools. A worked template and Flow example.

### Not changed

- Sending a template message stays REST (`builderbot_docs` returns the body); it is not an MCP tool.
- `SKILL.md` keeps the `3510` token budget. The new text lives in `SKILL.md` itself: only that file reaches the gateway.

## v2.4 — Strict MCP answer contract, project settings & runtime guards

Aligns the skill with `builderbot-mcp-http` `schemaVersion = 2`. Inputs are now
validated strictly by the MCP server: any unknown key or invalid shape returns
`code: "validation_error"` with the failing field path and an actionable hint
instead of being silently discarded.

### Fixed (Breaking silent-drop shapes in v2.1–v2.3)

- **Top-level `rules` on answers:** Routing rules for `add_chatpdf`, conditional
  capture, and calendars live in the answer's top-level `rules` array — **never**
  under `plugins.openai.rules` (rejected with `validation_error`).
- **String `options.media` and `options.gotoFlow`:** `options.media` is a public
  URL string (not `{ url }`) and `options.gotoFlow` is a destination flow UUID
  string or `null` (not `{ flowId }` or `{}`).
- **One-call `add_chatpdf` creation:** `builderbot_create_answer` with
  `type: "add_chatpdf"` and `plugins.openai` provisions the platform assistant
  in a single call. No second `update_answer` call is needed to create it.
- **Google Calendar shape:** Uses `type: "add_google_calendar"` with
  `plugins.scheduleGoogleCalendar` (`action`, `calendarId`, `durationMeeting`,
  `canDuplicate`, `maxEventsPerHour`) and top-level `rules` matching
  `available | not_available | misunderstood` (not `plugins.calendar` or
  `missingInfo`).
- **`add_intent` operator:** `plugins.intent.rules[].condition` must be a valid
  operator from the closed 14-operator enum (e.g. `"==="`), never `""`.
- **Deploy statuses:** Actual statuses are
  `INITIALIZATION | READY_TO_SCAN | ONLINE | FAILED` (not `CONNECTED`). Calling
  `builderbot_deploy action='qr'` outside `READY_TO_SCAN` returns
  `deploy_qr_not_ready`.
- **Repeatable flow events:** Both `EVENTS.ACTION` and `EVENTS.IG_COMMENT` may
  appear in multiple flows; `EVENTS.VOICE_NOTE`, `EVENTS.MEDIA`, and
  `EVENTS.DOCUMENT` remain unique per project.

### Added

- **`builderbot_project action='settings'`:** Read or update project-level
  `provider` (`meta`, `baileys`, `instagram`, `telegram`, `evolution`, `wpp`,
  `HighLevel`), `timeZone` (IANA, e.g. `America/Mexico_City`), `phoneNumber`,
  `rejectCalls`, `showTypingEvents`, and `showRecordingEvents`.
- **`validation_error` recovery rule:** Never resend an unchanged payload after
  `validation_error`; apply the field-level `hint` and retry.

## v2.3 — Project blacklist MCP tool

Aligns the skill with the current BuilderBot MCP surface (16 tools). Agents can
block spam/abuse numbers without leaving MCP.

### Added

- **`builderbot_blacklist`** (`list` | `add` | `remove`) — Cloud project
  blacklist. `phones` is required for add/remove; numbers are stored as-is
  (no E.164). VERIFY after add/remove with `action='list'`.

### Changed

- Removed the "no Cloud blacklist endpoint" limitation. That was true of the
  self-hosted `@builderbot/bot` API (`bot.blacklist.add/remove`); Cloud has
  `GET/POST/DELETE /project/:id/blacklist`, now exposed as this MCP tool.
- The `add_blacklist` **flow-node** answer type is still not an MCP tool.

## v2.2 — MCP tool unification

Aligns the skill with the BuilderBot MCP surface after project-tool unification
(then 15 tools). Agents that
still call the old project tools get `unknown_tool` and retry the same name in a
loop unless this document is updated.

### Changed

- **Projects are one tool.** `builderbot_list_projects` and
  `builderbot_create_project` no longer exist. Use `builderbot_project` with
  `action='list' | 'get' | 'create' | 'update' | 'delete' | 'duplicate'`. The
  `uuid` from `list` is `projectId` for every other tool. `delete` requires
  `confirm=true`.
- **VERIFY no longer says `list_*`.** After a project mutation, verify with
  `builderbot_project action='list'`. Flow and answer lists are unchanged
  (`builderbot_list_flows`, `builderbot_list_answers`).
- **Deploy QR is `builderbot_deploy action='qr'`.** There is no
  `builderbot_get_qr` tool.

### Added

- **`builderbot_docs`** (`search` | `get` | `list`) — official API reference.
  Honor the `coverage` field: only call the named MCP tool; `not_exposed` is
  REST-only.
- **`builderbot_sanity_check`** — runtime health after a bot is `CONNECTED`.
- **`builderbot_read_logs`** — container logs when sanity check looks unhealthy.
- **`builderbot_install_skill_bbc`** — returns a local `npx skills add` command;
  does not install files on the MCP server.
- Recovery row for `unknown_tool` / retired names.

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
  (blacklist now via `builderbot_blacklist`, chatpdf can't call HTTP directly, etc.).

### Known limitations documented (not new behavior, just newly written down)

- BBC Cloud blacklist is now `builderbot_blacklist` (added in v2.3).
- `add_chatpdf` cannot directly trigger HTTP — must route via rules.
- Vision OCR is best-effort; never auto-credit money from parsed receipts.
- `{time}`/`{date}` are BBC-server-local, not business-local.
