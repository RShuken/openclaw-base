---
name: "linear-api"
clawhub_slug: "@-/linear-api"
clawhub_url: "https://clawhub.ai/@-/linear-api"
category: "integrations"
author: "<VERIFY — author handle not captured in baseline §B, listed under bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "none (no deploy-linear.md currently in this repo)"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/linear-api"
---

# linear-api

## What it is

Linear integration skill — exposes Linear issue CRUD, cycle/project queries, and comment operations to an OpenClaw agent via Linear's GraphQL API. Baseline §B lists three Linear options (`linear-api`, `linear-skill`, `linear-autopilot`); `linear-api` is the flat, API-first one.

## Why we recommend it

- We have no in-house Linear deploy-* skill today.
- Clients frequently need Linear read (for status scans, daily briefings) even if they don't want full autopilot.
- API-only shape is easier to audit with `skill-vetter` than an autopilot variant.

## When to pick this over our own deploy-*

- Any client using Linear. We ship nothing native for Linear.
- Pairs well with `deploy-daily-briefing.md` for cycle-aware briefings.

## When NOT to pick this

- Client's issue tracker is Asana / Shortcut / Jira — use the matching skill.
- If the client wants agents *creating* issues autonomously, vet `linear-autopilot` first (beyond read-only).

## Install

```bash
clawhub install linear-api
# Verify the actual author handle — baseline lists it as bare-slug `@-/linear-api`.
```

## Configure / wire into OpenClaw

Set `LINEAR_API_KEY` in `~/.openclaw/.env` (personal API key from Linear settings → API). Scope to read-only first per baseline §L ("Read-only first").

## Known issues / gotchas

- Linear's GraphQL schema evolves — skills that shipped pre-2026 may use deprecated fields. Verify last-updated date on install.
- **Author handle unverified** in our baseline research — confirm on clawhub.ai before citing externally.

## Citations

- https://clawhub.ai/@-/linear-api
- `audit/_baseline.md` §B
