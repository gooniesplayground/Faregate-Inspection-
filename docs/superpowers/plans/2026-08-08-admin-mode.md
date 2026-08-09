# Admin Mode with GitHub Publishing — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add an admin mode to `faregate.html` that edits/reorders techs, stations, and per-station arrays, and publishes changes as `data.json` to the GitHub repo so all users pick them up.

**Architecture:** The three hardcoded lists move to `data.json` at the repo root. The app fetches it on load (localStorage → embedded defaults as fallbacks) and renders all dropdowns from it. A hidden admin panel (5 taps on the title) edits a local working copy and publishes via the GitHub Contents API with a fine-grained PAT stored only in the admin device's localStorage.

**Tech Stack:** Vanilla JS in a single HTML file. No frameworks, no build step, no test framework (matches existing repo). Verification is manual in a browser against a local static server.

**Spec:** `docs/superpowers/specs/2026-08-08-admin-mode-design.md`

## Global Constraints

- Everything stays in `faregate.html` (single-file app) except the new `data.json` and README additions.
- Repo: `gooniesplayground/Faregate-Inspection-`, branch `main`, served at `https://gooniesplayground.github.io/Faregate-Inspection-/faregate`.
- localStorage keys: `faregate.local` (working data), `faregate.remote` (last known published data), `faregate.token` (GitHub PAT).
- List order in `data.json` IS display order; never sort.
- Station IDs must be unique; duplicates rejected with a toast.
- The inspection form's existing behavior (save/camera/clear/checklist/times) must not change.
- Do NOT `git push` until the final task, and ask the user before pushing (pushing deploys to all users).
- Local verification server: run `python -m http.server 8080` from the repo root, browse `http://localhost:8080/faregate.html`. (If Python is missing, `npx serve -l 8080` works the same.)

---

### Task 1: Create `data.json`

**Files:**
- Create: `data.json`

**Interfaces:**
- Produces: `data.json` with shape `{ techs: string[], stations: { id: string, line: string, name: string, arrays: string[] }[] }`. All later tasks rely on exactly this shape.

- [ ] **Step 1: Write `data.json`** at the repo root with exactly this content (seeded from the current hardcoded lists in `faregate.html` — data quirks like `"Center472-538"` and `"West 462-462"` are intentional, copy verbatim):

```json
{
  "techs": [
    "Asanti H (60572)",
    "Andy Y (59116)",
    "Charles L (59244)",
    "Edward H",
    "Eric P (62789)",
    "Francisco L (62763)",
    "Jimmy Z (59149)",
    "Jose J (60623)",
    "Joshua A (62726)",
    "Juan H (61629)",
    "Julio M (62727)",
    "Keith F (62790)",
    "Mason M (00000)",
    "Rafael T (61287)",
    "Revic C (58631)",
    "Roger D (60620)",
    "Sal A (00000)"
  ],
  "stations": [
    { "id": "120", "line": "🔴", "name": "North Hollywood", "arrays": ["West 154-158", "East 161-166"] },
    { "id": "131", "line": "🔴", "name": "Hollywood/Western", "arrays": ["Main"] },
    { "id": "129", "line": "🔴", "name": "Vermont/Santa Monica", "arrays": ["North 104-107", "South 110-112"] },
    { "id": "125", "line": "🔴", "name": "Wilshire/Vermont", "arrays": ["Main", "Elevator"] },
    { "id": "104", "line": "🔴", "name": "Westlake/MacArthur Park", "arrays": ["South"] },
    { "id": "124", "line": "🔴", "name": "7th Street/Metro Center", "arrays": ["Figueroa 49-57", "Hope 59-67", "Flower 69-71"] },
    { "id": "123", "line": "🔴", "name": "Pershing Square", "arrays": ["South 35-41", "North 43-47"] },
    { "id": "122", "line": "🔴", "name": "Civic Center", "arrays": ["North 26-32", "South 911-913"] },
    { "id": "126", "line": "🟣", "name": "Wilshire/Normandie", "arrays": ["Main"] },
    { "id": "127", "line": "🟣", "name": "Wilshire/Western", "arrays": ["Main"] },
    { "id": "2986", "line": "🟣", "name": "Wilshire/La Brea", "arrays": ["Main"] },
    { "id": "2987", "line": "🟣", "name": "Wilshire/Fairfax", "arrays": ["Main"] },
    { "id": "2988", "line": "🟣", "name": "Wilshire/La Cienega", "arrays": ["Main"] },
    { "id": "2978", "line": "🔵", "name": "Pomona", "arrays": ["West 486-490", "EastWest 493-494", "EastEast 498-497"] },
    { "id": "2977", "line": "🔵", "name": "La Verne", "arrays": ["West 478-479", "East 482-483"] },
    { "id": "2976", "line": "🔵", "name": "San Dimas", "arrays": ["West"] },
    { "id": "2975", "line": "🔵", "name": "Glendora", "arrays": ["West 462-462", "East 466-467"] },
    { "id": "73", "line": "🔵", "name": "Sierra Madre Villa", "arrays": ["Main"] },
    { "id": "168", "line": "🔵", "name": "Allen", "arrays": ["North East"] },
    { "id": "167", "line": "🔵", "name": "Lake", "arrays": ["Main"] },
    { "id": "144", "line": "🟢", "name": "Firestone", "arrays": ["North 605-606", "South 608-609"] },
    { "id": "145", "line": "🟢", "name": "Willowbrook - Rosa Parks", "arrays": ["West 340-347", "Center 333-337+460", "Transfer Mezzanine", "East 353+458/9", "South 450-452"] },
    { "id": "2941", "line": "🟢", "name": "LAX/MTC", "arrays": ["South 206-211", "North 213-218"] },
    { "id": "174", "line": "🟢", "name": "Aviation/Imperial", "arrays": ["East 411-413", "West 415-417", "Center472-538"] },
    { "id": "171", "line": "🟢", "name": "Douglas", "arrays": ["North 436-438", "South 440-530"] },
    { "id": "60", "line": "🟢", "name": "Norwalk", "arrays": ["East 300-305", "West 307-534"] }
  ]
}
```

