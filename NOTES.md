# QR Door Notify

Scan a QR code on the door → pick a reason (Visitor / Delivery / etc.) on a simple
web page → get a Pushover notification. No app needed for the visitor, no
account, nothing typed.

## Live setup (as of 2026-07-14)

- Door page: https://benny1979.github.io/qr_notify/
- `door-qr.png` in this repo is the printed QR code, already pointing at the URL above.
- Button config: a **secret** Gist (`7c25da9fc398fa08f7e7f028797305b5` — replaced
  the original public one once we realised a `reply` field might hold a phone
  number). Editing it directly in the Gist UI still works, but the easier way
  is the admin tool below.
- To edit buttons without touching the Gist UI or any code, use `gist_editor`
  (a separate reusable project, `G:\My Drive\Notes\AI\gist_editor\NOTES.md`):
  bookmark
  `https://benny1979.github.io/gist_editor/?gist=https://gist.githubusercontent.com/benny1979/7c25da9fc398fa08f7e7f028797305b5/raw/buttons.json&gistId=7c25da9fc398fa08f7e7f028797305b5&file=buttons.json`
  — password is the one saved in Bitwarden (see `gist_editor/NOTES.md`).
- Pipedream workflow `QR-door-notify` holds `PUSHOVER_TOKEN`/`PUSHOVER_USER` and
  is already deployed and wired into `index.html`.

```
QR code (printed once)
  -> GitHub Pages page (index.html)
       -> fetches button list from a Gist (buttons.json)
       -> on click, POSTs {title, message} to a Pipedream webhook
            -> Pipedream calls Pushover's API (holds the token, not the page)
                 -> notification on your phone
```

Editing the buttons later = editing the Gist in a browser. No code, no redeploy.

## One-time setup

### 1. GitHub repo + Pages
- Create a new GitHub repo (public is fine, nothing secret lives in it), push this folder.
- Repo Settings -> Pages -> deploy from `main` branch, root. Note the resulting URL,
  e.g. `https://<user>.github.io/qr_notify/`. This is what the printed QR code will point to.

### 2. Gist for button config
- Create a **secret** Gist (`gh gist create` without `--public`, or the "secret"
  option at gist.github.com) containing one file, `buttons.json` — secret
  because a button's `reply` text might contain personal details (e.g. a phone
  number) meant only for whoever scans that specific button, not for anyone
  who stumbles on the raw URL. Note secret just means unlisted, not
  access-controlled — anyone with the exact raw URL can still read it.

```json
[
  { "label": "Visitor",  "emoji": "🔔", "title": "Someone's at the door", "message": "A visitor is at the door.",
    "reply": "Thanks — I've been notified." },
  { "label": "Delivery", "emoji": "📦", "title": "Delivery", "message": "A delivery has arrived.",
    "reply": "I'm not in right now — please leave it on top of the bins, thank you." }
]
```

`reply` is optional — if a button omits it, the page falls back to a generic "Sent —
thank you." `reply` is what the *scanner* sees on-screen after tapping (e.g.
instructions for a delivery driver); `title`/`message` is what arrives in the
Pushover notification to you. Editing `reply` later (e.g. swapping the delivery
instructions between "leave on bins" and "call me") is just a Gist edit, same as
any other button change.

- Click the file's "Raw" button, copy that URL, dropping the commit-hash segment
  so it always serves the latest edit (`.../raw/buttons.json`, not
  `.../raw/<hash>/buttons.json`).
- Paste it into `index.html` as `GIST_RAW_URL`.
- To add/rename/remove a button later: either edit the Gist directly (Edit,
  change the JSON, Update gist), or use the `gist_editor` admin tool (see "Live
  setup" above) for a form instead of hand-editing JSON. Both take effect
  immediately, no redeploy.

### 3. Pipedream workflow
- New workflow at pipedream.com, trigger = **HTTP / Webhook**. Copy the endpoint URL.
- Paste it into `index.html` as `PIPEDREAM_WEBHOOK_URL`.
- Add a **Node.js** step with this code (set `PUSHOVER_TOKEN` / `PUSHOVER_USER` as
  Pipedream environment variables first, under the workflow's Settings — don't
  hardcode them in the step):

```javascript
export default defineComponent({
  async run({ steps, $ }) {
    const { title, message } = steps.trigger.event.body;
    await fetch("https://api.pushover.net/1/messages.json", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        token: process.env.PUSHOVER_TOKEN,
        user: process.env.PUSHOVER_USER,
        title,
        message,
      }),
    });
  },
});
```

### 4. Test end to end
- Open the GitHub Pages URL, tap a button, confirm the Pushover notification arrives.
- Check Pipedream's workflow inspector (Event History) if it doesn't — shows the
  incoming request and any errors from the Pushover call.

### 5. Print the QR code
- Once the Pages URL is live and stable, generate a QR code for it once (any free
  QR generator, e.g. qr-code-generator.com) and save it here as `door-qr.png`.
- No need for a QR-generator tool in this repo — the target URL never changes,
  only the buttons inside the page do.

## Why this design (in case it needs revisiting)

- **Buttons instead of one fixed message** — lets the visitor convey a specific
  reason without typing anything.
- **Config in a Gist, not the HTML** — button changes shouldn't require touching
  code or redeploying.
- **Pipedream in the middle, not a direct browser->Pushover call** — a direct call
  would put the Pushover API token in the public page's source, letting anyone who
  views source spam notifications indefinitely. Pipedream keeps the token
  server-side.
- **Pushover over ntfy** — already paid for, and its app supports priority alerts,
  custom sounds, quiet hours, which ntfy's free tier doesn't match.
- **No two-way chat** — considered and deliberately dropped. A reply path needs a
  session id, a place for the visitor's page to poll, and somewhere for the owner
  to type a reply — a genuinely separate small backend project, not an extension
  of this one. Revisit only if the button set turns out to not be expressive enough
  in practice.
