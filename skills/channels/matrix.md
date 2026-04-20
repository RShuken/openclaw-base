---
name: "Matrix"
category: "channel"
subcategory: "messaging"
openclaw_native_support: "broken"
openclaw_cli: "openclaw channels send --channel matrix  # BROKEN since 2026.2.17"
raw_api_fallback: "https://<homeserver>/_matrix/client/v3/rooms/<room_id>/send/m.room.message/<txn_id>"
auth_type: "app-password"
version_last_verified: "OpenClaw 2026.4.15"
last_checked: "2026-04-20"
maintenance_status: "broken"
cost_tier: "free"
privacy_tier: "hybrid"
requires:
  - "Matrix homeserver account (matrix.org or self-hosted Synapse/Dendrite)"
  - "Access token (login with password once, save token)"
  - "Room ID(s) the bot user has joined"
docs_urls:
  - "https://spec.matrix.org/latest/client-server-api/"
  - "https://github.com/openclaw/openclaw/issues"
---

# Matrix

## What it is

Federated, self-hostable messaging protocol — appealing for privacy-conscious or homelab deployments. OpenClaw nominally supports Matrix via `channels.matrix.*`, but **the native integration has been broken since OpenClaw 2026.2.17** (3+ versions at time of writing) per baseline §M. Raw Client-Server API via `curl` works fine and is the recommended fallback.

## Native OpenClaw support

**broken** — per baseline §M #9 and §M "Deprecation reports": "Matrix integration broken since 2026.2.17". As of 2026.4.15 the subcommand may still accept `--channel matrix` but send fails silently or with opaque errors. **Do not rely on the native integration.** Use raw API.

## Authentication

1. Create a Matrix account on matrix.org (or your own homeserver).
2. Login once to obtain an access token:
   ```bash
   curl -XPOST https://matrix.org/_matrix/client/v3/login \
     -d '{"type":"m.login.password","user":"bot","password":"..."}'
   ```
3. Save the returned `access_token` — this is the long-lived credential.
4. Join the target room from the bot account; note the room ID (`!abcdef:matrix.org`).

## Install / configure

```bash
# Pre-flight — confirm the known-broken status still holds on this version
openclaw channels --help | grep -i matrix
openclaw --version   # verify > 2026.2.17 (when it broke)

# Raw API send (works; bypasses the broken native path)
TXN=$(date +%s)
curl -XPUT "https://matrix.org/_matrix/client/v3/rooms/${ROOM_ID}/send/m.room.message/${TXN}" \
  -H "Authorization: Bearer ${ACCESS_TOKEN}" \
  -d '{"msgtype":"m.text","body":"hello from openclaw"}'
```

## Wire into skills

No skills in the base library currently route through Matrix. If a client requires Matrix, build a thin wrapper around the raw Client-Server API and call it from skills directly rather than depending on `openclaw channels send --channel matrix`.

## Rate limits

- Matrix.org public server: soft rate limits around 10 messages/sec per client, varies by homeserver
- Self-hosted Synapse/Dendrite: configurable in homeserver settings (`rc_message`, `rc_joins`, etc.)
- 429 returns `M_LIMIT_EXCEEDED` with `retry_after_ms`

## Known issues / gotchas

- **BROKEN in OpenClaw native path since 2026.2.17** (baseline §M #9). No public fix announced as of 2026.4.15. Track GitHub issues on openclaw/openclaw repo.
- If you must use Matrix today: raw Client-Server API only; do not waste time debugging `openclaw channels send --channel matrix`.
- Access tokens don't expire by default on matrix.org but are revoked on explicit logout — don't call `/logout` from the bot account.
- End-to-end encryption (E2EE) is NOT handled by raw API sends — bot will only work in unencrypted rooms unless you integrate an E2EE-capable SDK (matrix-nio, matrix-js-sdk).

## When to pick this channel

Client explicitly requires Matrix (privacy-centric, self-hosted, or federation-native). Otherwise default to Telegram — the native OpenClaw path is more reliable. Re-evaluate every OpenClaw release by checking if baseline §M still flags Matrix as broken.

## Citations

- `audit/_baseline.md` §M #9, line 345; §M Deprecation reports, line 362
- https://spec.matrix.org/latest/client-server-api/
- openclaw/openclaw GitHub issues (search "matrix")
