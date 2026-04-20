---
name: "<Channel name>"
category: "channel"
subcategory: "<messaging | voice | async-forum>"
openclaw_native_support: "<yes | partial | broken | no>"
openclaw_cli: "openclaw channels send --channel <id>"
raw_api_fallback: "<e.g. https://api.telegram.org/bot... for Telegram>"
auth_type: "<bot-token | oauth | app-password | webhook>"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "<active | broken | deprecated | needs-setup>"
cost_tier: "<free | rate-limited | paid>"
privacy_tier: "<cloud | hybrid | local>"
requires:
  - "<e.g. Bot token from BotFather>"
  - "<e.g. macOS for BlueBubbles>"
docs_urls:
  - "<primary docs URL>"
---

# <Channel name>

## What it is

<One paragraph — what this channel is and how OpenClaw integrates with it.>

## Native OpenClaw support

<yes / partial / broken — what works via `openclaw channels`, what needs raw API>

## Authentication

<Bot tokens, OAuth flow, app passwords, etc. Exact steps to get credentials.>

## Install / configure

```bash
# Pre-flight — check OpenClaw supports this channel
openclaw channels --help | grep -i <channel>

# Configure the channel
openclaw channels add --type <channel-type> --name <name> ...
# or edit openclaw.json directly if the CLI doesn't support this channel yet
```

## Wire into skills

<Which skills in the library use this channel? How do they configure it?>

## Rate limits

<Channel-specific rate limits and headers.>

## Known issues / gotchas

<Known bugs per baseline §M if relevant, community-reported auth issues, etc.>

## When to pick this channel

<What makes this channel a good choice for specific use cases.>

## Citations

- <OpenClaw docs URL>
- <Platform API docs>
- <GitHub issues if relevant>
