# Editable Checklist — Design

**Date:** 2026-08-08
**Extends:** 2026-08-08-admin-mode-design.md (implemented and live)

## Goal

Make the inspection checklist items editable in admin mode, like techs,
stations, and arrays.

## Design

### Data

`data.json` gains a top-level `"checklist"` array seeded with the current
11 items in form order:

```json
"checklist": [
  "Login Completed", "TAPs", "QR Scans", "Barrier alignment",
  "Loose Bolts Check", "Barrier speed", "Corrosion Check",
  "Customer Displays", "Status Displays", "Decals", "Pics (upload to chat)"
]
```

Array order is display order; never sorted. `DEFAULT_DATA` in
`faregate.html` gains the same key.

### Backward compatibility

Existing cached copies (localStorage) and the currently-published
`data.json` lack the `checklist` key. Every load path normalizes: after a
dataset passes `isValidData`, if `checklist` is not a non-empty array of
strings, it is set to a copy of `DEFAULT_DATA.checklist`. `isValidData`
itself does NOT require the key (old data must not be rejected).

### Form

The 11 hardcoded `<label class="check-item">` elements are replaced by a
`renderChecklist()` that builds them from `localData.checklist` (same
markup, two-column grid, checkboxes unchecked). Re-rendering resets check
state; that is acceptable because re-renders happen on data changes
(admin edits, remote updates), not during normal form use. The
"tap title to select all" toggle and `buildSummary()` already operate on
the live DOM and need no changes. `renderAll()` calls `renderChecklist()`.

### Admin

New **Checklist** section between Stations and Settings, identical in
behavior to Techs: rows with edit ✏️ / delete 🗑, "Add item" input+button
appending to the end, press-and-hold drag reordering via the existing
`makeReorderable`. All mutations go through `markChanged()`; Publish and
Discard need no changes (they operate on whole datasets).

**Guard:** deleting the last remaining checklist item is rejected with a
toast ("At least one checklist item is required") so the form can never
render an empty checklist section.

## Out of scope

No checkbox-state persistence, no changes to save/PNG behavior beyond the
new item order, no other admin changes.

## Testing

Local server + browser: form renders 11 items from data; add/edit/delete/
reorder reflect on the form immediately; deleting down to one item blocks
with a toast; a dataset without `checklist` (simulate stale localStorage)
renders the default 11; publish path unchanged (verified with dummy token
locally). Live: publish once after deploy so `data.json` gains the key.
