# `bbc-skill-mcp` — Official BuilderBot Cloud MCP Skill (v2.5)

Agent skill for building, configuring, validating, and deploying conversational WhatsApp/Instagram/Telegram bots on **BuilderBot Cloud (BBC)** through the `builderbot-mcp-http` server (`schemaVersion = 2`).

---

## Repository Structure

| File | Purpose |
| :--- | :--- |
| [`SKILL.md`](./SKILL.md) | Primary skill definition loaded into the agent context (`name: bbc-skill-tool`, budget: `3510` tokens). |
| [`CHANGELOG.md`](./CHANGELOG.md) | Release history from v2.1 through v2.5. |
| [`references/learned-patterns.md`](./references/learned-patterns.md) | Production patterns: top-level `rules` router, `{aiResponse}` webhook pipe, IANA `timeZone` settings, vision receipt proof, lead capture. |
| [`references/advanced-patterns.md`](./references/advanced-patterns.md) | Exact payload recipes for `add_http`, `gotoFlow`, `add_intent`, `add_google_calendar`, `add_chatpdf`, media answers, and deploy lifecycle. |
| [`references/verticals.md`](./references/verticals.md) | Prompt and flow templates by business vertical (salon, restaurant, e-commerce, creators, events, lead-gen). |

---

## Key Contract Rules in v2.5 (`schemaVersion = 2`)

1. **Strict input validation (`validation_error`):** Unknown keys or wrong field types are rejected with a field path and actionable `hint` instead of being silently stripped.
2. **Top-level `rules`:** Conditional routing rules for `add_chatpdf`, conditional capture, and Google Calendar go in the answer's top-level `rules` array — never inside `plugins.openai.rules`.
3. **String `options.media` and `options.gotoFlow`:** `options.media` is a public URL string; `options.gotoFlow` is a flow UUID string (or `null`).
4. **Single-call `add_chatpdf`:** Creating an `add_chatpdf` answer with `plugins.openai` provisions the platform assistant automatically in one call.
5. **Google Calendar (`add_google_calendar`):** Uses `plugins.scheduleGoogleCalendar` (`action`, `calendarId`, `durationMeeting`) + top-level `rules` for `available | not_available | misunderstood`.
6. **Project settings (`builderbot_project action='settings'`):** Reads or updates `provider`, IANA `timeZone`, `phoneNumber`, `rejectCalls`, `showTypingEvents`, and `showRecordingEvents`.
7. **Deploy statuses:** `INITIALIZATION | READY_TO_SCAN | ONLINE | FAILED` (`action='qr'` outside `READY_TO_SCAN` returns `deploy_qr_not_ready`).
8. **Repeatable events:** `EVENTS.ACTION` and `EVENTS.IG_COMMENT` can appear in multiple flows; `EVENTS.VOICE_NOTE`, `EVENTS.MEDIA`, and `EVENTS.DOCUMENT` are unique per project.
9. **Webhook, WhatsApp templates and Flows (v2.5):** `builderbot_webhook`, `builderbot_whatsapp_template` and `builderbot_whatsapp_flow` (the last two need WhatsApp connected through Meta). A new template is reviewed by Meta and is usable only once `APPROVED`; sending a template message is REST, not an MCP tool.
10. **Confirmation gate (v2.5):** deleting a template and publishing or retiring a Flow return `code: "confirmation_required"` until the call carries `confirm: true`, which the agent sets only after the person's explicit yes.
11. **Requires the matching MCP version:** these tools exist only in `builderbot-mcp-http` builds that register them (19 tools). Publish this skill after that server is deployed, otherwise the agent will call tools that do not exist yet (`unknown_tool`).

---

## How IronClaw Consumes This Repo

`scripts/ironclaw/bootstrap-reborn.sh` in `builderbot-cloud` clones this repository (`https://github.com/leifermendez/bbc-skill-mcp.git`), copies `SKILL.md` and `references/` into `scripts/ironclaw/skills/bbc-skill-tool/`, and installs `SKILL.md` as `bbc-skill-tool` via the IronClaw HTTP API.
