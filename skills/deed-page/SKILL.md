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

Keep one token per agent in private durable storage, for example `~/.config/deed-page/token` with `chmod 600` on the file and `chmod 700` on the directory, not only in a session variable. Every publish command below loads it from `DEEDPAGE_TOKEN` or that file and sends `Authorization: Bearer` only when a token exists, so publishes to new slugs, in this session and later ones, stay under one owner:

```sh
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set --
if [ -n "$TOKEN" ]; then set -- -H "Authorization: Bearer $TOKEN"; fi
```

Authenticated responses carry no `token`, so never overwrite a saved token with an empty value. If the host has no private persistent storage, tell the user and offer to keep the token in a credential store they manage.

### 2. Pick a slug

Lowercase letters, digits, and hyphens, 1 to 63 characters, no leading or trailing hyphen. Add a random suffix to avoid collisions:

```bash
SLUG="report-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' \n')"
```

### 3. Publish

Single HTML file (stored as `index.html`):

```sh
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set --
if [ -n "$TOKEN" ]; then set -- -H "Authorization: Bearer $TOKEN"; fi
RESP=$(curl -sS -T report.html "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: <your agent name>" "$@")
```

Folder (must contain `index.html` at its root):

```sh
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set --
if [ -n "$TOKEN" ]; then set -- -H "Authorization: Bearer $TOKEN"; fi
tar -C ./site -czf /tmp/site.tar.gz .
RESP=$(curl -sS -T /tmp/site.tar.gz "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: <your agent name>" "$@")
```

### 4. Answer the user and save the token

A first publish without a token also returns `token` once. Save it immediately, then show the response with the token masked. The file is written only when the response has a non-empty token, so a saved token is never replaced with an empty one. Never put the token in a page, commit, or log:

```sh
NEW=$(printf '%s' "$RESP" | sed -n 's/.*"token": *"\(deed_tok_[^"]*\)".*/\1/p')
if [ -n "$NEW" ]; then
  (umask 077 && mkdir -p ~/.config/deed-page && chmod 700 ~/.config/deed-page && printf '%s' "$NEW" > ~/.config/deed-page/token && chmod 600 ~/.config/deed-page/token)
  TOKEN=$NEW
fi
printf '%s\n' "$RESP" | sed 's/"token": *"[^"]*"/"token":"<saved>"/'
```

Reply with the `url` field.

### 5. Update or delete

Re-run the publish command to ship a new version; it sends the saved token. Delete only when the user asks:

```bash
curl -sS -X DELETE "https://deed.page/v1/sites/$SLUG" -H "Authorization: Bearer $TOKEN"
```

`TOKEN` is loaded as in step 1.

## Examples

### Example 1: Share a generated report

```sh
SLUG="q3-report-$(head -c4 /dev/urandom | od -An -tx1 | tr -d ' \n')"
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set --
if [ -n "$TOKEN" ]; then set -- -H "Authorization: Bearer $TOKEN"; fi
RESP=$(curl -sS -T q3-report.html "https://deed.page/v1/sites?slug=$SLUG" -H "X-Client: Claude Code" "$@")
# {"ok":true,"url":"https://q3-report-1a2b3c4d.deed.page/","slug":"q3-report-1a2b3c4d","version":"v1","token":"deed_tok_..."}
# Then save the token as in step 4.
```

### Example 2: Inline page with no file on disk

```sh
TOKEN=${DEEDPAGE_TOKEN:-$(cat ~/.config/deed-page/token 2>/dev/null)}
set --
if [ -n "$TOKEN" ]; then set -- -H "Authorization: Bearer $TOKEN"; fi
RESP=$(curl -sS -X POST "https://deed.page/v1/sites" -H 'content-type: application/json' -H "X-Client: Codex" "$@" \
  -d "{\"slug\":\"$SLUG\",\"files\":{\"index.html\":\"<!doctype html><h1>Hello</h1>\"}}")
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
