# Editable Checklist Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Checklist items become data-driven and editable (add/edit/delete/reorder) in admin mode, with the last item protected from deletion.

**Architecture:** Follows the shipped admin-mode pattern exactly: new `checklist` array in `data.json`/`DEFAULT_DATA`, normalization for old cached data, form section rendered from data, admin section identical to Techs plus a last-item delete guard.

**Tech Stack:** Vanilla JS in faregate.html. No test framework; manual browser verification.

**Spec:** `docs/superpowers/specs/2026-08-08-editable-checklist-design.md`

## Global Constraints

- Everything stays in `faregate.html` + `data.json`.
- `checklist` order is display order; adds append; never sort.
- Old datasets without `checklist` must normalize to the default 11 items — never be rejected and never render an empty checklist.
- Deleting the last checklist item is rejected with toast exactly: `At least one checklist item is required`.
- All mutations go through `markChanged()`. Existing form behavior (select-all toggle, buildSummary, save/PNG) unchanged.
- Do not push to the remote (controller handles push with the user).

---

### Task 1: Data-driven, admin-editable checklist

**Files:**
- Modify: `data.json`, `faregate.html`

**Interfaces:**
- Consumes (all exist): `DEFAULT_DATA`, `deepCopy`, `isValidData`, `localData`, `remoteData`, `loadStored`, `saveStored`, `LS_LOCAL`, `LS_REMOTE`, `renderAll`, `fetchRemoteData`, `markChanged`, `renderAdmin`, `makeReorderable`, `moveItem`, `showToast`; admin CSS classes `.admin-row`, `.admin-row-head`, `.admin-row-name`, `.admin-list`, `.admin-add`.
- Produces: `normalizeData(d)`, `renderChecklist()`, `renderChecklistAdmin()`, elements `#checkList`, `#checkNew`, `#checkAdd`.

- [ ] **Step 1: `data.json`** — add a top-level `"checklist"` key (after `"techs"`), value exactly:

```json
["Login Completed", "TAPs", "QR Scans", "Barrier alignment", "Loose Bolts Check", "Barrier speed", "Corrosion Check", "Customer Displays", "Status Displays", "Decals", "Pics (upload to chat)"]
```

(keep the file's existing 2-space formatting style — one string per line is fine).

- [ ] **Step 2: `DEFAULT_DATA`** — add the same `checklist` array (same position) to the `DEFAULT_DATA` literal in faregate.html.

- [ ] **Step 3: Normalization.** After the `isValidData` function definition, add:

```js
function normalizeData(d){
  if (!Array.isArray(d.checklist) || d.checklist.length === 0){
    d.checklist = deepCopy(DEFAULT_DATA.checklist);
  }
  return d;
}
```

Apply it at every load site:
- initial loads: after the two `if (!isValidData(...)) ... = deepCopy(DEFAULT_DATA);` fallbacks, add `localData = normalizeData(localData);` and `remoteData = normalizeData(remoteData);`
- in `fetchRemoteData`, right after the `isValidData(data)` check passes, add `normalizeData(data);` before any assignment.

`isValidData` itself must NOT be changed (old data stays valid).

- [ ] **Step 4: Form renders checklist from data.** In the HTML, delete the 11 hardcoded `<label class="check-item">…</label>` lines inside `<div class="checklist">`, leaving the empty container. Add after `renderStationOptions`:

```js
function renderChecklist(){
  const box = document.querySelector(".checklist");
  box.innerHTML = "";
  localData.checklist.forEach(item => {
    const label = document.createElement("label");
    label.className = "check-item";
    const cb = document.createElement("input");
    cb.type = "checkbox";
    const span = document.createElement("span");
    span.textContent = item;
    label.append(cb, " ", span);
    box.appendChild(label);
  });
}
```

and change `renderAll` to `{ renderTechOptions(); renderStationOptions(); renderChecklist(); }`. (`buildSummary` reads `c.nextElementSibling.textContent` — the span remains the checkbox's next element sibling, so it needs no change. The select-all title toggle also needs no change.)

- [ ] **Step 5: Admin section HTML.** Insert between the Stations `</section>` and the Settings `<section>`:

```html
<section>
  <h3>Checklist</h3>
  <div class="admin-list" id="checkList"></div>
  <div class="admin-add">
    <input id="checkNew" placeholder="New checklist item" />
    <button type="button" class="btn btn-primary" id="checkAdd">Add</button>
  </div>
</section>
```

- [ ] **Step 6: Admin JS.** Add `renderChecklistAdmin` next to `renderTechAdmin` (same shape, plus the last-item guard and aria-labels):

```js
function renderChecklistAdmin(){
  const list = document.getElementById("checkList");
  list.innerHTML = "";
  localData.checklist.forEach((item, i) => {
    const row  = document.createElement("div");
    row.className = "admin-row";
    const head = document.createElement("div");
    head.className = "admin-row-head";

    const name = document.createElement("span");
    name.className = "admin-row-name";
    name.textContent = item;

    const edit = document.createElement("button");
    edit.type = "button";
    edit.textContent = "✏️";
    edit.setAttribute("aria-label", "Edit " + item);
    edit.addEventListener("click", () => {
      const v = prompt("Edit checklist item", item);
      if (v && v.trim()){ localData.checklist[i] = v.trim(); markChanged(); }
    });

    const del = document.createElement("button");
    del.type = "button";
    del.textContent = "🗑";
    del.setAttribute("aria-label", "Delete " + item);
    del.addEventListener("click", () => {
      if (localData.checklist.length <= 1){
        showToast("At least one checklist item is required");
        return;
      }
      if (confirm(`Delete ${item}?`)){ localData.checklist.splice(i, 1); markChanged(); }
    });

    head.append(name, edit, del);
    row.appendChild(head);
    list.appendChild(row);
  });
}
```

Update `renderAdmin` to call it after `renderStationAdmin()` (before `updateAdminChrome()`).

Add next to the `techAdd` listener:

```js
document.getElementById("checkAdd").addEventListener("click", () => {
  const input = document.getElementById("checkNew");
  const v = input.value.trim();
  if (!v) return;
  localData.checklist.push(v);
  input.value = "";
  markChanged();
});
```

Add next to the existing persistent-list `makeReorderable` attachments:

```js
makeReorderable(document.getElementById("checkList"), (from, to) => {
  moveItem(localData.checklist, from, to);
  markChanged();
});
```

- [ ] **Step 7: Verify in browser** (serve repo root on :8080, use browser tools; override prompt/confirm via javascript_tool as needed):
  - Form shows the 11 checklist items; select-all title toggle still toggles all; Save button still produces a PNG download (spot-check it triggers without error).
  - Stale-cache simulation: set localStorage `faregate.local`/`faregate.remote` to valid data WITHOUT a `checklist` key, reload → form still shows the default 11, no dirty badge (both sides normalized equally).
  - Admin: new Checklist section lists 11 items; add → appears at form bottom + badge; edit → updates form; delete works with confirm; delete repeatedly down to 1 item → last delete attempt toasts "At least one checklist item is required" and item survives; drag-reorder a row → form order follows.
  - Restore clean state (reset localStorage, reload, badge hidden, 11 items). Console free of uncaught errors.

- [ ] **Step 8: Commit**

```bash
git add data.json faregate.html
git commit -m "feat: editable checklist in admin mode"
```
