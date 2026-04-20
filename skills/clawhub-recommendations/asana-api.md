---
name: "asana-api"
clawhub_slug: "@-/asana-api"
clawhub_url: "https://clawhub.ai/@-/asana-api"
category: "integrations"
author: "<VERIFY — baseline §B lists it as bare-slug with managed-OAuth tag>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-asana.md"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/asana-api"
---

# asana-api

## What it is

Managed-OAuth Asana integration for OpenClaw. Alternatives in baseline §B: `asana-pat` (personal-token) and `asana-agent-skill`. `asana-api` is tagged "managed-OAuth" which is the easier onboarding path for multi-user or non-technical client installs.

## Why we recommend it

- Managed OAuth avoids forcing clients to mint and rotate PATs.
- Our own `deploy-asana.md` has been WORKING but duplicates what this skill already handles.
- Per baseline §B: "evaluate before scratch-building."

## When to pick this over our own deploy-*

- Greenfield Asana installs — prefer this skill as the primary integration.
- Multi-user workspaces where per-user OAuth beats a shared PAT.

## When NOT to pick this

- Highly customized Asana workflows (complex rules, webhooks, custom fields) where our `deploy-asana.md` logic is still useful on top.
- Air-gapped / no-cloud-callback environments — managed OAuth needs a public callback URL.

## Install

```bash
clawhub install asana-api
# Author handle not captured in baseline — verify on clawhub.ai.
```

## Configure / wire into OpenClaw

OAuth flow on first run — the skill typically opens a browser or prints a device-code URL. Verify behavior on headless enrollments.

Pair with `openclaw cron add --name asana-sync --cron "*/15 * * * *"` for periodic sync (see `../deploy-asana.md`).

## Known issues / gotchas

- Managed-OAuth skills on ClawHub generally route callbacks through a third-party relay — vet the relay domain before trusting (see `skill-vetter.md`).
- Asana rate limits: 150 req/min per PAT / OAuth-token.

## Citations

- https://clawhub.ai/@-/asana-api
- `skills/deploy-asana.md`
- `audit/_baseline.md` §B
