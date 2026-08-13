# Open-Chat Camera Button Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Camera button opens the configured Google Chat space; Save/Camera labels switch to "Submit to Google Chat" / "Open Google Chat" when chat is configured.

**Architecture:** Same pattern as chatWebhook: new `chatSpaceLink` string in data + admin Apply field, plus an `updateActionButtons()` label/behavior switch called from `renderAll()`.

**Tech Stack:** Vanilla JS in faregate.html (+ data.json, README). Manual browser verification.

**Spec:** `docs/superpowers/specs/2026-08-12-chat-space-link-design.md`

## Global Constraints

- Only `faregate.html`, `data.json`, `README.md` change.
- `chatSpaceLink`: empty = off; non-empty must start with `https://chat.google.com/`; invalid Apply toasts exactly `❌ Not a Google Chat link` and changes nothing.
- Unconfigured devices: labels exactly `Save` and `📷 Camera`, behavior byte-for-byte as today.
- All mutations via `markChanged()`; `isValidData` and the publish flow untouched. No push (controller handles).

---

### Task 1: chatSpaceLink + dynamic buttons

**Files:**
- Modify: `data.json`, `faregate.html`

**Interfaces:**
- Consumes (all exist): `DEFAULT_DATA`, `normalizeData`, `localData`, `markChanged`, `renderAdmin`, `renderAll`, `showToast`, `#saveBtn`, `#cameraBtn`, `#cameraInput`, the Settings section (contains `#webhookInput` row), the `#webhookApply` listener.
- Produces: data key `chatSpaceLink`, elements `#spaceLinkInput`, `#spaceLinkApply`, function `updateActionButtons()`.

- [ ] **Step 1: Data.** In `data.json`, add `"chatSpaceLink": "",` immediately after the `"chatWebhook"` line. Same addition in `DEFAULT_DATA` in faregate.html.

- [ ] **Step 2: Normalize.** In `normalizeData`, after the chatWebhook coercion, add:

```js
  if (typeof d.chatSpaceLink !== "string"){
    d.chatSpaceLink = "";
  }
```

- [ ] **Step 3: Admin field.** In the Settings section HTML, insert ABOVE the webhook `.admin-add` row:

```html
      <div class="admin-add">
        <input id="spaceLinkInput" placeholder="Google Chat space link (optional)" autocomplete="off" />
        <button type="button" class="btn btn-primary" id="spaceLinkApply">Apply</button>
      </div>
      <p style="font-size:11px;margin:4px 0 8px;color:#64748b">
        The camera button opens this chat space so techs can post photos
        with Chat's own camera. In Google Chat: space menu → Copy link.
        Apply, then Publish. Leave empty for the normal camera.
      </p>
```

- [ ] **Step 4: Wire the field.** In `renderAdmin`, next to the webhookInput line, add:

```js
  document.getElementById("spaceLinkInput").value = localData.chatSpaceLink || "";
```

Next to the `webhookApply` listener add:

```js
document.getElementById("spaceLinkApply").addEventListener("click", () => {
  const v = document.getElementById("spaceLinkInput").value.trim();
  if (v && !v.startsWith("https://chat.google.com/")){
    showToast("❌ Not a Google Chat link");
    return;
  }
  localData.chatSpaceLink = v;
  markChanged();
  showToast(v ? "💬 Camera opens chat — Publish to roll out" : "Camera back to normal — Publish to roll out");
});
```

- [ ] **Step 5: Dynamic buttons.** Add after `renderChecklist`:

```js
function updateActionButtons(){
  document.getElementById("saveBtn").textContent =
    localData.chatWebhook ? "Submit to Google Chat" : "Save";
  document.getElementById("cameraBtn").textContent =
    localData.chatSpaceLink ? "Open Google Chat" : "📷 Camera";
}
```

Add `updateActionButtons();` as the last line of `renderAll()`.

In the `#cameraBtn` click listener, replace the body with:

```js
      if (localData.chatSpaceLink){
        window.open(localData.chatSpaceLink, "_blank");
        return;
      }
      document.getElementById("cameraInput").click();
```

(The `#cameraInput` change listener and everything else stays untouched.)

- [ ] **Step 6: Verify in browser** (serve :8080, browser tools, 5-tap admin):
  - Unconfigured: buttons read exactly `Save` and `📷 Camera`; Camera click triggers the hidden file input (spy on cameraInput.click), Save unchanged.
  - Admin: `https://example.com` in space-link Apply → exact toast `❌ Not a Google Chat link`, no dirty. `https://chat.google.com/room/TESTROOM` → Apply → dirty badge, form buttons now read `Open Google Chat` (webhook still empty so Save still reads `Save`).
  - Stub `window.open` via javascript_tool; Camera click → open called with the link, file input NOT clicked.
  - Set a fake webhook too (`https://chat.googleapis.com/v1/spaces/T/messages?key=k&token=t`) → Save button reads `Submit to Google Chat`. Check the 3-button row renders acceptably with the longer labels (no overflow clipping; wrapping OK) — screenshot or measure.
  - Discard → labels revert to `Save` / `📷 Camera`; badge gone. Clean localStorage, console clean.

- [ ] **Step 7: Commit** — `git add data.json faregate.html`, message `feat: camera button opens Google Chat space; dynamic chat button labels`

---

### Task 2: README

**Files:**
- Modify: `README.md`

- [ ] **Step 1:** Replace the heading `### Optional: send inspections to a Google Chat space` and its intro sentence with:

```markdown
### Optional: connect the app to Google Chat

Two separate hookups, each optional:

- **Webhook URL** — every **Save** posts the inspection (as text) into the
  space automatically. The button label changes to "Submit to Google Chat".
- **Space link** — the camera button becomes **"Open Google Chat"** and
  jumps straight into the space, where techs use Chat's own camera to post
  photos.
```

Keep the existing numbered webhook steps, then append after them:

```markdown
For the space link: in Google Chat, open the space → click the space
name → **Copy link to this space** (on some versions it's in the ⋮ menu).
Paste it into admin **Settings → Google Chat space link** → **Apply** →
**Publish**.
```

- [ ] **Step 2: Commit** — message `docs: Google Chat space link instructions`
