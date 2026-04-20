---
name: "ai-meeting-prep"
clawhub_slug: "@-/ai-meeting-prep"
clawhub_url: "https://clawhub.ai/@-/ai-meeting-prep"
category: "productivity"
author: "<VERIFY — baseline §B lists bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "none (our meeting-prep skill was engagement-specific and has been deleted)"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/ai-meeting-prep"
---

# ai-meeting-prep

## What it is

Meeting preparation skill — for each upcoming calendar event, pulls attendee context (recent emails, LinkedIn, prior meeting notes, CRM entry) and drafts a pre-meeting brief. Baseline §B lists `meeting-prep`, `ai-meeting-prep`, `meeting-prep-agent` as strong matches.

## Why we recommend it

- We don't currently ship a generic meeting-prep skill — our prior one was engagement-specific and was removed.
- Community options are well-trodden; one-shot install beats bespoke build.

## When to pick this over our own deploy-*

- Any client with a calendar-heavy role (execs, founders, sales) who wants context before every meeting.
- Pairs with `@steipete/gog` (calendar + Gmail) and a CRM skill (`personal-crm` or client's existing HubSpot/Salesforce).

## When NOT to pick this

- Highly regulated industries where attendee data enrichment must flow through approved vendors only.
- Client's meetings are internal-only and don't benefit from enrichment.

## Install

```bash
clawhub install ai-meeting-prep
# Verify author handle and pick between `meeting-prep`, `ai-meeting-prep`, `meeting-prep-agent`.
```

## Configure / wire into OpenClaw

Needs calendar (gog) + email (gog) + optional CRM. Schedule:

```bash
openclaw cron add --name meeting-prep --cron "0 6 * * *"   # morning sweep
openclaw cron add --name meeting-prep-intraday --cron "*/30 * * * *"  # refresh
```

## Known issues / gotchas

- Enrichment quality correlates with source coverage — install CRM before enabling to avoid empty briefs.
- Running on every event with an LLM call = cost spike; cap to meetings with external attendees.

## Citations

- https://clawhub.ai/@-/ai-meeting-prep
- `audit/_baseline.md` §B
