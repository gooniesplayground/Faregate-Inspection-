# Save = Chat OR Phone Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Save becomes either/or: with no webhook configured it saves the PNG to the phone exactly as today; with a webhook configured it ONLY sends the text summary to Google Chat (no PNG) — except when the send fails, where it falls back to the PNG download so no inspection is ever lost.

**Architecture:** Extract the existing html2canvas/download block into `saveScreenshotToPhone()` unchanged, then branch the Save click handler on `localData.chatWebhook`.

**Spec basis:** user decision 2026-08-12 on top of `docs/superpowers/specs/2026-08-12-chat-webhook-design.md`; the failure-fallback is a deliberate safety addition.

## Global Constraints

- Everything stays in `faregate.html`.
- No-webhook path must be byte-for-byte today's behavior (`📸 Saving...` → `✅ Saved to Downloads!` / existing failure toast).
- Webhook path must not run html2canvas at all when the send succeeds.
- `sendSummaryToChat`, admin UI, publish flow untouched.
- No push (controller handles).

---

### Task 1: Branch the Save button

**Files:**
- Modify: `faregate.html` — the `#saveBtn` click handler only

**Interfaces:**
- Consumes: `localData.chatWebhook`, `sendSummaryToChat()`, `autoFillTimesIfBlank()`, `showToast()`, existing screenshot code.
- Produces: `saveScreenshotToPhone()`.

- [ ] **Step 1: Extract.** Take the entire body of the current save handler AFTER the `autoFillTimesIfBlank();` line (filename computation, hiding actions, notes swap, `showToast("📸 Saving...")`, the `setTimeout`/html2canvas block with its success and catch paths) and move it verbatim into a new function `function saveScreenshotToPhone(){ ... }` placed just before the save handler. Inside its html2canvas success path, replace the Task-1 chat block (`sendSummaryToChat().then(...)`) with the original plain `showToast("✅ Saved to Downloads!");` — the chat decision now lives outside this function.

- [ ] **Step 2: New handler.** The save handler becomes:

```js
    // Save button — chat-only when a webhook is configured, phone-only otherwise
    document.getElementById("saveBtn").addEventListener("click", () => {
      autoFillTimesIfBlank();

      if (localData.chatWebhook){
        showToast("📤 Sending to chat...");
        sendSummaryToChat().then(chat => {
          if (chat === "sent"){
            showToast("✅ Sent to chat!");
          } else {
            showToast("⚠️ Chat send failed — saving to phone instead");
            saveScreenshotToPhone();
          }
        });
      } else {
        saveScreenshotToPhone();
      }
    });
```

- [ ] **Step 3: Verify in browser** (serve :8080, browser tools, monkey-patch fetch for chat.googleapis.com):
  - No webhook: Save → `📸 Saving...` then `✅ Saved to Downloads!`, PNG download triggered, zero chat fetches, zero html2canvas surprises.
  - Webhook set (TEST url) + mocked `ok:true`: Save → `📤 Sending to chat...` then `✅ Sent to chat!`; NO png download, html2canvas NOT invoked (spy on window.html2canvas), body JSON `text` contains "FAREGATE INSPECTION"; form buttons/notes untouched (no hide/restore ran).
  - Webhook set + mocked `ok:false` (and separately a rejecting fetch): warning toast, then PNG downloads via the fallback; buttons/notes restored properly afterward.
  - Restore clean state (empty webhook, localStorage reset, badge hidden). Console clean.

- [ ] **Step 4: Commit** — `git add faregate.html`, message `feat: Save sends to chat only when webhook set, phone only otherwise`