- [ ] **Step 2: Validate it parses and matches the old data**

Run: `python -m json.tool data.json > /dev/null && echo OK`
Expected: `OK`

Cross-check by eye against `faregate.html`: 17 techs, 26 stations, and each station's `arrays` matches the old `arrayData` entry for the same id.

- [ ] **Step 3: Commit**

```bash
git add data.json
git commit -m "feat: extract techs/stations/arrays into data.json"
```

---

### Task 2: Render the form from data (fetch → localStorage → defaults)

**Files:**
- Modify: `faregate.html` — the `<select id="technician">` (~lines 237–256), `<select id="station">` (~266–294), and the whole `<script>` block (~356–609)

**Interfaces:**
- Consumes: `data.json` from Task 1.
- Produces (later tasks rely on these exact names):
  - `let localData` / `let remoteData` — working and last-published datasets (`{techs, stations}` objects)
  - `const LS_LOCAL = "faregate.local"`, `LS_REMOTE = "faregate.remote"`, `LS_TOKEN = "faregate.token"`
  - `deepCopy(obj)`, `loadStored(key)`, `saveStored(key, val)`
  - `isDirty()` → boolean (localData differs from remoteData)
  - `renderAll()` — re-renders tech/station/array dropdowns from `localData`, preserving selections
  - `fetchRemoteData()` → Promise; updates `remoteData` (and `localData` if it was clean), saves both, calls `renderAll()`
  - `showToast(msg)` — already exists, unchanged

- [ ] **Step 1: Strip the hardcoded options.** In the technician select, delete all `<option>` lines except `<option value="">Select</option>`. Same for the station select. (The array select already has only the placeholder.)

- [ ] **Step 2: Replace the data layer.** In the `<script>`, replace the `const arrayData = {...};` block with:

```js
const DEFAULT_DATA = /* paste the exact full contents of data.json here */;

const LS_LOCAL  = "faregate.local";
const LS_REMOTE = "faregate.remote";
const LS_TOKEN  = "faregate.token";

function deepCopy(obj){ return JSON.parse(JSON.stringify(obj)); }

function loadStored(key){
  try { return JSON.parse(localStorage.getItem(key)); }
  catch(e){ return null; }
}

function saveStored(key, val){ localStorage.setItem(key, JSON.stringify(val)); }

let localData  = loadStored(LS_LOCAL)  || deepCopy(DEFAULT_DATA);
let remoteData = loadStored(LS_REMOTE) || deepCopy(DEFAULT_DATA);

function isDirty(){ return JSON.stringify(localData) !== JSON.stringify(remoteData); }

function renderTechOptions(){
  const sel = document.getElementById("technician");
  const prev = sel.value;
  sel.innerHTML = '<option value="">Select</option>';
  localData.techs.forEach(t => {
    const o = document.createElement("option");
    o.textContent = t;
    sel.appendChild(o);
  });
  sel.value = prev;
}

function renderStationOptions(){
  const sel = document.getElementById("station");
  const prev = sel.value;
  sel.innerHTML = '<option value="">Select</option>';
  localData.stations.forEach(s => {
    const o = document.createElement("option");
    o.value = s.id;
    o.textContent = `${s.line} ${s.name}`;
    sel.appendChild(o);
  });
  sel.value = prev;
  populateArrayOptions(sel.value);
}

function renderAll(){ renderTechOptions(); renderStationOptions(); }

async function fetchRemoteData(){
  try {
    const res = await fetch(`data.json?t=${Date.now()}`, { cache: "no-store" });
    if (!res.ok) return;
    const data = await res.json();
    if (!data || !Array.isArray(data.techs) || !Array.isArray(data.stations)) return;
    const wasClean = !isDirty();
    remoteData = data;
    if (wasClean) localData = deepCopy(data);
    saveStored(LS_REMOTE, remoteData);
    saveStored(LS_LOCAL, localData);
    renderAll();
  } catch (e) { /* offline — keep local/default data */ }
}
```

