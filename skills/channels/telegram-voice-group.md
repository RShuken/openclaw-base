---
name: "Telegram Voice (Group Calls)"
category: "channel"
subcategory: "voice"
openclaw_native_support: "partial"
openclaw_cli: "openclaw channels send --channel telegram  # voice via ClawHub skill"
raw_api_fallback: "https://core.telegram.org/api (MTProto) — voice chat admin via GroupCall API"
auth_type: "bot-token"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "free"
privacy_tier: "cloud"
requires:
  - "Telegram bot token (as for regular Telegram)"
  - "Supergroup with voice chat enabled"
  - "ClawHub skill `telegram-voice-group` installed"
docs_urls:
  - "https://clawhub.ai/skills/telegram-voice-group"
  - "https://core.telegram.org/api/voice-chats"
---

# Telegram Voice (Group Calls)

## What it is

Voice-chat (Group Call / Video Chat) participation in a Telegram supergroup. Not a standalone protocol — it's an extension to the Telegram channel handled via the ClawHub `telegram-voice-group` skill. The agent can join a voice chat, transcribe, and respond in real-time. **This is a ClawHub skill layered on top of the Telegram channel**, not a separate entry in `openclaw channels list`.

## Native OpenClaw support

**partial** — the Telegram channel itself is native (`openclaw channels send --channel telegram`), but voice-chat features come from the ClawHub skill `telegram-voice-group` (per baseline §B, referenced in `deploy-messaging-setup.md`). Treat this as "Telegram + extension" rather than a separate channel.

## Authentication

Same as the Telegram channel:
1. Bot token from @BotFather.
2. Additionally, the bot needs `can_manage_video_chats` admin permission in the supergroup (set via `promoteChatMember` Bot API call).
3. Voice chats require MTProto-level calls for actual audio streaming — the ClawHub skill handles this.

## Install / configure

```bash
# Base Telegram must be configured first — see telegram.md
openclaw channels list | grep -i telegram

# Install the voice skill from ClawHub
openclaw skills install telegram-voice-group
# or per current CLI surface (verify with --help):
openclaw skills add telegram-voice-group
```

## Wire into skills

- `deploy-messaging-setup.md` — lists `telegram-voice-group` as a ClawHub alternative to evaluate
- Pairs naturally with transcription skills (`deploy-fathom-pipeline.md`, `deploy-video-analysis.md`) for meeting capture

## Rate limits

- Inherits Telegram's base rate limits (30 msgs/sec global, 1/sec per chat)
- Voice chat join/leave is not published but is light — don't flap
- Audio stream quality handled by Telegram infra; no bot-side rate limit on audio

## Known issues / gotchas

- Requires the supergroup to have voice chat enabled (client-side Telegram UI setting).
- Bot must be explicitly promoted with `can_manage_video_chats` — regular bot admin is not enough.
- ClawHub skill install path uncertain on 2026.4.15 — verify `openclaw skills --help` before promising
- Audio processing = compute. Plan for GPU or cloud STT if volume is high.
- Unverified: whether the ClawHub skill supports multi-participant transcription or single-speaker only.

## When to pick this channel

Client runs voice meetings in a Telegram supergroup and wants the agent to attend, transcribe, and respond. Or building a voice-first agent where Telegram is the delivery surface. Not a default — add when voice is explicitly required.

## Citations

- https://core.telegram.org/api/voice-chats
- `skills/deploy-messaging-setup.md` (ClawHub alternatives list)
- `audit/_baseline.md` §B (ClawHub skill catalog)
