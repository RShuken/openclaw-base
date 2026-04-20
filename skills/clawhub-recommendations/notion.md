---
name: "notion (steipete)"
clawhub_slug: "@steipete/notion"
clawhub_url: "https://clawhub.ai/@steipete/notion"
category: "integrations"
author: "@steipete"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "active"
license: "Unknown <VERIFY>"
recommends_over: "none (our scratch-built notion skill was engagement-specific and has been deleted)"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@steipete/notion"
---

# notion by @steipete

## What it is

Community-standard Notion integration skill for OpenClaw agents. Gives an agent tools to search, fetch, create, and update Notion pages and databases through Notion's REST API, with managed auth.

## Why we recommend it

- @steipete's skills are the most-installed reference implementations in their categories; his gog ships 158k downloads.
- Strong README structural patterns (§H of baseline) — Quick Reference table, platform sections, security disclaimer.
- Our prior notion skill was engagement-specific and has been removed from `skills/`.

## When to pick this over our own deploy-*

- Any new client needing Notion read/write. We have no in-house Notion skill anymore.
- Template-backed or database-heavy Notion usage (page-per-record patterns).

## When NOT to pick this

- Client uses Notion through MCP (e.g., Claude.ai's Notion MCP already available) and doesn't need a CLI-shaped skill.
- Strict data-residency requirement — Notion is US-cloud-only.

## Install

```bash
clawhub install @steipete/notion
# Verify on target; some surfaces use `openclaw skills install` in 2026.4.15+.
```

## Configure / wire into OpenClaw

Requires a Notion integration token (create at https://www.notion.so/my-integrations) and page-share with the integration. Per the typical @steipete pattern, auth lives in `~/.openclaw/.env` or the skill's own `auth-profiles.json` entry.

## Known issues / gotchas

- Notion API rate limits: 3 req/s per integration. Heavy sync jobs need backoff.
- Integrations don't inherit access to new pages automatically — user must share each root page.

## Citations

- https://clawhub.ai/@steipete/notion
- `audit/_baseline.md` §B
