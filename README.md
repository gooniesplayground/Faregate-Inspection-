## Setup

Follow these steps once and you'll have your own copy of the app, running
under your own GitHub account, fully under your control. Takes about
10–15 minutes. Everything is free.

### Step 1: Create a GitHub account

1. Go to **[github.com/signup](https://github.com/signup)**.
2. Enter your email, create a password, pick a username.
   ⚠️ **Your username becomes part of the app's web address**
   (`https://YOUR-USERNAME.github.io/...`), so pick something you don't
   mind techs seeing.
3. Verify your email when GitHub asks.

### Step 2: Get your own copy of the app

1. While logged in, open this repository's page on github.com.
2. Click the **Fork** button (top right of the page).
3. On the next screen, just click **Create fork**.

You now have your own complete copy of the app under your account.
(Note: the copy must stay **Public** — GitHub's free plan only hosts
websites from public repositories.)

### Step 3: Turn on the website

1. In **your** new repository, click **Settings** (top menu of the repo).
2. In the left sidebar, click **Pages**.
3. Under **Branch**, change "None" to **`main`**, leave the folder as
   **`/ (root)`**, and click **Save**.
4. Wait 1–2 minutes, then refresh the page. A box appears at the top
   saying **"Your site is live at https://YOUR-USERNAME.github.io/REPO-NAME/"**.

### Step 4: Open your app

Your app's address is that link **plus `faregate` on the end**:

```text
https://YOUR-USERNAME.github.io/REPO-NAME/faregate
```

Open it on your phone. You should see the inspection form with all the
stations and techs. **This is the link you give your techs.**

### Step 5: Create your access token

The token is like a special password that lets the app save changes back
to your repository. Only admins need one.

1. On github.com, click your profile picture (top right) →
   **Settings**.
2. In the left sidebar, scroll to the bottom → **Developer settings**.
3. Click **Personal access tokens → Fine-grained tokens →
   Generate new token**.
4. Fill it in:
   - **Token name:** anything, e.g. `faregate-admin`
   - **Expiration:** click the dropdown and pick a custom date up to
     1 year out (when it expires, you'll repeat this step to make a
     new one)
   - **Repository access:** choose **"Only select repositories"**, then
     select your faregate repository from the list
   - **Permissions:** click **"+ Add permissions"**, find **Contents**
     in the list, select it, then change its dropdown to
     **"Read and write"**. (GitHub adds "Metadata: Read-only" by itself —
     that's normal.) Everything else stays "No access".
5. Click **Generate token**.
6. **Copy the token now** (it starts with `github_pat_`). GitHub shows it
   only this once. ⚠️ Treat it like a password — don't text or email it.

### Step 6: Put the token in the app

1. Open **your** app link (from Step 4) on the phone you'll admin from.
2. Tap the **Faregate Inspection** title 5 times → admin panel opens.
3. Scroll to **Settings**, paste the token, tap **Save**.
4. Tap **Test connection** — you should see **"✅ Token works"**.

The token is saved only in that phone's browser — it is never uploaded
anywhere. If you ever lose the phone, go back to GitHub's token page and
delete (revoke) the token.

### Step 7: Do one test publish

1. In the admin panel, add a fake tech named `Test Person`.
2. Tap **Publish**. You should see **"✅ Published!"**.
3. Wait a minute, then open the app on a second device (or a private
   browser window) — `Test Person` should appear in the Tech dropdown.
4. Delete `Test Person` and Publish again to clean up.

**You're done.** Give the techs the link from Step 4, and manage the lists
from your phone whenever things change.

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

---

## Good to know

- **The app auto-adapts to its home.** It figures out which repository to
  publish to from its own web address, so forks and renames need zero code
  changes.
- **Every published change is saved in the repository's history**
  (the repo's "Commits" page). If a bad edit ever gets published, nothing
  is lost — the previous version is always recoverable.
- **If Publish says "Token invalid or expired"**, the token has hit its
  expiration date — create a new one (Step 5) and save it in the app
  (Step 6).
- **If two admins publish at the same time**, the app detects it, refuses
  to overwrite the other person's change, and asks you to review and
  publish again.

---

## For admins: editing the lists

1. Open the app and tap the **Faregate Inspection** title **5 times
   quickly**. The admin panel opens.
2. Add ➕, edit ✏️, or delete 🗑 techs, stations, each station's arrays,
   and the checklist items.
3. To reorder a list: **press and hold** an item for half a second, then
   drag it up or down.
4. Changes show up on *your* device right away. When you're happy, tap
   **Publish** — within about a minute, everyone gets the update the next
   time they open the app.
5. Made a mess? Tap **Discard changes** to go back to the last published
   version.

Publishing requires a one-time token setup — see
[Step 5](#step-5-create-your-access-token) above. Without it, the admin
panel opens but Publish stays disabled.
