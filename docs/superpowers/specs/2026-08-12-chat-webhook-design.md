# Send-to-Chat on Save — Design

**Date:** 2026-08-12
**Extends:** admin-mode + editable-checklist designs (implemented and live)

## Goal

When a tech taps **Save**, the app posts the inspection as a text message
to a private Google Chat space automatically (no extra taps), in addition
to the existing PNG download. Admins configure it once by pasting the
space's incoming-webhook URL into admin Settings and publishing.

## Design

### Data

`data.json` gains a top-level `"chatWebhook"` string (default `""` =
feature off). Distributed to all devices through the existing
fetch/publish pipeline, so techs need zero setup. `DEFAULT_DATA` gains the
same key; `normalizeData` coerces a missing/non-string value to `""`
(backward compatible with cached data). `isValidData` unchanged.

Exposure note (accepted): the webhook URL is public in the repo. Worst
case is text spam into the space; remedy is regenerating the webhook and
re-publishing.

### Admin UI

Settings section gains, above the token controls:

- a text input `#webhookInput` showing the current `localData.chatWebhook`
  (repopulated on renderAdmin)
- an **Apply** button `#webhookApply`: value must be empty (disables the
  feature) or start with `https://chat.googleapis.com/v1/spaces/` —
  otherwise toast `❌ Not a Google Chat webhook URL` and leave data
  unchanged. Valid values set `localData.chatWebhook` via the normal
  `markChanged()` flow (dirty badge → Publish distributes it).
- a hint line: "Paste the space's webhook URL, Apply, then Publish."

### Save behavior

Unchanged when `chatWebhook` is empty. When set, `Save`:

1. Runs the existing screenshot/download flow (unchanged, including its
   failure path).
2. After the PNG succeeds, POSTs `buildSummary()` (already existing
   function, currently unused) as JSON `{ "text": ... }` with
   `Content-Type: application/json; charset=UTF-8` to the webhook.
3. Toast on the combined outcome:
   - sent ok → `✅ Saved + sent to chat!`
   - send failed (non-2xx or network) → `⚠️ Saved. Chat send failed —
     send manually` (PNG is still the record; nothing is lost)

CORS verified: chat.googleapis.com answers preflight with
`access-control-allow-origin: <origin>`, POST + content-type allowed, so a
normal fetch works and real success/failure is observable.

## Out of scope

No image upload to chat (text only, per decision), no Camera-photo
sending, no retry queue for failed sends.

## Testing

Local browser: admin Apply validation (bad URL toast; valid URL sets
dirty), Save with empty webhook behaves exactly as today; Save with a
mocked webhook (monkey-patched fetch) shows the sent toast and posts
`{text}` JSON with the summary; mocked failure shows the warning toast and
the PNG still downloads. Live: real webhook end-to-end once the user
obtains one (they paste → Apply → Publish → Save posts to the space).

## README

Setup guide gains an optional step: enabling webhooks in the Chat space
(space name → Apps & integrations → Webhooks → Add), pasting into admin
Settings, Apply, Publish. Plain-language, matching the existing README
style.
