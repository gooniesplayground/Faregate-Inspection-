# Skip Google's Account Picker — Design

**Date:** 2026-08-12
**Extends:** chat-space-link design (live)

## Goal

"Open Google Chat" stops showing Google's account chooser: each tech
enters their work Gmail once per device, and the app appends
`authuser=<email>` to the space link so Google opens the right account
directly.

## Design

### Per-device storage

localStorage key `faregate.chatEmail` (constant `LS_CHAT_EMAIL`). Never
published, never synced — same trust model as the admin token and drafts.

### First-tap prompt

In the "Open Google Chat" branch of the camera-button handler: if no
stored email, `prompt("Enter your work Gmail once to skip Google's
account picker (Cancel to skip):")`. A value containing `@` is trimmed
and stored; then the link opens with the hint. Cancel/empty/invalid →
open the plain link this time and ask again on a future tap (a typo'd
non-email is never stored; invalid input shows toast
`❌ That doesn't look like an email`).

### Link building

`openChatLink()` builds: stored email → append
`authuser=<encodeURIComponent(email)>` using `&` when the link already
has a `?`, else `?`. No stored email → plain link. Still `window.open`
synchronously inside the click handler (popup-blocker safe).

### Correction path (Settings)

Admin Settings gains a per-device row under the space-link row: input
`#chatEmailInput` (prefilled from localStorage on renderAdmin) + Save
button `#chatEmailSave`. Saving empty clears the stored email (picker
returns / prompt will re-ask); saving a value requires `@`. Hint text
marks it as "this device only". No dirty badge, no `markChanged()` —
this is device state, not published data.

### Unchanged

Everything else — the shared `chatSpaceLink`, labels, webhook flow,
drafts, publish.

Caveat (accepted, documented): if the phone routes chat.google.com links
into the native Google Chat app, the app makes its own account choice
and the hint has no effect.

## Testing

Browser: no stored email + prompt returns work@example.com → opens
`<link>?authuser=work%40example.com` (stub window.open) and stores it;
next tap skips the prompt; prompt cancel → plain link, nothing stored,
re-asks next tap; invalid input → toast, nothing stored, plain link;
link that already contains `?` gets `&authuser=`; Settings field shows
the stored email, saving a new one changes the hint, saving empty clears
it; no dirty badge from any of this. Console clean.
