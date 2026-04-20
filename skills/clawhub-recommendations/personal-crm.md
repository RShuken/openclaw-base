---
name: "personal-crm"
clawhub_slug: "@-/personal-crm"
clawhub_url: "https://clawhub.ai/@-/personal-crm"
category: "crm"
author: "<VERIFY — baseline §B lists as bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-personal-crm.md (evaluate first)"
cost_tier: "free-install"
privacy_tier: "hybrid"
docs_urls:
  - "https://clawhub.ai/@-/personal-crm"
---

# personal-crm

## What it is

Community personal CRM skill — tracks people, relationships, meeting history, and follow-up reminders for an individual user. Baseline §B labels it a "strong match" for our `deploy-personal-crm.md`; alternatives include `heleni-personal-crm` and plain `crm`.

## Why we recommend it

- Lightweight alternative to our 20-table CRM schema where the client only needs relationship tracking.
- Maintained by the community — battle-tested on common "stay in touch" patterns.

## When to pick this over our own deploy-*

- Client wants quick relationship-tracking without the full ingestion-pipeline + Telegram NL-query interface our `deploy-personal-crm.md` provides.
- Solo operators, creators, founders.

## When NOT to pick this

- Client needs deep integration (Fathom transcripts → CRM linking, Telegram NL queries, multi-source dedup) — stick with our `deploy-personal-crm.md`.
- Client has an existing CRM (HubSpot, Salesforce) that's the source of truth.

## Install

```bash
clawhub install personal-crm
# Verify author handle on clawhub.ai; try alternatives `heleni-personal-crm`, `crm` if this one is dormant.
```

## Configure / wire into OpenClaw

Typically file-based (markdown entries per person in a `people/` folder) or SQLite-backed. Read the README — shape varies.

Wire to Telegram for quick-add via `openclaw channels send` commands or by installing the matching `agent-telegram` skill from baseline §B.

## Known issues / gotchas

- Multiple "personal-crm" skills on ClawHub — verify you have the right author before committing a client.
- If the skill is file-based, back up the `people/` folder via `deploy-db-backups.md`.

## Citations

- https://clawhub.ai/@-/personal-crm
- `skills/deploy-personal-crm.md`
- `audit/_baseline.md` §B
