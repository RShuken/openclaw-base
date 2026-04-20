# Channels — index

This directory holds one file per messaging channel that OpenClaw supports. Each file has YAML frontmatter an audit agent can parse (native support status, CLI surface, auth type, rate limits, last-checked date).

**Parent skills:**
- [`../deploy-messaging-setup.md`](../deploy-messaging-setup.md) — the primary routing doc that configures the communication backbone
- [`../deploy-telegram-forum.md`](../deploy-telegram-forum.md) — Telegram-specific Forums/Topics setup

**Template for adding new channels:** [`_template.md`](./_template.md)

## Index

| File | Channel | OpenClaw support | Auth | Status | Last checked |
|------|---------|------------------|------|--------|--------------|
| [`telegram.md`](./telegram.md) | Telegram | yes | bot-token | active | 2026-04-20 |
| [`slack.md`](./slack.md) | Slack | yes | oauth | active | 2026-04-20 |
| [`discord.md`](./discord.md) | Discord | yes | bot-token | active | 2026-04-20 |
| [`whatsapp.md`](./whatsapp.md) | WhatsApp | yes | oauth | active (paid) | 2026-04-20 |
| [`signal.md`](./signal.md) | Signal | yes | app-password | active | 2026-04-20 |
| [`bluebubbles-imessage.md`](./bluebubbles-imessage.md) | BlueBubbles (iMessage) | partial | app-password | needs-setup (macOS-only) | 2026-04-20 |
| [`matrix.md`](./matrix.md) | Matrix | broken | app-password | **broken since 2026.2.17** | 2026-04-20 |
| [`telegram-voice-group.md`](./telegram-voice-group.md) | Telegram Voice (Group Calls) | partial | bot-token | active (via ClawHub skill) | 2026-04-20 |

## Subcategories

- **messaging** — text-first channels (Telegram, Slack, Discord, WhatsApp, Signal, BlueBubbles, Matrix)
- **voice** — audio/voice-chat extensions (Telegram Voice Groups)
- **async-forum** — threaded forum-style channels (Telegram Forums — see `deploy-telegram-forum.md`; reuses the `telegram.md` base entry)

## Quick pick

- **Default for individual principal:** Telegram (free, reliable, Forums support)
- **Team already in Slack:** Slack (mention-only mode)
- **Team already in Discord:** Discord
- **International / consumer-facing:** WhatsApp (paid, per-conversation)
- **Privacy-critical / regulatory:** Signal (dedicated phone number required)
- **iMessage-only US client:** BlueBubbles (needs a Mac relay)
- **Self-hosted / federated:** Matrix — **but native integration is broken on current OpenClaw**, use raw API only
- **Voice meeting capture on Telegram:** `telegram-voice-group` ClawHub skill on top of Telegram

## How to add a new channel

1. Copy [`_template.md`](./_template.md) → `<channel>.md`
2. Fill in the frontmatter YAML fields (never fabricate `openclaw_native_support` — verify with `openclaw channels --help` on a live install)
3. Write the body (What it is / Native support / Auth / Install / Wire into skills / Rate limits / Known issues / When to pick / Citations)
4. Add a row to the Index table in this README
5. If any `deploy-*` skill in the parent directory is a natural consumer, cross-reference it in "Wire into skills"
6. If the channel is broken, mark `maintenance_status: broken` and call out the breaking version in the body

## For audit / update-checker agents

Every channel file has YAML frontmatter at the top with machine-parseable fields:

- `name`, `category: channel`, `subcategory`
- `openclaw_native_support` (`yes` / `partial` / `broken` / `no`)
- `openclaw_cli`, `raw_api_fallback`, `auth_type`
- `version_last_verified`, `last_checked` (bump when you re-check)
- `maintenance_status` (`active` / `broken` / `deprecated` / `needs-setup`)
- `cost_tier`, `privacy_tier`
- `requires` (array), `docs_urls` (array)

An audit agent should:
1. Parse every `channels/*.md` frontmatter (skip `_template.md` and this `README.md`).
2. Run `openclaw channels --help` + `openclaw channels list` on a current install; diff against the `openclaw_native_support` field.
3. For each file marked `broken`, re-check whether the upstream issue is resolved — update `maintenance_status` and note version.
4. Hit each channel's `docs_urls` for deprecation notices or API version changes (WhatsApp Graph API version, Telegram Bot API minor, etc.).
5. Update `version_last_verified` + `last_checked` in the frontmatter.
6. Report a summary PR of channels that moved (broken → active, active → broken, new channel added to `openclaw channels`).

No single file needs to be re-read to audit another. Each channel is independently verifiable.

## Known-broken snapshot (2026-04-20)

- **Matrix** — native integration broken since 2026.2.17 per baseline §M #9. Use raw Client-Server API.
- **BlueBubbles** — bundled skill flagged `△ needs setup`; workable but not turnkey.
- **Telegram Voice** — depends on ClawHub skill `telegram-voice-group`; install path unverified on 2026.4.15.

## Unverified items (flagged in individual files)

- Discord: no known OpenClaw-side bugs, but failure mode is usually silent — confirm send success in-band.
- Signal: whether OpenClaw ships signal-cli as a sidecar or expects operator install.
- WhatsApp: whether a dedicated deploy-* skill exists for the 2026.4.18 multi-account flow.
- Telegram Voice: ClawHub skill install path on 2026.4.15; multi-speaker transcription support.
