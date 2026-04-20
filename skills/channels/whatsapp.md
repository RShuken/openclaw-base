---
name: "WhatsApp"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "yes"
openclaw_cli: "openclaw channels send --channel whatsapp"
raw_api_fallback: "https://graph.facebook.com/v20.0/<PHONE_NUMBER_ID>/messages"
auth_type: "oauth"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "active"
cost_tier: "paid"
privacy_tier: "cloud"
requires:
  - "Meta Business account + verified phone number"
  - "WhatsApp Cloud API access token + phone_number_id"
  - "Approved message templates for outbound conversations"
docs_urls:
  - "https://developers.facebook.com/docs/whatsapp/cloud-api"
---

# WhatsApp

## What it is

WhatsApp Business Cloud API integration. OpenClaw `channels send --channel whatsapp --to <e164_phone>` delivers to WhatsApp users. As of **2026.4.18** changelog (baseline §4C.7 version notes), OpenClaw supports **multi-account isolation** — one agent can hold multiple WhatsApp Business numbers without session cross-contamination. Inbound messages arrive via Meta webhook.

## Native OpenClaw support

**yes** — baseline §4C.7 lists WhatsApp among the native `openclaw channels` channels, and 2026.4.18 shipped the multi-account isolation feature specifically for WhatsApp.

## Authentication

1. Meta Business Suite → create a WhatsApp Business app
2. Verify a phone number (separate from any personal WhatsApp)
3. System user → generate a permanent access token with `whatsapp_business_messaging` + `whatsapp_business_management` scopes
4. Note the `phone_number_id` (NOT the phone number itself) and `business_account_id`
5. Save to `channels.whatsapp.accessToken` + `channels.whatsapp.phoneNumberId` in openclaw.json
6. Configure webhook URL in Meta dashboard for inbound messages

## Install / configure

```bash
# Pre-flight
openclaw channels --help | grep -i whatsapp
openclaw channels list | grep -i whatsapp

# Send (recipient in E.164: +14155551234)
openclaw channels send --channel whatsapp --to "+14155551234" --text "hello"
```

## Wire into skills

- `deploy-messaging-setup.md` — listed as a supported secondary platform
- No WhatsApp-specific deploy-* skill in base yet — unverified whether multi-account flow has a dedicated skill post-2026.4.18

## Rate limits

- Cloud API: 80 messages/sec per phone number by default; can scale to 1000/sec with tier upgrade
- Business-initiated conversations require approved message templates
- 24-hour session window for free-form replies after a user-initiated message
- Pricing: per-conversation, varies by country and category (marketing / utility / authentication)

## Known issues / gotchas

- **Paid channel** — every business-initiated conversation costs money per Meta pricing. Unlike Telegram/Discord, you cannot assume free-tier.
- 24-hour window: if the user hasn't messaged the agent in 24h, outbound must use a pre-approved template (free-form text is rejected).
- Template approval is a manual Meta review process — can take hours to days.
- Multi-account isolation is 2026.4.18+ — earlier versions had session bleed between numbers (unverified scope of impact).
- Personal WhatsApp automation (non-Business API) is against ToS; don't attempt it.

## When to pick this channel

Client's customers/principal live in WhatsApp (common outside US) and SMS/Telegram aren't acceptable substitutes. Paid channel — justify the per-conversation cost.

## Citations

- https://developers.facebook.com/docs/whatsapp/cloud-api
- https://developers.facebook.com/docs/whatsapp/pricing
- `audit/_baseline.md` §4C.7 (2026.4.18 multi-account isolation)
