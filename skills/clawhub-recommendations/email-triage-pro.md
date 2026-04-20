---
name: "email-triage-pro"
clawhub_slug: "@-/email-triage-pro"
clawhub_url: "https://clawhub.ai/@-/email-triage-pro"
category: "productivity"
author: "<VERIFY — baseline §B lists bare slug>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-urgent-email.md"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/email-triage-pro"
---

# email-triage-pro

## What it is

Email triage skill — classifies inbox into urgent / reply / defer / archive buckets and surfaces the urgent ones to the agent / user. Baseline §B lists three options: `email-triage`, `expanso-email-triage`, `email-triage-pro`. The `-pro` variant is the most featureful.

## Why we recommend it

- Direct match to our `deploy-urgent-email.md` purpose.
- "Pro" variant generally includes better classification prompts and categorization than the plain skill.

## When to pick this over our own deploy-*

- Any greenfield client who wants urgent-email surfacing without us maintaining the classification prompts.
- Pairs with `@steipete/gog` for the Gmail read layer.

## When NOT to pick this

- Client needs highly custom triage rules (per-sender, per-label routing) — our `deploy-urgent-email.md` is easier to fork for that.
- Strict privacy / on-prem — email-triage-pro is cloud-only.

## Install

```bash
clawhub install email-triage-pro
# Verify author handle on clawhub.ai. Baseline lists bare slug.
```

## Configure / wire into OpenClaw

Needs Gmail access (install `@steipete/gog` first). Cron recommended:

```bash
openclaw cron add --name email-triage --cron "*/5 * * * *"
```

Per `../deploy-urgent-email.md`, route classification calls through a cheap model (`URGENT_EMAIL_MODEL=openai-codex/gpt-5.4-nano` or similar).

## Known issues / gotchas

- Classification quality degrades with domain-specific jargon — test on the client's actual inbox sample before trusting.
- Cost can spiral if routed through a frontier model — always force the cheap model via env override.

## Citations

- https://clawhub.ai/@-/email-triage-pro
- `skills/deploy-urgent-email.md`
- `audit/_baseline.md` §B
