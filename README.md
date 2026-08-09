# Faregate-Inspection-

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