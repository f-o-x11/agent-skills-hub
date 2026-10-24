---
name: deed-page
description: "Publish a generated HTML file or static folder to a permanent public https://<slug>.deed.page/ URL with one curl call, then update or delete it."
risk: critical
source: https://github.com/f-o-x11/deed-page-skill/tree/main/skills/deed-page
---

# deed.page publish

## Overview

[deed.page](https://deed.page) hosts static files for agents. One `curl` upload of an HTML file or a tarred folder returns a live `https://<slug>.deed.page/` URL. No account, API key, or email code is needed; the first publish returns an owner token that is the only key for later updates and deletes.

Risk: **critical**. Publishing makes files public on the internet, and update/delete change a live site. Confirm with the user before publishing anything that could contain private data.

## When to Use This Skill

- Use when the user asks to publish, host, share online, "put this online", or "give me a link" for something you generated: an HTML report, dashboard, slide deck, landing page, prototype, game, or docs folder.
- Use when you run unattended and need a public URL without asking a human for a login, code, or click.
- Do not use for secrets, private data, or anything needing server-side code or a database.

## Step-by-Step Guide

### 1. Load the owner token (if one exists)

Keep one token per agent in `~/.config/deed-page/token` (mode 600) so all sites stay under one owner:

```bash
export DEEDPAGE_TOKEN="$(cat ~/.config/deed-page/token 2>/dev/null)"
```

### 2. Pick a slug

Lowercase letters, digits, and hyphens, 1 to 63 characters, no leading or trailing hyphen. Add a random suffix to avoid collisions:

```bash
SLUG="report-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' \n')"
```

### 3. Publish

Single HTML file (stored as `index.html`):

```bash
curl -sS -T report.html "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: <your agent name>" ${DEEDPAGE_TOKEN:+-H "Authorization: Bearer $DEEDPAGE_TOKEN"}
```

Folder (must contain `index.html` at its root):

```bash
tar -C ./site -czf /tmp/site.tar.gz .
curl -sS -T /tmp/site.tar.gz "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: <your agent name>" ${DEEDPAGE_TOKEN:+-H "Authorization: Bearer $DEEDPAGE_TOKEN"}
```

### 4. Answer the user and save the token

Reply with the `url` field. A first publish without a token also returns `token` once. Save it immediately and never put it in a page, commit, or log:

```bash
mkdir -p ~/.config/deed-page && printf '%s' "$TOKEN" > ~/.config/deed-page/token && chmod 600 ~/.config/deed-page/token
```

### 5. Update or delete

Re-run the publish command with `DEEDPAGE_TOKEN` set to ship a new version. Delete only when the user asks:

```bash
curl -sS -X DELETE "https://deed.page/v1/sites/$SLUG" -H "Authorization: Bearer $DEEDPAGE_TOKEN"
```

## Examples

### Example 1: Share a generated report

```bash
SLUG="q3-report-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' \n')"
curl -sS -T q3-report.html "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: Claude Code"
# {"ok":true,"url":"https://q3-report-1a2b3c4d.deed.page/","slug":"q3-report-1a2b3c4d","version":"v1","token":"deed_tok_..."}
```

### Example 2: Inline page with no file on disk

```bash
curl -sS -X POST "https://deed.page/v1/sites" -H 'content-type: application/json' -H "X-Client: Codex" \
  -d "{\"slug\":\"$SLUG\",\"files\":{\"index.html\":\"<!doctype html><h1>Hello</h1>\"}}"
```

## Best Practices

- ✅ **Do:** reuse one saved token so every site stays under one owner.
- ✅ **Do:** add `-H "X-Ttl: 86400"` for throwaway previews that should expire.
- ❌ **Don't:** publish credentials, `.env` files, or personal data; Free sites are public.
- ❌ **Don't:** delete or overwrite a site the user did not ask you to change.

## Limitations

- Static files only: no server-side code, databases, or build step.
- Free plan: 25 sites, 25 MB per site, 60 publishes per hour, 4 MB per single request (larger uploads use the manifest flow in the docs).
- A lost token cannot be recovered unless a recovery email was attached; the site stays online.

## Troubleshooting

**Problem:** `409 slug_taken`
**Solution:** Use the `suggestion` field from the response or a new random suffix.

**Problem:** `422 content_rejected`
**Solution:** The upload contained executable or phishing markers. Remove that file; do not retry as-is.

**Problem:** `429` rate limited
**Solution:** Wait `retry_after` seconds before publishing again.

Full API, MCP server (`https://deed.page/mcp`), versions, custom domains, and passwords: https://deed.page/docs
