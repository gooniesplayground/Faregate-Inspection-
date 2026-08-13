# Send-to-Chat on Save Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Save posts the inspection summary text to a configurable Google Chat webhook automatically, alongside the existing PNG download.

**Architecture:** Follows the shipped pattern: new `chatWebhook` string in `data.json`/`DEFAULT_DATA` normalized for old caches, an admin Settings field to set it, and a post-download fetch in the Save handler using the existing (currently unused) `buildSummary()`.

**Tech Stack:** Vanilla JS in faregate.html. No test framework; manual browser verification with a monkey-patched fetch.

**Spec:** `docs/superpowers/specs/2026-08-12-chat-webhook-design.md`

## Global Constraints

- Everything stays in `faregate.html` + `data.json` (+ README in Task 2).
- Webhook value: empty string = off; non-empty must start with `https://chat.googleapis.com/v1/spaces/`; invalid Apply attempts toast exactly `❌ Not a Google Chat webhook URL` and change nothing.
- Save behavior with empty webhook must be byte-for-byte today's behavior (same toasts).
- All mutations via `markChanged()`. `isValidData` unchanged. No push (controller handles).

---

### Task 1: chatWebhook data + admin field + Save send

**Files:**
- Modify: `data.json`, `faregate.html`

**Interfaces:**
- Consumes (all exist): `DEFAULT_DATA`, `normalizeData`, `localData`, `markChanged`, `renderAdmin`, `showToast`, `buildSummary`, the Save handler (`#saveBtn` listener), admin Settings `<section>` (contains `#tokenInput`), `.admin-add` CSS.
- Produces: `data.json`/`DEFAULT_DATA` key `chatWebhook` (string), elements `#webhookInput`, `#webhookApply`, function `sendSummaryToChat()`.

- [ ] **Step 1: Data.** In `data.json`, add `"chatWebhook": "",` as a top-level key right after the `"techs"` array's closing bracket (i.e., before `"checklist"`). Add the same key in the same position to `DEFAULT_DATA` in faregate.html.

- [ ] **Step 2: Normalize.** In `normalizeData`, after the checklist coercion, add:

```js
  if (typeof d.chatWebhook !== "string"){
    d.chatWebhook = "";
  }
```

- [ ] **Step 3: Admin Settings field.** In the Settings `<section>` of the admin panel HTML, insert BEFORE the token `.admin-add` row:

```html
      <div class="admin-add">
        <input id="webhookInput" placeholder="Google Chat webhook URL (optional)" autocomplete="off" />
        <button type="button" class="btn btn-primary" id="webhookApply">Apply</button>
      </div>
      <p style="font-size:11px;margin:4px 0 8px;color:#64748b">
        Save sends each inspection to this chat. Paste the space's webhook
        URL, tap Apply, then Publish. Leave empty to turn off.
      </p>
```

- [ ] **Step 4: Wire the field.** In `renderAdmin`, before `updateAdminChrome()`, add:

```js
  document.getElementById("webhookInput").value = localData.chatWebhook || "";
```

Next to the `tokenSave` listener, add:

```js
document.getElementById("webhookApply").addEventListener("click", () => {
  const v = document.getElementById("webhookInput").value.trim();
  if (v && !v.startsWith("https://chat.googleapis.com/v1/spaces/")){
    showToast("❌ Not a Google Chat webhook URL");
    return;
  }
  localData.chatWebhook = v;
  markChanged();
  showToast(v ? "💬 Chat sending on — Publish to roll out" : "Chat sending off — Publish to roll out");
});
```

- [ ] **Step 5: Send on Save.** Add next to `buildSummary`:

```js
async function sendSummaryToChat(){
  if (!localData.chatWebhook) return "off";
  try {
    const res = await fetch(localData.chatWebhook, {
      method: "POST",
      headers: { "Content-Type": "application/json; charset=UTF-8" },
      body: JSON.stringify({ text: buildSummary() })
    });
    return res.ok ? "sent" : "failed";
  } catch (e) {
    return "failed";
  }
}
```

In the Save handler's html2canvas success path, replace the line
`showToast("✅ Saved to Downloads!");` with:

```js
            sendSummaryToChat().then(chat => {
              if (chat === "sent") showToast("✅ Saved + sent to chat!");
              else if (chat === "failed") showToast("⚠️ Saved. Chat send failed — send manually");
              else showToast("✅ Saved to Downloads!");
            });
```

Nothing else in the Save handler changes (screenshot failure path untouched).

- [ ] **Step 6: Verify in browser** (serve :8080, browser tools, 5-tap admin, override prompt/confirm as needed):
  - Empty webhook: Save → identical to today (`📸 Saving...` then `✅ Saved to Downloads!`), no fetch to chat.googleapis.com (spy on window.fetch to count).
  - Admin: paste `https://example.com/x` → Apply → exact toast `❌ Not a Google Chat webhook URL`, no dirty badge. Paste `https://chat.googleapis.com/v1/spaces/TEST/messages?key=k&token=t` → Apply → dirty badge on, webhookInput repopulates after re-render.
  - Save with webhook set + monkey-patched fetch returning `{ok:true}` for chat.googleapis.com: PNG still downloads AND toast `✅ Saved + sent to chat!`; the intercepted request body is JSON with a `text` field containing "FAREGATE INSPECTION".
  - Monkey-patched failure (`{ok:false}`): toast `⚠️ Saved. Chat send failed — send manually`, PNG still downloads.
  - Restore clean state (empty webhook, badge hidden, localStorage reset). Console clean.

- [ ] **Step 7: Commit** — `git add data.json faregate.html` , message `feat: auto-send inspection summary to Google Chat on Save`

---

### Task 2: README section

**Files:**
- Modify: `README.md`

- [ ] **Step 1:** After the "Step 7: Do one test publish" section and before "## Good to know", insert:

```markdown
### Optional: send inspections to a Google Chat space

With this on, every **Save** also posts the inspection (as text) into a
Google Chat space of your choice — automatically, no extra taps for techs.

1. In Google Chat, open the space. Click the **space name** at the top →
   **Apps & integrations** (on some accounts: **Manage webhooks**).
2. Under **Webhooks**, click **Add webhook**, name it (e.g. `Faregate`),
   and click **Save**. Copy the URL it shows.
   *(No Webhooks section? The space is on a personal Google account or
   your Workspace admin has disabled webhooks — this feature needs a
   Workspace space with webhooks allowed.)*
3. In the app: tap the title 5 times → **Settings** → paste the URL into
   the **Google Chat webhook URL** field → **Apply** → **Publish**.
4. Within a minute, every tech's Save button also sends to the chat. Do
   one test Save to confirm.

To turn it off: clear the field, **Apply**, **Publish**. If strangers ever
spam the space, delete the webhook in Google Chat, add a fresh one, and
repeat these steps.
```

- [ ] **Step 2: Commit** — message `docs: Google Chat webhook setup instructions`
