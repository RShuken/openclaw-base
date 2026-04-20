---
name: "morning-daily-briefing"
clawhub_slug: "@-/morning-daily-briefing"
clawhub_url: "https://clawhub.ai/@-/morning-daily-briefing"
category: "productivity"
author: "<VERIFY — baseline §B lists bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-daily-briefing.md"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/morning-daily-briefing"
---

# morning-daily-briefing

## What it is

Daily briefing skill — pulls calendar, email, task, and news/market inputs and produces a single morning summary. Baseline §B labels it a "strong match" vs. our `deploy-daily-briefing.md`; alternatives `ai-daily-briefing` and `daily-news-briefing`.

## Why we recommend it

- Popular, battle-tested pattern; composable with whatever source skills a client already has installed.
- Our `deploy-daily-briefing.md` can become a thin recipe that chains this skill into the client's delivery channel (Telegram, email).

## When to pick this over our own deploy-*

- Any client who wants a morning briefing without bespoke source selection.
- Works especially well stacked with `@steipete/gog` (calendar + email) and `linear-api` / `asana-api` (task inputs).

## When NOT to pick this

- Client's briefing must include proprietary data sources (internal dashboards, non-public APIs) that the ClawHub skill doesn't model.
- Multi-persona / advisory-council briefings — see `deploy-advisory-council.md` instead.

## Install

```bash
clawhub install morning-daily-briefing
# Verify author handle on clawhub.ai.
```

## Configure / wire into OpenClaw

```bash
openclaw cron add --name morning-briefing --cron "0 7 * * *"
```

Route delivery through `openclaw channels send --channel telegram` (or email via gog). Per baseline §L, prefer local Ollama for summary generation to keep recurring cost near zero.

## Known issues / gotchas

- Multiple source integrations = surface area for prompt injection (see §K — "email PI extracting private keys"). Use read-only tokens and `skill-vetter.md` pre-install.
- Summary length drifts over time — pin a max-tokens budget.

## Citations

- https://clawhub.ai/@-/morning-daily-briefing
- `skills/deploy-daily-briefing.md`
- `audit/_baseline.md` §B
