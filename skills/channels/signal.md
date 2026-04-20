---
name: "Signal"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "yes"
openclaw_cli: "openclaw channels send --channel signal"
raw_api_fallback: "signal-cli JSON-RPC / REST wrapper (self-hosted)"
auth_type: "app-password"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "free"
privacy_tier: "hybrid"
requires:
  - "Dedicated Signal phone number (bot cannot share a human's number)"
  - "signal-cli or signald backend registered with that number"
  - "Verification SMS/voice to complete registration"
docs_urls:
  - "https://github.com/AsamK/signal-cli"
  - "https://signald.org/"
---

# Signal

## What it is

End-to-end encrypted messaging. OpenClaw supports Signal as a channel (per baseline §4C.7) but under the hood it bridges to `signal-cli` or `signald` — Signal has no first-party bot API. The agent's phone number becomes a real Signal account; anyone with that number can message it. Good for privacy-sensitive deployments where Telegram/WhatsApp are off the table.

## Native OpenClaw support

**yes** — listed in baseline §4C.7. The bridge model means "native" here means OpenClaw handles the signal-cli/signald wiring; it doesn't mean OpenClaw reimplements the Signal protocol.

## Authentication

1. Obtain a dedicated phone number (Twilio, Google Voice with SMS, or a physical SIM). Signal doesn't allow two accounts on one number.
2. Install signal-cli or signald on the OpenClaw host:
   ```bash
   # signal-cli
   brew install signal-cli  # macOS
   # or download from github.com/AsamK/signal-cli/releases
   ```
3. Register the number:
   ```bash
   signal-cli -a "+14155551234" register
   # receive SMS code, then:
   signal-cli -a "+14155551234" verify 123456
   ```
4. Configure OpenClaw: `channels.signal.number` + `channels.signal.backend` in openclaw.json.

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -i signal
openclaw channels list | grep -i signal

# Send a test
openclaw channels send --channel signal --to "+14155559999" --text "hello"
```

## Wire into skills

No Signal-specific deploy-* skill in the base library. Route through generic `openclaw channels send --channel signal` from any skill that supports `${NOTIFICATION_CHANNEL}` parameterization (e.g., `deploy-health-monitoring.md`).

## Rate limits

- Signal has aggressive anti-abuse rate limits, not formally documented
- Sending to many recipients in quick succession will trigger captcha challenges or temporary account locks
- Group messages: 1001 members max per group
- Attachments: 100 MB limit per file

## Known issues / gotchas

- **Dedicated phone number required** — no sharing with a human's Signal account. Budget a Twilio or similar number as part of install.
- Signal may require re-verification periodically or if the account is flagged.
- signal-cli/signald need to stay running; treat as a service (launchd/systemd). Dying mid-message loses it.
- Rate-limit triggers can result in **hours-long account locks** — do not batch blast. 1 message/second is the informal safe ceiling.
- Unverified: whether OpenClaw 2026.4.15 ships signal-cli as a sidecar or expects the operator to install it separately. Check `openclaw channels add --type signal --help` on target.

## When to pick this channel

Client is privacy-paranoid, operates under regulatory constraint (legal, medical, activist), or is in a region where Telegram/WhatsApp are unacceptable. Accept the phone-number cost and setup overhead.

## Citations

- https://github.com/AsamK/signal-cli
- https://signald.org/
- `audit/_baseline.md` §4C.7
