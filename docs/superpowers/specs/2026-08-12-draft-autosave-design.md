# Form Draft Autosave — Design

**Date:** 2026-08-12
**Extends:** admin-mode / editable-checklist / chat-webhook designs (live)

## Goal

Reopening the app (home-screen icon, tab reload, browser memory eviction)
restores the in-progress form exactly as the tech left it.

## Design

### Draft storage

localStorage key `faregate.draft` (constant `LS_DRAFT`), shape:

```json
{ "t": 1755043200000, "tech": "...", "date": "YYYY-MM-DD", "station": "id",
  "array": "...", "start": "HH:MM", "end": "HH:MM", "notes": "...",
  "checked": ["Login Completed", "..."] }
```

`checked` stores checklist item *labels* (not indexes) so it survives
admin reorders/renames gracefully (renamed items simply come back
unchecked).

### Saving

`saveDraft()` serializes the form and writes the key. Triggered by:
- `input` and `change` events delegated on the form element (covers
  selects, date/time inputs, textarea, checkbox taps)
- the select-all checklist-title toggle handler (programmatic check
  changes fire no events)
- the Clear button, AFTER it resets fields — so the draft records the
  post-clear state (tech/station/date kept, rest empty), preserving
  Clear's existing semantics across reloads

### Restoring

`restoreDraft()` runs at init after `setCurrentDate()` + `renderAll()`
and before `fetchRemoteData()`:

- Missing draft, malformed `t`, or older than 12 hours
  (`DRAFT_MAX_AGE_MS`) → remove the key and do nothing (fresh form,
  today's date).
- Otherwise restore every field; empty stored date keeps the auto-set
  today. Station restore calls `populateArrayOptions` before restoring
  the array value. Any select value that no longer exists (admin deleted
  it) falls back to `""`. Notes height re-syncs (same auto-expand logic).
  Checklist boxes re-check by label.

### Checklist survives re-renders

`renderChecklist()` currently rebuilds with all boxes unchecked, so an
admin edit or the startup remote fetch wipes an in-progress checklist.
It now snapshots currently-checked labels from the DOM before clearing
and re-applies them to the rebuilt rows. (Independent of drafts but
required for restore to survive `fetchRemoteData`'s re-render.)

### Unchanged

Save/PNG/chat-send, Publish/Discard, admin panel, `data.json` — no schema
or admin changes. Draft is per device, never synced.

## Testing

Browser: fill form → reload → everything restored (station's arrays too);
draft older than 12 h (forge `t`) → fresh form; Clear → reload keeps only
tech/station/date; select-all toggle → reload restores all checked;
checked boxes survive a simulated `fetchRemoteData` re-render; deleted
station in draft → falls back to placeholder without errors.
