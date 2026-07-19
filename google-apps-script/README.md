# RFQ backend (Google Apps Script)

`Code.gs` handles the RFQ form on the site: appends submissions to a
Google Sheet and emails a notification (see `NOTIFY_EMAIL` at the top
of the file). It's linked to the live Apps Script project via
[`clasp`](https://github.com/google/clasp) — `.clasp.json` in this
folder points at the script (`scriptId`), so changes can be pushed
directly instead of pasting code into the browser editor.

## Making a change live

Editing `Code.gs` and running `clasp push` is **not enough** — it only
updates the editor's saved source. The live `/exec` URL embedded in
`index.html` (`RFQ_ENDPOINT`) stays pinned to whatever *version* it
was last deployed at. Three steps, every time:

```bash
cd google-apps-script

# 1. Save the edited code
clasp push

# 2. Snapshot it as a new immutable version
clasp create-version "short description of the change"
# -> prints: Created version <N>

# 3. Point the EXISTING deployment at that new version
clasp redeploy <deploymentId> -V <N> -d "short description"
```

`<deploymentId>` is the long `AKfycb...` string — it's the same one
that appears in the live URL
(`https://script.google.com/macros/s/<deploymentId>/exec`), i.e. the
current value of `RFQ_ENDPOINT` in `index.html`. Find it (or confirm
which deployment is active) with:

```bash
clasp list-deployments
```

This lists every deployment ever created for the project, each
tagged with the version it's currently serving (e.g. `@7`). The one
matching the URL in `index.html` is the one to `redeploy`.

## Redeploy vs. deploy — don't mix these up

- `clasp redeploy <deploymentId> -V <N>` — updates an **existing**
  deployment to serve a new version. **The `/exec` URL does not
  change.** This is what you want for every routine code change.
- `clasp create-deployment` (`clasp deploy`) — creates a **brand
  new** deployment with a **different** `/exec` URL. Only use this
  if you actually need a new URL; if you do, `RFQ_ENDPOINT` in
  `index.html` must be updated to match, or the live site keeps
  talking to the old one.

## Verifying a change actually went live

Since the deploy step is easy to get wrong, always confirm with a
real request against the exact URL from `RFQ_ENDPOINT`:

```bash
curl -s -L -d '{"fullName":"Test","email":"test@example.com","phone":"+91 9999999999","quantity":"test","sector":"Other","details":"verify deploy"}' \
  -H "Content-Type: text/plain;charset=utf-8" \
  "https://script.google.com/macros/s/<deploymentId>/exec"
```

A `{"status":"ok"}` response confirms the endpoint is reachable and
ran without throwing — check the Sheet / inbox to confirm the new
code's behavior (e.g. a changed `NOTIFY_EMAIL`) took effect, since a
stale deployment would also return `{"status":"ok"}` using the *old*
code.

## Design notes (why the code looks like this)

- **Honeypot** (`data.company_website`): a hidden form field real
  users never see or fill. If it's non-empty, the request is treated
  as a bot and dropped — but still returns a normal-looking success
  response, so the bot doesn't notice and adapt.
- **Size limit** (`MAX_FILE_BYTES`, 25MB): checked *before* touching
  the Sheet or Mail, matching Gmail's attachment cap. Rejecting late
  would leave a submission logged in the Sheet with no notification
  ever sent.
- **No Drive storage**: uploaded files are attached to the
  notification email only, never written to Drive — by design
  (avoids unbounded storage growth from a public form).
