# QR Door Notify

Scan a QR code on the door → pick a reason (Visitor / Delivery / etc.) on a simple
web page → get a Pushover notification. No app needed for the visitor, no
account, nothing typed.

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
- Create a Gist at gist.github.com containing one file, `buttons.json`:

```json
[
  { "label": "Visitor",  "emoji": "🔔", "title": "Someone's at the door", "message": "A visitor is at the door." },
  { "label": "Delivery", "emoji": "📦", "title": "Delivery",              "message": "A delivery has arrived." }
]
```

- Click the file's "Raw" button, copy that URL.
- Paste it into `index.html` as `GIST_RAW_URL`.
- To add/rename/remove a button later: open the Gist, Edit, change the JSON, Update
  public gist. Takes effect immediately, no redeploy.

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
