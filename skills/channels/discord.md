---
name: "Discord"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "yes"
openclaw_cli: "openclaw channels send --channel discord"
raw_api_fallback: "https://discord.com/api/v10/"
auth_type: "bot-token"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Discord application + bot at discord.com/developers/applications"
  - "Bot token + invited to a guild with required intents"
  - "Server (guild) ID + channel IDs"
docs_urls:
  - "https://discord.com/developers/docs/intro"
  - "https://discord.com/developers/docs/topics/gateway"
---

# Discord

## What it is

Server + channel model — good for community-facing agents or clients who organize around Discord servers. OpenClaw `channels send --channel discord` targets a `channel_id`, and inbound events flow via the Gateway (WebSocket) or Interactions webhook. Threads-within-channels are supported and map naturally to per-topic routing similar to Telegram Forums.

## Native OpenClaw support

**yes** — per baseline §4C.7. Send and receive both covered. Slash commands registration and voice-channel features are provider-specific and not CLI-exposed.

## Authentication

1. Visit `discord.com/developers/applications` → "New Application"
2. Add a Bot, copy the bot token (once — can only be regenerated, not re-viewed)
3. Enable intents as needed: `GUILDS`, `GUILD_MESSAGES`, `MESSAGE_CONTENT` (privileged)
4. OAuth2 → URL generator → select `bot` scope + permissions → invite URL
5. Save token to `channels.discord.botToken` in openclaw.json

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -i discord
openclaw channels list | grep -i discord

# Send a test (channel_id, not name)
openclaw channels send --channel discord --to "${CHANNEL_ID}" --text "hello"
```

## Wire into skills

- `deploy-messaging-setup.md` — Discord is listed as a supported primary/secondary platform
- No Discord-specific skill currently — use generic messaging routes through `openclaw channels send`

## Rate limits

- Global: 50 requests/sec per bot
- Per-route: varies (e.g., 5 messages per 5 seconds in a channel)
- Response headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset-After`
- 429 response includes `retry_after` (in seconds, as float)
- Gateway: one identify per 5 seconds

## Known issues / gotchas

- `MESSAGE_CONTENT` is a privileged intent since 2022 — must be approved for bots in 100+ guilds, otherwise messages come through with empty content
- Bot token rotation: once regenerated, old token invalidates immediately with no grace period
- Unverified — no known Discord-specific bugs in OpenClaw baseline §M (as of 2026-04-20); if Discord breaks the signal is usually silent (send returns ok but message never appears)

## When to pick this channel

Client already runs a community/team on Discord; voice-channel features are wanted (though OpenClaw's Discord voice support is unverified — check CLI before promising). Good for multi-user agents where Telegram's chat model feels constrained.

## Citations

- https://discord.com/developers/docs/topics/rate-limits
- https://discord.com/developers/docs/topics/gateway
- `audit/_baseline.md` §4C.7
