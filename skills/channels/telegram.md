---
name: "Telegram"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "yes"
openclaw_cli: "openclaw channels send --channel telegram"
raw_api_fallback: "https://api.telegram.org/bot<TOKEN>/"
auth_type: "bot-token"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Bot token from @BotFather"
  - "Chat ID (user, group, or forum/supergroup)"
docs_urls:
  - "https://core.telegram.org/bots/api"
  - "https://docs.openclaw.ai/channels/telegram"
---

# Telegram

## What it is

Primary messaging backbone for most OpenClaw deployments. Flat DM, group chats, and Forum/Topics supergroups are all supported. OpenClaw reads `channels.telegram.*` from `openclaw.json` and exposes `openclaw channels send --channel telegram --to <chat_id> --text "..."`. Forum/Topics setup (creating threads, setting `can_manage_topics`) is not exposed through the CLI yet and goes through the raw Bot API.

## Native OpenClaw support

**yes** — send/receive via `openclaw channels send --channel telegram`. Webhook inbound is wired through `channels.telegram.webhookUrl` + `webhookSecret` (see `diagnose-cron.md`). Forum/Topics creation still needs `curl` to `api.telegram.org`.

## Authentication

1. Open Telegram, message `@BotFather`, run `/newbot`, choose name + handle.
2. Copy the bot token (format: `7123456789:AAF...`).
3. Save to `~/.openclaw/openclaw.json` under `channels.telegram.botToken`, or to `.env` as `TELEGRAM_BOT_TOKEN`.
4. Discover chat IDs via `curl https://api.telegram.org/bot${TOKEN}/getUpdates` after someone messages the bot.

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -i telegram
openclaw channels list | grep -i telegram

# Verify token
curl -sS "https://api.telegram.org/bot${TOKEN}/getMe"

# Send a test
openclaw channels send --channel telegram --to "${CHAT_ID}" --text "hello"
```

## Wire into skills

- `deploy-messaging-setup.md` — primary configuration skill, sets up topic routing
- `deploy-telegram-forum.md` — Forum/Topics mode, thread creation, bot permissions
- `deploy-health-monitoring.md`, `deploy-db-backups.md`, `deploy-food-journal.md` — consumers that send through `openclaw channels send --channel telegram`
- ClawHub alternatives: `agent-telegram`, `rho-telegram-alerts`

## Rate limits

- 30 messages/sec across different chats globally
- 1 message/sec per individual chat (burst tolerated, sustained throttled)
- 20 messages/minute per group chat
- `429 Too Many Requests` returns `retry_after` seconds — honor it

## Known issues / gotchas

- `openclaw telegram` subcommand does NOT exist as of 2026.4.15 — use `openclaw channels` (per baseline §4C.3)
- Forum/Topics configuration isn't exposed through the CLI — use raw Bot API for `createForumTopic`, `editForumTopic`
- `messaging.*` config namespace doesn't exist — everything lives under `channels.telegram.*`

## When to pick this channel

Default pick for any OpenClaw agent with a human principal. Free, reliable, supports threaded organization via Forums, works everywhere, good bot API. Prefer over Slack/Discord unless the client already lives in one of those.

## Citations

- https://core.telegram.org/bots/api
- `skills/deploy-telegram-forum.md`
- `audit/_baseline.md` §4C.3, §4C.7
