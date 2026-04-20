---
name: "BlueBubbles (iMessage)"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "partial"
openclaw_cli: "openclaw channels send --channel bluebubbles  # needs-setup"
raw_api_fallback: "http://<mac-host>:1234/api/v1/message/text (BlueBubbles Server REST)"
auth_type: "app-password"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "needs-setup"
cost_tier: "free"
privacy_tier: "hybrid"
requires:
  - "macOS machine (physical Mac) signed into the target iMessage account"
  - "BlueBubbles Server app installed on that Mac"
  - "BlueBubbles password/API key"
  - "Network reachability from OpenClaw host to the BlueBubbles Server"
docs_urls:
  - "https://bluebubbles.app/"
  - "https://documentation.bluebubbles.app/"
---

# BlueBubbles (iMessage)

## What it is

Third-party bridge that turns a physical Mac into an iMessage relay. The Mac runs the BlueBubbles Server app (which hooks into Messages.app via AppleScript + private APIs). OpenClaw talks to BlueBubbles Server over REST/Socket.IO to send and receive iMessages. The bundled OpenClaw skill for BlueBubbles shows `△ needs setup` status — the CLI surface exists but the install isn't turnkey.

## Native OpenClaw support

**partial / needs-setup** — OpenClaw ships a BlueBubbles skill stub that declares support, but end-to-end requires a running BlueBubbles Server that OpenClaw doesn't install for you. You're responsible for the Mac, the Messages.app sign-in, the BlueBubbles Server app, and networking.

## Authentication

1. Dedicated Mac (Mac mini M-series works great) signed into the iCloud / Apple ID that owns the iMessage account.
2. Install BlueBubbles Server from `bluebubbles.app/downloads`.
3. On first run it generates a password/API key — save it.
4. Grant Messages.app + Accessibility + Full Disk Access permissions in System Settings → Privacy.
5. In OpenClaw: `channels.bluebubbles.url` (e.g. `http://mac-mini.local:1234`) + `channels.bluebubbles.password`.

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -iE 'bluebubbles|imessage'
openclaw channels list | grep -iE 'bluebubbles|imessage'

# Verify server reachable
curl -sS "http://${BB_HOST}:1234/api/v1/server/info?password=${BB_PASSWORD}"

# Send a test
openclaw channels send --channel bluebubbles --to "+14155551234" --text "hello"
```

## Wire into skills

No base skill routes primarily through BlueBubbles. Use it as an additional channel when the client's principal is iMessage-only (common for iPhone-only US clients). Pair with Telegram or Slack as the primary.

## Rate limits

- No formal rate limit from BlueBubbles — limited by Messages.app's own send cadence (informally ~1 message/sec sustained)
- Apple-side: bursts can trigger iMessage "Not Delivered" errors or temporary throttles
- Attachments: Apple's iMessage size limits apply (~100 MB for video, varies)

## Known issues / gotchas

- **macOS only** — Linux/Windows OpenClaw hosts cannot run BlueBubbles Server. Client machine is a Mac, or deploy a Mac mini as a dedicated iMessage relay.
- Messages.app sign-in drops periodically (2FA prompts, iCloud session expiry) — BlueBubbles can't message during these. Plan for monitoring.
- Private API usage means Apple can break it with any macOS update — BlueBubbles Server releases lag macOS by days to weeks.
- `needs-setup` status in the bundled skill means there's no "one-command install" — expect manual config.
- Not E2EE-preserving: message passes through BlueBubbles plaintext on the LAN/WAN between the Mac and OpenClaw host (unless you tunnel via Tailscale/Cloudflare Tunnel).

## When to pick this channel

Client's world is iMessage and they won't migrate to Telegram. Common for US consumer clients, iPhone-first users, and anyone whose social graph is locked in iMessage. Accept the Mac-relay overhead.

## Citations

- https://bluebubbles.app/
- https://documentation.bluebubbles.app/
- OpenClaw bundled skills listing (shows `△ needs setup` for bluebubbles)
