---
name: "nano-banana-pro"
clawhub_slug: "@steipete/nano-banana-pro"
clawhub_url: "https://clawhub.ai/@steipete/nano-banana-pro"
category: "content-creation"
author: "@steipete"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "active"
license: "Unknown <VERIFY>"
recommends_over: "deploy-image-gen.md"
cost_tier: "api-key-required"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@steipete/nano-banana-pro"
---

# nano-banana-pro by @steipete

## What it is

Gemini 3 Pro Image (aka "Nano Banana Pro 2") image generation and editing skill for OpenClaw agents. Our `deploy-image-gen.md` already references a `nano-banana-pro-2` workspace skill as its primary implementation — this is the upstream ClawHub distribution.

## Why we recommend it

- Baseline §B flags it as the popular pick vs. `best-image-generation` / `cheapest-image-generation`.
- @steipete ships maintained, well-documented skills (same author as `gog`).
- Tracks Gemini 3 Pro Image model updates without the client having to rebuild their own wrapper.

## When to pick this over our own deploy-*

- Prefer installing this over scratch-coding the Gemini 3 Pro Image client.
- `deploy-image-gen.md` can remain as the deployment recipe that *uses* this skill.

## When NOT to pick this

- Client needs on-device / local image gen (use SDXL / FLUX locally instead — not in this catalog).
- Strictly free-tier workflows — Gemini 3 Pro Image is paid per call after free quota.

## Install

```bash
clawhub install @steipete/nano-banana-pro
```

Set `GEMINI_API_KEY` in `~/.openclaw/.env`.

## Configure / wire into OpenClaw

The skill exposes `generate_image.py` (or equivalent) to the agent. Our `deploy-image-gen.md` wraps this in a `${WORKSPACE}/skills/nano-banana-pro-2/` layout — keep the wrapping path consistent or update both.

## Known issues / gotchas

- Gemini API quota tiers shift — budget monitoring is on you.
- Edit operations (`-i <input>`) require the input image already exist on disk at the path passed.

## Citations

- https://clawhub.ai/@steipete/nano-banana-pro
- `skills/deploy-image-gen.md`
- `audit/_baseline.md` §B
