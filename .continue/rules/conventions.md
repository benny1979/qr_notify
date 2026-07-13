# Conventions

- Single-file static site by design — don't split `index.html` into multiple
  files or add a build step/framework for a page this small.
- `GIST_RAW_URL` and `PIPEDREAM_WEBHOOK_URL` are the only two project-specific
  constants in `index.html`. Both are plain public URLs — fine to hardcode, don't
  hold secrets (the Pushover token lives in Pipedream's env vars, never here).
- Button contents (label/emoji/title/message) are never hardcoded in `index.html`
  — they come from the Gist. If asked to "add a button," edit the Gist, not this
  file.
- No two-way chat / reply path — deliberately out of scope. See "Why this design"
  in `NOTES.md` before reviving that idea.