- [ ] **Step 3: Rewrite `populateArrayOptions`** (keep the same name; it's called from the station change listener) to read from `localData` and preserve the selection:

```js
function populateArrayOptions(siteId){
  const sel = document.getElementById("array");
  const prev = sel.value;
  sel.innerHTML = '<option value="">Select</option>';
  const st = localData.stations.find(s => s.id === siteId);
  (st ? st.arrays : []).forEach(a => {
    const o = document.createElement("option");
    o.value = a;
    o.textContent = a;
    sel.appendChild(o);
  });
  sel.value = prev;
}
```

- [ ] **Step 4: Fix the Clear-button bug and clean up init.** The current Clear handler (`clearBtn` listener) has the notes auto-expand listener and `setCurrentDate()` accidentally nested inside it, and the same code repeats at the bottom of the script. Replace the Clear handler AND the trailing init block with:

```js
// Clear button — keeps Tech, Station, and Date
document.getElementById("clearBtn").addEventListener("click", () => {
  document.getElementById("array").value = "";
  document.getElementById("timeStarted").value = "";
  document.getElementById("timeFinished").value = "";
  document.getElementById("notes").value = "";
  document.querySelectorAll('.checklist input[type="checkbox"]').forEach(c => c.checked = false);
});

// Init
const notesEl = document.getElementById("notes");
notesEl.addEventListener("input", () => {
  notesEl.style.height = "auto";
  notesEl.style.height = notesEl.scrollHeight + "px";
});

setCurrentDate();
renderAll();
fetchRemoteData();
```

There must be exactly ONE notes-listener registration and ONE `setCurrentDate()` call in the whole script when done.

- [ ] **Step 5: Verify in browser.** Serve the repo root and open `http://localhost:8080/faregate.html`:
  - Tech dropdown shows all 17 names; station dropdown shows all 26 with emojis.
  - Picking "🟢 Willowbrook - Rosa Parks" fills the array dropdown with its 5 arrays.
  - DevTools → Application → Local Storage shows `faregate.local` and `faregate.remote`.
  - DevTools → Network tab → set "Offline", reload: dropdowns still populate (from localStorage).
  - Set network back online. Save button still downloads a PNG; Clear still clears; date auto-fills.

- [ ] **Step 6: Commit**

```bash
git add faregate.html
git commit -m "feat: render dropdowns from data.json with localStorage/default fallback"
```

---

### Task 3: Admin panel skeleton + 5-tap gesture

**Files:**
- Modify: `faregate.html` — add CSS to `<style>`, add panel HTML before `<div class="toast">`, add JS after the data layer

**Interfaces:**
- Consumes: `isDirty()`, `LS_TOKEN` from Task 2.
- Produces (later tasks rely on these):
  - Elements: `#adminPanel`, `#techList`, `#techNew`, `#techAdd`, `#stationList`, `#stNewName`, `#stNewId`, `#stNewLine`, `#stAdd`, `#tokenInput`, `#tokenSave`, `#tokenTest`, `#tokenForget`, `#publishBtn`, `#discardBtn`, `#adminDirty`
  - `renderAdmin()` — calls `renderTechAdmin()`, `renderStationAdmin()`, `updateAdminChrome()`
  - `renderTechAdmin()` / `renderStationAdmin()` — empty stubs here, implemented in Tasks 4–5
  - `updateAdminChrome()` — shows/hides dirty badge, enables Publish only when a token exists
  - `markChanged()` — persists `localData`, re-renders form dropdowns and admin panel
  - `openAdmin()` / `closeAdmin()`
  - CSS classes: `.admin-row` (block), `.admin-row-head` (flex header inside a row), `.admin-row.dragging`, `.admin-list`, `.station-detail`

- [ ] **Step 1: Add CSS** to the end of the `<style>` block (before the `@media` query):

```css
/* Admin panel */
.admin{
  position:fixed; inset:0; background:var(--bg);
  z-index:100; overflow-y:auto; padding:8px;
}
.admin-inner{ max-width:430px; margin:0 auto; }
.admin-head{ display:flex; align-items:center; gap:8px; padding:2px 2px 6px; }
.admin-head h2{ flex:1; margin:0; font-size:18px; }
.admin-dirty{ color:#b45309; font-size:12px; font-weight:700; }
.admin section{
  background:var(--card); border:1px solid #e5eaf0;
  border-radius:12px; padding:10px; margin-top:8px;
}
.admin h3{ margin:0 0 6px; font-size:14px; }
.admin-list{ display:grid; gap:4px; }
.admin-row{
  display:block; background:#fafbfd; border:1px solid var(--line);
  border-radius:8px; padding:6px 8px; font-size:13px;
  user-select:none; -webkit-user-select:none; -webkit-touch-callout:none;
}
.admin-row.dragging{
  opacity:.9; transform:scale(1.02);
  box-shadow:0 6px 14px rgba(0,0,0,.18); background:#e0f2fe;
}
.admin-row-head{ display:flex; align-items:center; gap:6px; }
.admin-row-name{ flex:1; line-height:1.3; }
.admin-row-head button{
  border:0; background:none; font-size:15px; cursor:pointer; padding:2px 4px; min-height:0;
}
.admin-add{ display:flex; gap:6px; margin-top:8px; }
.admin-add input, .admin-add select{ flex:1; min-width:0; }
.station-detail{ display:grid; gap:6px; padding:8px 0 2px; }
.station-detail label{ font-size:11px; font-weight:700; display:block; margin:0 0 2px; }
.admin-actions{ display:grid; grid-template-columns:1fr 1fr; gap:6px; margin:10px 0 24px; }
```

- [ ] **Step 2: Add the panel HTML** right before `<div class="toast" id="toast"></div>`:

```html
<div class="admin" id="adminPanel" hidden>
  <div class="admin-inner">
    <div class="admin-head">
      <h2>⚙️ Admin</h2>
      <span class="admin-dirty" id="adminDirty" hidden>● unpublished</span>
      <button type="button" class="btn btn-now" id="adminClose">✕ Close</button>
    </div>

    <section>
      <h3>Techs</h3>
      <div class="admin-list" id="techList"></div>
      <div class="admin-add">
        <input id="techNew" placeholder="Name (ID)" />
        <button type="button" class="btn btn-primary" id="techAdd">Add</button>
      </div>
    </section>

    <section>
      <h3>Stations</h3>
      <div class="admin-list" id="stationList"></div>
      <div class="admin-add">
        <select id="stNewLine">
          <option>🔴</option><option>🟣</option><option>🔵</option><option>🟢</option>
          <option>🟡</option><option>🟠</option><option>⚪</option><option>⚫</option>
        </select>
        <input id="stNewName" placeholder="Station name" />
        <input id="stNewId" placeholder="ID" style="max-width:70px" />
        <button type="button" class="btn btn-primary" id="stAdd">Add</button>
      </div>
    </section>

    <section>
      <h3>Settings</h3>
      <div class="admin-add">
        <input id="tokenInput" type="password" placeholder="GitHub token" autocomplete="off" />
        <button type="button" class="btn btn-primary" id="tokenSave">Save</button>
      </div>
      <div class="admin-add">
        <button type="button" class="btn btn-now" id="tokenTest">Test connection</button>
        <button type="button" class="btn btn-danger" id="tokenForget">Forget token</button>
      </div>
    </section>

    <div class="admin-actions">
      <button type="button" class="btn btn-primary" id="publishBtn">⬆️ Publish</button>
      <button type="button" class="btn btn-danger" id="discardBtn">Discard changes</button>
    </div>
  </div>
</div>
```

- [ ] **Step 3: Add admin JS** after the data-layer functions:

```js
// ----- Admin mode -----
function renderTechAdmin(){}    // implemented in Task 4
function renderStationAdmin(){} // implemented in Task 5

function updateAdminChrome(){
  document.getElementById("adminDirty").hidden = !isDirty();
  document.getElementById("publishBtn").disabled = !localStorage.getItem(LS_TOKEN);
}

function renderAdmin(){
  renderTechAdmin();
  renderStationAdmin();
  updateAdminChrome();
}

function markChanged(){
  saveStored(LS_LOCAL, localData);
  renderAll();
  renderAdmin();
}

function openAdmin(){
  document.getElementById("adminPanel").hidden = false;
  renderAdmin();
}

function closeAdmin(){
  document.getElementById("adminPanel").hidden = true;
}

document.getElementById("adminClose").addEventListener("click", closeAdmin);

// 5 taps on the title within 3 s opens admin
let titleTaps = 0, titleTapTimer = null;
document.querySelector(".title").addEventListener("click", () => {
  titleTaps++;
  clearTimeout(titleTapTimer);
  titleTapTimer = setTimeout(() => { titleTaps = 0; }, 3000);
  if (titleTaps >= 5){ titleTaps = 0; openAdmin(); }
});
```

- [ ] **Step 4: Verify in browser.** Reload the local server page:
  - 5 quick taps on "Faregate Inspection" opens the panel; 4 taps then a pause does not.
  - ✕ Close returns to the form; the form is unchanged behind it.
  - Publish button is disabled (no token saved yet); no "unpublished" badge shows.

- [ ] **Step 5: Commit**

```bash
git add faregate.html
git commit -m "feat: hidden admin panel skeleton with 5-tap gesture"
```

---

### Task 4: Techs add/edit/delete

**Files:**
- Modify: `faregate.html` — replace the `renderTechAdmin` stub; add listeners after the gesture code

**Interfaces:**
- Consumes: `localData`, `markChanged()`, `showToast()` from Tasks 2–3; `#techList`, `#techNew`, `#techAdd`.
- Produces: working `renderTechAdmin()`. Rows are `.admin-row` elements that are DIRECT children of `#techList`, each containing an `.admin-row-head` (Task 6's drag code depends on this structure).

- [ ] **Step 1: Implement `renderTechAdmin`** (replace the empty stub):

```js
function renderTechAdmin(){
  const list = document.getElementById("techList");
  list.innerHTML = "";
  localData.techs.forEach((t, i) => {
    const row  = document.createElement("div");
    row.className = "admin-row";
    const head = document.createElement("div");
    head.className = "admin-row-head";

    const name = document.createElement("span");
    name.className = "admin-row-name";
    name.textContent = t;

    const edit = document.createElement("button");
    edit.type = "button";
    edit.textContent = "✏️";
    edit.addEventListener("click", () => {
      const v = prompt("Edit tech name", t);
      if (v && v.trim()){ localData.techs[i] = v.trim(); markChanged(); }
    });

    const del = document.createElement("button");
    del.type = "button";
    del.textContent = "🗑";
    del.addEventListener("click", () => {
      if (confirm(`Delete ${t}?`)){ localData.techs.splice(i, 1); markChanged(); }
    });

    head.append(name, edit, del);
    row.appendChild(head);
    list.appendChild(row);
  });
}
```

- [ ] **Step 2: Wire the Add control** (add after the gesture code):

```js
document.getElementById("techAdd").addEventListener("click", () => {
  const input = document.getElementById("techNew");
  const v = input.value.trim();
  if (!v) return;
  localData.techs.push(v);
  input.value = "";
  markChanged();
});
```

- [ ] **Step 3: Verify in browser.**
  - Admin panel lists all 17 techs.
  - Add "Test T (99999)" → appears at the bottom of the admin list AND at the bottom of the form's Tech dropdown; "● unpublished" badge appears.
  - Edit it to "Test U (99999)" → both places update.
  - Delete it (confirm dialog appears) → gone from both; badge disappears (data matches remote again).
  - Reload the page → no stray changes persist (you deleted your test entry).

- [ ] **Step 4: Commit**

```bash
git add faregate.html
git commit -m "feat: tech add/edit/delete in admin panel"
```

---

### Task 5: Stations and arrays add/edit/delete

**Files:**
- Modify: `faregate.html` — replace the `renderStationAdmin` stub; add the station-add listener

**Interfaces:**
- Consumes: `localData`, `markChanged()`, `showToast()`; `#stationList`, `#stNewName`, `#stNewId`, `#stNewLine`, `#stAdd`.
- Produces:
  - working `renderStationAdmin()` with one expandable `.admin-row` per station (direct child of `#stationList`, header in `.admin-row-head`, expanded content in `.station-detail`)
  - inside an expanded station, the arrays render as their own `.admin-list` (id `arrayList-<stationId>`) of `.admin-row`s with `.admin-row-head` — Task 6 attaches drag to it
  - `let expandedStationId` — id of the currently expanded station or `null`
  - `const LINE_EMOJIS = ["🔴","🟣","🔵","🟢","🟡","🟠","⚪","⚫"]`

- [ ] **Step 1: Implement `renderStationAdmin`** (replace the empty stub):

```js
const LINE_EMOJIS = ["🔴","🟣","🔵","🟢","🟡","🟠","⚪","⚫"];
let expandedStationId = null;

function renderStationAdmin(){
  const list = document.getElementById("stationList");
  list.innerHTML = "";
  localData.stations.forEach((s, i) => {
    const row  = document.createElement("div");
    row.className = "admin-row";
    const head = document.createElement("div");
    head.className = "admin-row-head";

    const name = document.createElement("span");
    name.className = "admin-row-name";
    name.textContent = `${s.line} ${s.name}`;

    const toggle = document.createElement("button");
    toggle.type = "button";
    toggle.textContent = expandedStationId === s.id ? "▲" : "▼";
    const doToggle = () => {
      expandedStationId = expandedStationId === s.id ? null : s.id;
      renderStationAdmin();
    };
    toggle.addEventListener("click", doToggle);
    name.addEventListener("click", doToggle);

    head.append(name, toggle);
    row.appendChild(head);

    if (expandedStationId === s.id) row.appendChild(buildStationDetail(s, i));
    list.appendChild(row);
  });
}

function buildStationDetail(s, i){
  const detail = document.createElement("div");
  detail.className = "station-detail";

  // Name / ID / line editors
  const nameWrap = document.createElement("div");
  nameWrap.innerHTML = "<label>Name</label>";
  const nameIn = document.createElement("input");
  nameIn.value = s.name;
  nameIn.addEventListener("change", () => {
    if (nameIn.value.trim()){ s.name = nameIn.value.trim(); markChanged(); }
  });
  nameWrap.appendChild(nameIn);

  const idWrap = document.createElement("div");
  idWrap.innerHTML = "<label>Station ID</label>";
  const idIn = document.createElement("input");
  idIn.value = s.id;
  idIn.addEventListener("change", () => {
    const v = idIn.value.trim();
    if (!v){ idIn.value = s.id; return; }
    if (localData.stations.some(o => o !== s && o.id === v)){
      showToast("❌ Station ID already in use");
      idIn.value = s.id;
      return;
    }
    if (expandedStationId === s.id) expandedStationId = v;
    s.id = v;
    markChanged();
  });
  idWrap.appendChild(idIn);

  const lineWrap = document.createElement("div");
  lineWrap.innerHTML = "<label>Line</label>";
  const lineSel = document.createElement("select");
  LINE_EMOJIS.forEach(e => {
    const o = document.createElement("option");
    o.textContent = e;
    lineSel.appendChild(o);
  });
  lineSel.value = s.line;
  lineSel.addEventListener("change", () => { s.line = lineSel.value; markChanged(); });
  lineWrap.appendChild(lineSel);

  detail.append(nameWrap, idWrap, lineWrap);

  // Arrays list
  const arrLabel = document.createElement("label");
  arrLabel.textContent = "Arrays";
  detail.appendChild(arrLabel);

  const arrList = document.createElement("div");
  arrList.className = "admin-list";
  arrList.id = `arrayList-${s.id}`;
  s.arrays.forEach((a, ai) => {
    const aRow  = document.createElement("div");
    aRow.className = "admin-row";
    const aHead = document.createElement("div");
    aHead.className = "admin-row-head";

    const aName = document.createElement("span");
    aName.className = "admin-row-name";
    aName.textContent = a;

    const aEdit = document.createElement("button");
    aEdit.type = "button";
    aEdit.textContent = "✏️";
    aEdit.addEventListener("click", () => {
      const v = prompt("Edit array name", a);
      if (v && v.trim()){ s.arrays[ai] = v.trim(); markChanged(); }
    });

    const aDel = document.createElement("button");
    aDel.type = "button";
    aDel.textContent = "🗑";
    aDel.addEventListener("click", () => {
      if (confirm(`Delete array ${a}?`)){ s.arrays.splice(ai, 1); markChanged(); }
    });

    aHead.append(aName, aEdit, aDel);
    aRow.appendChild(aHead);
    arrList.appendChild(aRow);
  });
  detail.appendChild(arrList);

  // Add array
  const addWrap = document.createElement("div");
  addWrap.className = "admin-add";
  const addIn = document.createElement("input");
  addIn.placeholder = "New array name";
  const addBtn = document.createElement("button");
  addBtn.type = "button";
  addBtn.className = "btn btn-primary";
  addBtn.textContent = "Add";
  addBtn.addEventListener("click", () => {
    const v = addIn.value.trim();
    if (!v) return;
    s.arrays.push(v);
    markChanged();
  });
  addWrap.append(addIn, addBtn);
  detail.appendChild(addWrap);

  // Delete station
  const delBtn = document.createElement("button");
  delBtn.type = "button";
  delBtn.className = "btn btn-danger";
  delBtn.textContent = "🗑 Delete station";
  delBtn.addEventListener("click", () => {
    if (confirm(`Delete ${s.name} and its ${s.arrays.length} array(s)?`)){
      localData.stations.splice(i, 1);
      expandedStationId = null;
      markChanged();
    }
  });
  detail.appendChild(delBtn);

  return detail;
}
```

- [ ] **Step 2: Wire station Add** (after the techAdd listener):

```js
document.getElementById("stAdd").addEventListener("click", () => {
  const name = document.getElementById("stNewName").value.trim();
  const id   = document.getElementById("stNewId").value.trim();
  const line = document.getElementById("stNewLine").value;
  if (!name || !id){ showToast("Name and ID required"); return; }
  if (localData.stations.some(s => s.id === id)){
    showToast("❌ Station ID already in use");
    return;
  }
  localData.stations.push({ id, line, name, arrays: [] });
  document.getElementById("stNewName").value = "";
  document.getElementById("stNewId").value = "";
  markChanged();
});
```

- [ ] **Step 3: Verify in browser.**
  - All 26 stations listed; tapping one expands name/ID/line editors and its arrays.
  - Rename a station → form's Station dropdown updates immediately; badge shows.
  - Change an ID to one already in use (e.g., set Norwalk's ID to `124`) → toast, value reverts.
  - Add an array to a station → picking that station on the form shows it in the Array dropdown.
  - Add station "Test Station" ID `9999` line 🟡 → appears in both lists. Adding another with ID `9999` → rejected with toast.
  - Delete "Test Station" (confirm mentions its array count) → gone everywhere. Undo your other test edits (or Discard once Task 7 lands — for now re-edit them back).
  - Reload → dropdowns still correct.

- [ ] **Step 4: Commit**

```bash
git add faregate.html
git commit -m "feat: station and array management in admin panel"
```

---

### Task 6: Long-press drag reordering

**Files:**
- Modify: `faregate.html` — add the reorder helper, attach it to the tech and station lists, and attach it to each array sublist inside `buildStationDetail`

**Interfaces:**
- Consumes: `.admin-row` / `.admin-row-head` structure from Tasks 4–5; `markChanged()`; `arrList` variable inside `buildStationDetail`.
- Produces: `makeReorderable(listEl, onReorder)` and `moveItem(arr, from, to)`.

- [ ] **Step 1: Add the helper** (after `markChanged`):

```js
function moveItem(arr, from, to){
  const [m] = arr.splice(from, 1);
  arr.splice(to, 0, m);
}

// Long-press (~400 ms) a row to lift it, drag to reorder, release to drop.
function makeReorderable(listEl, onReorder){
  let pressTimer = null, dragging = null, startIndex = -1, startY = 0;

  listEl.addEventListener("pointerdown", e => {
    const row = e.target.closest(".admin-row");
    if (!row || row.parentElement !== listEl) return;      // ignore rows of nested lists
    if (!e.target.closest(".admin-row-head")) return;      // only the header lifts
    if (e.target.closest("button, input, select")) return; // not from controls
    startY = e.clientY;
    pressTimer = setTimeout(() => {
      dragging = row;
      startIndex = [...listEl.children].indexOf(row);
      row.classList.add("dragging");
      if (navigator.vibrate) navigator.vibrate(30);
    }, 400);
  });

  listEl.addEventListener("pointermove", e => {
    if (!dragging){
      if (Math.abs(e.clientY - startY) > 10) clearTimeout(pressTimer); // it's a scroll
      return;
    }
    const others = [...listEl.children].filter(r => r !== dragging);
    const after = others.find(r => {
      const rect = r.getBoundingClientRect();
      return e.clientY < rect.top + rect.height / 2;
    });
    if (after) listEl.insertBefore(dragging, after);
    else listEl.appendChild(dragging);
  });

  // Block page scroll and the long-press context menu only while dragging
  listEl.addEventListener("touchmove", e => { if (dragging) e.preventDefault(); }, { passive: false });
  listEl.addEventListener("contextmenu", e => {
    // keep long-press paste working inside text inputs
    if (!e.target.closest("input, select, textarea")) e.preventDefault();
  });

  const finish = () => {
    clearTimeout(pressTimer);
    if (!dragging) return;
    dragging.classList.remove("dragging");
    const endIndex = [...listEl.children].indexOf(dragging);
    const from = startIndex;
    dragging = null;
    if (endIndex !== from) onReorder(from, endIndex);
  };
  listEl.addEventListener("pointerup", finish);
  listEl.addEventListener("pointercancel", finish);
}
```

- [ ] **Step 2: Attach to the persistent lists** (once, right after the helper — these list elements are never replaced, only their children):

```js
makeReorderable(document.getElementById("techList"), (from, to) => {
  moveItem(localData.techs, from, to);
  markChanged();
});

makeReorderable(document.getElementById("stationList"), (from, to) => {
  moveItem(localData.stations, from, to);
  markChanged();
});
```

- [ ] **Step 3: Attach to array sublists.** In `buildStationDetail`, right after `detail.appendChild(arrList);`, add:

```js
makeReorderable(arrList, (from, to) => {
  moveItem(s.arrays, from, to);
  markChanged();
});
```

- [ ] **Step 4: Verify in browser** (mouse, then DevTools device-toolbar touch emulation):
  - Press-and-hold a tech row ~half a second → it lifts (shadow/highlight); drag up/down reorders; release → order sticks and the form's Tech dropdown shows the new order; badge shows.
  - A quick click/tap on a row does NOT lift it; edit/delete buttons still work.
  - In touch emulation: slow scroll over the lists still scrolls the panel (no accidental lifts while moving); long-press then drag reorders without scrolling the page.
  - Reorder stations → Station dropdown order follows. Expand a station with multiple arrays (e.g., Willowbrook), reorder its arrays → Array dropdown follows. Dragging an array row never moves station rows.
  - Restore the original order (or note that Discard arrives in Task 7).

- [ ] **Step 5: Commit**

```bash
git add faregate.html
git commit -m "feat: long-press drag reordering for techs, stations, and arrays"
```

---

### Task 7: Token settings, Publish, and Discard

**Files:**
- Modify: `faregate.html` — add GitHub API code and wire the Settings/Publish/Discard controls

**Interfaces:**
- Consumes: `localData`, `remoteData`, `LS_TOKEN`, `saveStored`, `deepCopy`, `fetchRemoteData`, `renderAll`, `renderAdmin`, `updateAdminChrome`, `showToast`; elements `#tokenInput`, `#tokenSave`, `#tokenTest`, `#tokenForget`, `#publishBtn`, `#discardBtn`.
- Produces: `publishData()`, `testConnection()` wired to the UI.

- [ ] **Step 1: Add the GitHub API code** (after the reorder attachments):

```js
// ----- GitHub publishing -----
const REPO_API = "https://api.github.com/repos/gooniesplayground/Faregate-Inspection-/contents/data.json";

function ghHeaders(token){
  return { "Authorization": `Bearer ${token}`, "Accept": "application/vnd.github+json" };
}

function b64EncodeUtf8(str){
  const bytes = new TextEncoder().encode(str);
  let bin = "";
  bytes.forEach(b => bin += String.fromCharCode(b));
  return btoa(bin);
}

async function testConnection(){
  const token = localStorage.getItem(LS_TOKEN);
  if (!token){ showToast("Save a token first"); return; }
  try {
    const res = await fetch(REPO_API, { headers: ghHeaders(token) });
    if (res.ok) showToast("✅ Token works");
    else if (res.status === 401 || res.status === 403) showToast("❌ Token invalid or expired");
    else showToast(`❌ GitHub says ${res.status}`);
  } catch (e) {
    showToast("❌ No connection");
  }
}

async function publishData(){
  const token = localStorage.getItem(LS_TOKEN);
  if (!token){ showToast("Save a token in Settings first"); return; }
  const btn = document.getElementById("publishBtn");
  btn.disabled = true;
  try {
    const head = await fetch(`${REPO_API}?t=${Date.now()}`, { headers: ghHeaders(token) });
    if (head.status === 401 || head.status === 403){ showToast("❌ Token invalid or expired — check Settings"); return; }
    if (!head.ok){ showToast(`❌ GitHub says ${head.status}`); return; }
    const sha = (await head.json()).sha;

    const res = await fetch(REPO_API, {
      method: "PUT",
      headers: ghHeaders(token),
      body: JSON.stringify({
        message: "Update data.json via admin mode",
        content: b64EncodeUtf8(JSON.stringify(localData, null, 2) + "\n"),
        sha
      })
    });

    if (res.ok){
      remoteData = deepCopy(localData);
      saveStored(LS_REMOTE, remoteData);
      renderAdmin();
      showToast("✅ Published! Live for everyone in ~1 minute");
    } else if (res.status === 409){
      await fetchRemoteData();
      renderAdmin();
      showToast("⚠️ Someone else published first — review and publish again");
    } else if (res.status === 401 || res.status === 403){
      showToast("❌ Token invalid or expired — check Settings");
    } else {
      showToast(`❌ Publish failed (${res.status})`);
    }
  } catch (e) {
    showToast("❌ No connection — changes kept locally");
  } finally {
    btn.disabled = false;
    updateAdminChrome();
  }
}
```

- [ ] **Step 2: Wire the controls:**

```js
document.getElementById("tokenSave").addEventListener("click", () => {
  const input = document.getElementById("tokenInput");
  const v = input.value.trim();
  if (!v){ showToast("Paste a token first"); return; }
  localStorage.setItem(LS_TOKEN, v);
  input.value = "";
  updateAdminChrome();
  showToast("🔑 Token saved on this device");
});

document.getElementById("tokenForget").addEventListener("click", () => {
  localStorage.removeItem(LS_TOKEN);
  updateAdminChrome();
  showToast("Token forgotten");
});

document.getElementById("tokenTest").addEventListener("click", testConnection);
document.getElementById("publishBtn").addEventListener("click", publishData);

document.getElementById("discardBtn").addEventListener("click", () => {
  if (!isDirty()){ showToast("No unpublished changes"); return; }
  if (confirm("Discard all unpublished changes?")){
    localData = deepCopy(remoteData);
    saveStored(LS_LOCAL, localData);
    renderAll();
    renderAdmin();
    showToast("Changes discarded");
  }
});
```

- [ ] **Step 3: Verify locally (no real token yet).**
  - Publish is disabled with no token. Save token `dummy` → Publish enables; Test connection → "❌ Token invalid or expired". Publish → same error, nothing committed.
  - Make an edit (badge shows) → Discard → confirm → edits gone, badge gone, dropdowns back to published state.
  - Forget token → Publish disables again.
  - Note: the `fetch` to api.github.com from `localhost` works (GitHub API sends CORS `*`).

- [ ] **Step 4: Commit**

```bash
git add faregate.html
git commit -m "feat: GitHub token settings, publish, and discard"
```

---

### Task 8: README token instructions, push, and end-to-end verification

**Files:**
- Modify: `README.md` — add an "Admin mode" section

**Interfaces:**
- Consumes: everything prior.

- [ ] **Step 1: Add to `README.md`:**

```markdown
## Admin mode

Open the app and tap the **Faregate Inspection** title 5 times quickly.
From the admin panel you can add, edit, delete, and reorder (press-and-hold,
then drag) techs, stations, and each station's arrays. Changes apply to your
device immediately and go live for everyone when you tap **Publish**
(allow ~1 minute for GitHub Pages to update).

### One-time token setup (admins only)

Publishing commits `data.json` to this repo, which requires a GitHub token:

1. On github.com (logged in as an account with write access to this repo) go to
   **Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token**.
2. Name it `faregate-admin`, set expiration (e.g., 1 year).
3. **Repository access:** "Only select repositories" → `Faregate-Inspection-`.
4. **Permissions → Repository permissions → Contents: Read and write.** Leave everything else "No access".
5. Generate, copy the token (starts with `github_pat_`).
6. In the app: admin panel → **Settings** → paste the token → **Save**, then **Test connection**.

The token is stored only in your browser's localStorage on that device —
never in the repo. If the token expires or leaks, revoke it on github.com
and generate a new one.
```

- [ ] **Step 2: Commit the README**

```bash
git add README.md
git commit -m "docs: admin mode and token setup instructions"
```

- [ ] **Step 3: Ask the user before pushing.** Pushing deploys to every user. Ask; on yes:

```bash
git push origin main
```

- [ ] **Step 4: End-to-end verification on the live site** (`https://gooniesplayground.github.io/Faregate-Inspection-/faregate`), coordinating with the user since only they can create the token:
  - Page loads, dropdowns populated from `data.json`.
  - User creates the fine-grained token per README, saves it in Settings, Test connection → "✅ Token works".
  - User makes a trivial edit (e.g., add tech "Test T (00001)") and Publishes → check the repo: a new commit "Update data.json via admin mode" exists on `data.json`.
  - After ~1 minute, reload on a second device/incognito window → the new tech appears.
  - User deletes the test tech and publishes again to restore the real list.
