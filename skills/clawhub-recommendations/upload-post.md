---
name: "upload-post"
clawhub_slug: "@-/upload-post"
clawhub_url: "https://clawhub.ai/@-/upload-post"
category: "content-creation"
author: "<VERIFY — baseline §B lists as bare slug, 'popular cross-platform'>"
downloads_last_checked: "unverified"
version_last_verified: "unverified"
last_checked: "2026-04-20"
maintenance_status: "unverified"
license: "Unknown <VERIFY>"
recommends_over: "deploy-tiktok-posting.md, deploy-youtube-posting.md"
cost_tier: "paid-service"
privacy_tier: "cloud"
docs_urls:
  - "https://clawhub.ai/@-/upload-post"
---

# upload-post

## What it is

Cross-platform social media posting skill — pushes a single video/post to TikTok, YouTube, Instagram, LinkedIn, X, etc. through a unified API. Baseline §B highlights it as the popular cross-platform pick alongside single-platform alts (`tiktok-uploader`, `publora-tiktok`).

## Why we recommend it

- One skill replaces our separate `deploy-tiktok-posting.md` and `deploy-youtube-posting.md` for clients posting the same content to multiple platforms.
- Avoids maintaining per-platform auth flows when the workflow is fundamentally "same asset, N destinations."

## When to pick this over our own deploy-*

- Multi-platform creators (most of them) — cheaper to maintain one skill + one service account than N.
- Pairs with `deploy-content-pipeline.md` as the final publish step.

## When NOT to pick this

- Platform-specific edit requirements (TikTok-only effects, YouTube chapters with timestamps, X thread splitting) — the unified layer flattens these.
- Clients explicitly want TikTok compliance/warmup (see `deploy-tiktok-compliance.md`, `deploy-tiktok-warmup.md`) which are account-hygiene flows, not publishing.

## Install

```bash
clawhub install upload-post
# Verify author handle — baseline lists bare slug.
```

## Configure / wire into OpenClaw

upload-post typically uses a third-party relay service (paid tier) to handle N platform auths in one SDK. Requires an account on that service — verify pricing and data-sharing terms before rolling out to a client.

## Known issues / gotchas

- Third-party relay = extra trust surface. Run through `skill-vetter.md` before installing.
- Per-platform quirks (TikTok watermark, YouTube category ID, LinkedIn char limits) may still leak through — test each destination.
- Pricing tiers change; verify current costs before committing a client.

## Citations

- https://clawhub.ai/@-/upload-post
- `skills/deploy-tiktok-posting.md`
- `skills/deploy-youtube-posting.md`
- `audit/_baseline.md` §B
