# Architecture

Static site, no build step, no backend code in this repo.

- `index.html` — the whole app. Fetches button config from a GitHub Gist
  (`GIST_RAW_URL`) at page load, renders one button per entry, and on click POSTs
  `{title, message}` to a Pipedream webhook (`PIPEDREAM_WEBHOOK_URL`).
- Button config (`buttons.json`) lives in a Gist, not in this repo — that's the
  point: editing buttons is a Gist edit, not a code change or redeploy.
- Notification delivery (Pipedream workflow -> Pushover API) is configured in
  Pipedream's UI, not in this repo. See `NOTES.md` for the exact Node.js step and
  setup sequence.
- Hosting: GitHub Pages, deployed from this repo's default branch.

Full design rationale and setup steps: `NOTES.md`.
