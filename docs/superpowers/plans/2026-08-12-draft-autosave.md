# Form Draft Autosave Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The in-progress form auto-saves to localStorage on every change and restores on page load, so home-screen relaunches don't lose work.

**Architecture:** New `faregate.draft` localStorage key written by a delegated form listener, restored at init; `renderChecklist` preserves checked state across rebuilds.

**Tech Stack:** Vanilla JS in faregate.html only. Manual browser verification.

**Spec:** `docs/superpowers/specs/2026-08-12-draft-autosave-design.md`

## Global Constraints

- Only `faregate.html` changes. No `data.json` schema change, no admin-panel changes, no changes to Save/PNG/chat-send/Publish logic.
- Draft key exactly `"faregate.draft"`; expiry 12 hours.
- Clear's semantics (keeps Tech/Station/Date) must hold across a reload.
- No push (controller handles).

---

### Task 1: Draft autosave + restore + checklist state preservation

**Files:**
- Modify: `faregate.html`

**Interfaces:**
- Consumes (all exist): `loadStored`, `saveStored`, `populateArrayOptions`, `renderChecklist`, `renderAll`, `setCurrentDate`, `fetchRemoteData`, the `#inspectionForm` element, the checklist-title select-all handler, the `#clearBtn` handler, the notes auto-expand listener.
- Produces: `LS_DRAFT`, `DRAFT_MAX_AGE_MS`, `saveDraft()`, `restoreDraft()`.

- [ ] **Step 1: Constants.** Next to the other `LS_*` constants add:

```js
const LS_DRAFT  = "faregate.draft";
const DRAFT_MAX_AGE_MS = 12 * 60 * 60 * 1000;
```

- [ ] **Step 2: saveDraft.** Add after `restoreDraft`'s planned location (near the render functions, e.g. after `renderChecklist`):

```js
function saveDraft(){
  saveStored(LS_DRAFT, {
    t: Date.now(),
    tech: document.getElementById("technician").value,
    date: document.getElementById("date").value,
    station: document.getElementById("station").value,
    array: document.getElementById("array").value,
    start: document.getElementById("timeStarted").value,
    end: document.getElementById("timeFinished").value,
    notes: document.getElementById("notes").value,
    checked: [...document.querySelectorAll('.checklist input[type="checkbox"]:checked')]
      .map(c => c.nextElementSibling.textContent)
  });
}

function restoreDraft(){
  const d = loadStored(LS_DRAFT);
  if (!d || typeof d.t !== "number" || Date.now() - d.t > DRAFT_MAX_AGE_MS){
    localStorage.removeItem(LS_DRAFT);
    return;
  }
  const setSel = (id, val) => {
    const sel = document.getElementById(id);
    sel.value = val || "";
    if (sel.value !== (val || "")) sel.value = "";
  };
  setSel("technician", d.tech);
  if (d.date) document.getElementById("date").value = d.date;
  setSel("station", d.station);
  populateArrayOptions(document.getElementById("station").value);
  setSel("array", d.array);
  document.getElementById("timeStarted").value = d.start || "";
  document.getElementById("timeFinished").value = d.end || "";
  const notes = document.getElementById("notes");
  notes.value = d.notes || "";
  notes.style.height = "auto";
  notes.style.height = notes.scrollHeight + "px";
  const want = new Set(d.checked || []);
  document.querySelectorAll('.checklist input[type="checkbox"]').forEach(c => {
    c.checked = want.has(c.nextElementSibling.textContent);
  });
}
```

- [ ] **Step 3: Checklist state survives rebuilds.** In `renderChecklist`, before `box.innerHTML = "";` add:

```js
  const checkedLabels = new Set(
    [...box.querySelectorAll('input[type="checkbox"]:checked')]
      .map(c => c.nextElementSibling.textContent)
  );
```

and inside the forEach, after `cb.type = "checkbox";` add:

```js
    cb.checked = checkedLabels.has(item);
```

- [ ] **Step 4: Triggers.**
  - In the select-all checklist-title click handler, add `saveDraft();` as the last line (after the boxes are toggled).
  - In the `#clearBtn` click handler, add `saveDraft();` as the last line (after the fields are reset).
  - In the Init section (after the notes auto-expand listener), add:

```js
document.getElementById("inspectionForm").addEventListener("input", saveDraft);
document.getElementById("inspectionForm").addEventListener("change", saveDraft);
```

- [ ] **Step 5: Restore at init.** Change the init tail from `setCurrentDate(); renderAll(); fetchRemoteData();` to:

```js
setCurrentDate();
renderAll();
restoreDraft();
fetchRemoteData();
```

- [ ] **Step 6: Verify in browser** (serve :8080, browser tools):
  - Fill tech, station Willowbrook (145), an array, start/end times, notes, check 3 boxes → reload → every field identical, array dropdown populated with the right options, notes textarea grown, exactly those 3 boxes checked, no console errors.
  - Select-all via title tap → reload → all boxes checked.
  - Clear → reload → tech/station/date still set, array/times/notes/boxes empty.
  - Stale draft: via javascript_tool overwrite `faregate.draft`'s `t` to `Date.now() - 13*3600*1000`, reload → fresh form (today's date, nothing restored), draft key removed.
  - Deleted-station fallback: set a draft whose `station` is `"9999"` (nonexistent), reload → station shows "Select", no errors.
  - Re-render survival: check 2 boxes, then via javascript_tool call `renderChecklist()` directly → boxes stay checked.
  - Clean up: clear localStorage, reload; badge hidden; console clean.

- [ ] **Step 7: Commit** — `git add faregate.html`, message `feat: auto-save form draft and restore on reload`
