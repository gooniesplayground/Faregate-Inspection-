# Open-Chat Camera Button + Dynamic Labels — Design

**Date:** 2026-08-12
**Extends:** chat-webhook design (live)

## Goal

The Camera button opens the crew's Google Chat space (where the tech uses
Chat's own camera to post photos), and the action buttons are renamed to
match their chat roles.

## Design

### Data

`data.json` gains top-level `"chatSpaceLink": ""` (next to `chatWebhook`).
`DEFAULT_DATA` matches; `normalizeData` coerces non-strings to `""`.
Distributed via the normal publish pipeline; `isValidData` unchanged.

### Admin UI

Settings gains a second link row above the webhook row: input
`#spaceLinkInput` + Apply `#spaceLinkApply`. Validation: empty (off) or
starts with `https://chat.google.com/` — else toast exactly
`❌ Not a Google Chat link`. Valid values flow through `markChanged()`;
input repopulates on renderAdmin. Hint text notes: paste the space's
link (space menu → Copy link), Apply, Publish.

### Buttons (labels follow configuration)

A new `updateActionButtons()` (called from `renderAll()`) sets:

- **Save button** — webhook configured → label `Submit to Google Chat`;
  otherwise → `Save`. Behavior unchanged (existing either/or logic).
- **Camera button** — `chatSpaceLink` configured → label
  `Open Google Chat`, and clicking opens the link in a new tab (on
  phones this deep-links into the Google Chat app); otherwise → label
  `📷 Camera` with today's take-photo-and-download behavior.

Dynamic labels keep the buttons honest on devices/forks where chat isn't
configured; once the manager publishes both links, every phone shows the
chat labels. Button font stays as-is; longer labels may wrap to two lines
(acceptable at current sizes — verify visually).

### README

The webhook section becomes "Connect the app to Google Chat" covering
both fields: webhook URL (Save → chat) and space link (Camera button →
opens the space), including where to find each in Google Chat.

## Out of scope

No photo capture changes when unconfigured, no in-app photo upload to
Chat (not possible via webhook), no other admin changes.

## Testing

Browser: Apply validation for both good/bad space links; unconfigured →
buttons read Save / 📷 Camera and behave as today; configured (local
edit) → labels change, Camera click opens the link in a new tab (stub
window.open), Save behavior untouched; Discard restores labels. Console
clean.
