---
name: "earnings-calendar"
clawhub_slug: "@-/earnings-calendar"
clawhub_url: "https://clawhub.ai/@-/earnings-calendar"
category: "research"
author: "<VERIFY — baseline §B lists bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-earnings.md (stock-market flavor only)"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/earnings-calendar"
---

# earnings-calendar

## What it is

Public-company earnings calendar skill — surfaces upcoming earnings dates, pulls transcripts/summaries post-release, flags movers. Baseline §B lists three options (`earnings-reader`, `earnings-calendar`, `stock-earnings-review`) — all stock-market-focused.

## Why we recommend it

- When a client wants stock-earnings tracking, community skills cover it well.
- Our `deploy-earnings.md` targets *personal-revenue earnings tracking* (per baseline §B: "No equivalents for personal-revenue earnings tracker"), which is different — so this ClawHub skill covers the stock-market gap.

## When to pick this over our own deploy-*

- Client wants stock-market earnings surfacing (not personal revenue).
- Analyst / investor / founder watching a specific ticker list.

## When NOT to pick this

- Client's "earnings" actually means personal / business revenue — stick with `deploy-earnings.md` (our unique value-add per baseline §B implications).
- Real-time intraday trading signals — these skills are batch/daily, not real-time.

## Install

```bash
clawhub install earnings-calendar
# Evaluate `earnings-reader` and `stock-earnings-review` too — baseline lists all three.
```

## Configure / wire into OpenClaw

Usually needs an API key for market-data source (varies per skill). Cron:

```bash
openclaw cron add --name earnings-calendar --cron "0 5 * * MON"
```

For release-date one-shots, use `openclaw cron add --at "YYYY-MM-DD HH:MM" --delete-after-run` per `../deploy-earnings.md`.

## Known issues / gotchas

- Market-data API tiers change — free tiers often have 15-min delay and low ticker coverage.
- Skill name overlap — verify which of the three variants you actually installed.

## Citations

- https://clawhub.ai/@-/earnings-calendar
- `skills/deploy-earnings.md`
- `audit/_baseline.md` §B
