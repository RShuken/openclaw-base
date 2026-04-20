---
name: "Slack"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "yes"
openclaw_cli: "openclaw channels send --channel slack"
raw_api_fallback: "https://slack.com/api/chat.postMessage"
auth_type: "oauth"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "rate-limited"
privacy_tier: "cloud"
requires:
  - "Slack workspace admin (to install app)"
  - "Bot token (xoxb-*) and optionally app token (xapp-*) for Socket Mode"
  - "Scopes: chat:write, app_mentions:read, channels:history (as needed)"
docs_urls:
  - "https://api.slack.com/docs"
  - "https://api.slack.com/apis/socket-mode"
---

# Slack

## What it is

Secondary messaging channel for workspaces where the human already lives in Slack. OpenClaw's pattern is **mention-only mode** — the bot doesn't auto-respond in channels, only when explicitly `@mentioned`. Inbound events arrive via Events API webhook or Socket Mode (preferred for NAT'd machines). Used as a trigger surface for pipelines like `deploy-video-pipeline.md`.

## Native OpenClaw support

**yes** — `openclaw channels send --channel slack --to <channel_or_user_id> --text "..."` works. Socket Mode inbound is part of the `channels` subsystem per 2026.4.15 baseline. Mention-only policy is implemented in the agent's workspace rules (`TOOLS.md`), not enforced by the CLI.

## Authentication

1. Create a Slack app at `api.slack.com/apps` → "From scratch"
2. Add OAuth scopes: `chat:write`, `app_mentions:read`, plus per-feature scopes
3. Install to workspace → copy bot token `xoxb-...`
4. For Socket Mode: enable it, generate app-level token `xapp-...` with `connections:write`
5. Save tokens to `channels.slack.botToken` + `channels.slack.appToken` in openclaw.json

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -i slack
openclaw channels list | grep -i slack

# Send a test
openclaw channels send --channel slack --to "${CHANNEL_ID}" --text "hello from openclaw"
```

## Wire into skills

- `deploy-messaging-setup.md` — sets mention-only policy and routes
- `deploy-video-pipeline.md` — Slack mention is the trigger surface ("@assistant potential video idea")
- Generally paired with Telegram as secondary; Slack alone is rare

## Rate limits

- Tier 1: ~1 call/min — admin/heavy endpoints
- Tier 2: ~20 calls/min — conversations.*, users.*
- Tier 3: ~50 calls/min — chat.postMessage (most common)
- Tier 4: ~100 calls/min — auth.test
- Response headers `Retry-After` on 429 — honor exactly. New message posting tier applies as of 2025 API tier changes.

## Known issues / gotchas

- Community pattern: **mention-only** — without this discipline agents chatter constantly in channels and annoy teammates (per baseline §4C auth-profiles/messaging)
- Streaming + Slack socket mode had regressions — verify with `openclaw channels list` after any OpenClaw upgrade
- Tier 3 posting limit is per-workspace, not per-channel — multi-channel posting amortizes against the same budget

## When to pick this channel

When the client's team already works in Slack and adding another app (Telegram/Discord) would fragment attention. Also the default for work-adjacent agents where the human is on Slack 9-to-5 anyway.

## Citations

- https://api.slack.com/docs/rate-limits
- https://api.slack.com/apis/socket-mode
- `skills/deploy-video-pipeline.md`
