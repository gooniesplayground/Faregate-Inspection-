# Admin Mode with GitHub Publishing — Design

**Date:** 2026-08-08
**App:** Faregate Inspection Checklist (`faregate.html`, served at
`https://gooniesplayground.github.io/Faregate-Inspection-/faregate`)

## Goal

Add an admin mode to the single-file web app that lets an admin add, remove,
edit, and reorder technician names, station names, and each station's array
names — and publish those changes to the GitHub repo so every user's app
picks them up.

## Architecture

### Data file

The three lists currently hardcoded in `faregate.html` (tech `<option>`s,
station `<option>`s, and the `arrayData` JS object) move into a new
`data.json` at the repo root:

```json
{
  "techs": ["Asanti H (60572)", "Andy Y (59116)"],
  "stations": [
    {
      "id": "124",
      "line": "🔴",
      "name": "7th Street/Metro Center",
      "arrays": ["Figueroa 49-57", "Hope 59-67", "Flower 69-71"]
    }
  ]
}
```

- Arrays are nested inside their station: the station↔array correlation is
  structural, deleting a station removes its arrays, and orphaned arrays are
  impossible.
- List order in `data.json` is the display order of every dropdown. No
  auto-sorting anywhere.
- `data.json` is seeded from the current hardcoded lists (17 techs,
  26 stations with their existing IDs, line emojis, and arrays).

### App load

On startup the app:

1. Fetches `data.json` from the Pages site with a cache-busting query
   (`data.json?t=<timestamp>`).
2. On success: builds the Tech, Station, and Array dropdowns from it and
   writes the copy to `localStorage` (`faregateData`).
3. On fetch failure (offline): falls back to the `localStorage` copy;
   failing that, to a default dataset embedded in the HTML (a copy of the
   seed data).

The inspection form itself behaves exactly as today.

### Admin mode

**Access:** tapping the "Faregate Inspection" title 5 times within ~3
seconds opens a full-screen admin panel overlay. No PIN. Real protection is
the GitHub token: without one, Publish is disabled, so anyone who finds the
gesture can look but not publish.

**Panel sections:**

- **Techs** — each row shows the name with edit (✏️) and delete (🗑)
  buttons; an "Add tech" input+button appends a new name.
- **Stations** — each row shows `line emoji + name`; tapping a row expands
  it to:
  - edit station name, station ID, and line emoji (picker offering
    🔴 🟣 🔵 🟢 🟡 ⚪ 🟠 ⚫),
  - manage that station's arrays (add / edit / delete / reorder),
  - delete the station (with its arrays).
  - "Add station" appends a new station (name + ID + line required).
  - Station IDs must be unique; adding or editing to a duplicate ID is
    rejected with a toast.
- **Settings** — paste-field for the GitHub token (stored only in that
  device's `localStorage`, never in the repo), "Test connection" button,
  and a "Forget token" button.

**Reordering:** press-and-hold (~400 ms) on a tech row, station row, or
array row lifts it (visual feedback: slight scale/shadow); dragging moves
it within its own list; releasing drops it. Implemented with pointer
events; scrolling is suppressed only while a row is lifted. Works with
touch and mouse.

**Edit lifecycle:**

- All edits are local (in-memory + `localStorage`) until **Publish**.
- Local edits apply to the form's dropdowns immediately, so the admin can
  sanity-check before publishing.
- **Discard** restores the last-published dataset.
- A "unpublished changes" badge shows when local state differs from the
  last fetched `data.json`.
- Deletes ask for a one-tap confirm. No undo after publish; the git
  history of `data.json` is the recovery mechanism.

### Publishing

Publish commits `data.json` to `gooniesplayground/Faregate-Inspection-`
via the GitHub Contents API:

1. `GET /repos/{owner}/{repo}/contents/data.json` → current SHA.
2. `PUT` the same path with base64 content, the SHA, commit message
   `Update data.json via admin mode`, and the token.

**Failure handling (all surfaced as toasts):**

- 401/403 → "Token invalid or expired — check Settings."
- Network failure → "No connection — changes kept locally, try again."
- 409 / SHA mismatch (someone else published first) → refetch the remote
  file, keep local edits in memory, and tell the admin to review and
  publish again.

**Propagation:** other users get the new lists on their next app
open/reload after GitHub Pages redeploys (~30–60 s).

### Token setup (one-time, admin's device only)

A fine-grained Personal Access Token restricted to the one repo with only
**Contents: read/write** permission. Step-by-step creation instructions go
in the repo README. The token lives only in the admin device's
`localStorage`. Worst case if the device is compromised: someone can edit
this one repo — recoverable via git history.

## Cleanup included

The current Clear-button handler has init code accidentally nested inside
it (`faregate.html` lines ~584–598): the notes auto-expand listener and
`setCurrentDate()` are re-registered/re-run on every Clear tap, duplicating
listeners. The script restructure fixes this.

## Out of scope

- No PIN or client-side auth.
- No backend/server component.
- No undo/history UI (git history covers it).
- No changes to the inspection form's behavior, checklist items, save/
  camera/clear features, or visual design.

## Testing

- Local: serve the folder (`python -m http.server` or similar), verify
  load-from-JSON, offline fallback (DevTools offline), admin CRUD, drag
  reorder (mouse + touch emulation), and publish against the real repo
  with a test commit.
- Device: verify the 5-tap gesture, long-press reorder, and publish flow
  on a phone via the Pages URL.
