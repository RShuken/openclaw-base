---
name: "beehiiv"
clawhub_slug: "@-/beehiiv"
clawhub_url: "https://clawhub.ai/@-/beehiiv"
category: "integrations"
author: "<VERIFY — baseline §B lists as bare-slug with managed-OAuth tag>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-newsletter-crm.md (Beehiiv sync portion)"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/beehiiv"
---

# beehiiv

## What it is

Managed-OAuth Beehiiv integration skill for OpenClaw — subscriber sync, post metrics, publication management. Baseline §B lists alternatives: `beehiiv-integration`, `newsletter-digest`.

## Why we recommend it

- Managed OAuth means clients don't paste Beehiiv API keys into plaintext config.
- Our `deploy-newsletter-crm.md` already flags this as a preferred alternative under the Beehiiv sync half of the skill.

## When to pick this over our own deploy-*

- Any client who only needs Beehiiv sync (no HubSpot side of newsletter-crm).
- Newsletter-first agents that want subscriber-count / engagement metrics surfaced in daily briefings.

## When NOT to pick this

- Client needs bidirectional newsletter+CRM sync (HubSpot, Salesforce) — keep using `deploy-newsletter-crm.md` which combines both.
- Client's ESP is Substack / ConvertKit / Mailchimp — wrong skill, look for the matching integration.

## Install

```bash
clawhub install beehiiv
# Verify author handle on clawhub.ai.
```

## Configure / wire into OpenClaw

OAuth flow on install. Once connected:

```bash
openclaw cron add --name beehiiv-sync --cron "0 */4 * * *"
```

(Stagger with any other sync jobs per `../deploy-newsletter-crm.md`.)

## Known issues / gotchas

- Beehiiv API v2 migration may have invalidated some pre-2025 skill versions — verify version_last_verified on install.
- Rate limits are modest (100 req/min) — batch subscriber queries.

## Citations

- https://clawhub.ai/@-/beehiiv
- `skills/deploy-newsletter-crm.md`
- `audit/_baseline.md` §B
