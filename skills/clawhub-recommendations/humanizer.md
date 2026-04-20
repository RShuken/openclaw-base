---
name: "humanizer"
clawhub_slug: "@biostartechnology/humanizer"
clawhub_url: "https://clawhub.ai/@biostartechnology/humanizer"
category: "content-creation"
author: "@biostartechnology"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "active"
license: "Unknown <VERIFY>"
recommends_over: "deploy-humanizer.md"
cost_tier: "free-install"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@biostartechnology/humanizer"
---

# humanizer by @biostartechnology

## What it is

Community humanizer skill — rewrites AI-generated prose to pass AI-detection classifiers and sound more natural. Baseline §B flags this as the popular pick; alternatives `ai-humanizer` and `humanizer-enhanced` exist but are less-adopted.

## Why we recommend it

- Popular, well-reviewed per baseline §B ("Strong match — popular").
- Covers the common humanizer prompts without having to reinvent detection-passing heuristics.
- Plays nicely with the content pipeline skills we already ship.

## When to pick this over our own deploy-*

- Prefer this as the implementation backing our `deploy-humanizer.md` recipe.
- Any client whose content pipeline requires AI-detection-resistant copy.

## When NOT to pick this

- Clients who explicitly want their AI-generated content labeled as AI-generated (disclosure-first workflows).
- Use cases that need reproducible, auditable rewrites — humanizers introduce stylistic variance by design.

## Install

```bash
clawhub install @biostartechnology/humanizer
```

## Configure / wire into OpenClaw

Typically invoked by the content pipeline agent as a sub-skill. Wire it so `deploy-content-pipeline.md` or `deploy-humanizer.md` calls into it rather than duplicating the prompt templates.

## Known issues / gotchas

- Detection-classifier arms race — humanizers that worked 3 months ago may drift. Re-verify against current GPTZero / Originality.ai releases when adopting.
- Some classifiers flag over-humanized text as "too casual" — calibrate style floor per client.

## Citations

- https://clawhub.ai/@biostartechnology/humanizer
- `skills/deploy-humanizer.md`
- `audit/_baseline.md` §B
