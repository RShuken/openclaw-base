# Standard Deployment Protocol — {{FAMILY_NAME}} Ventures

> **📌 REFERENCE SNAPSHOT — NOT FOR DEFAULT DEPLOYMENT.** This file was preserved from the {{PRINCIPAL_NAME}} / {{COMPANY_NAME}} engagement as a reference example of how a custom office/VC deployment was built. **Do NOT install this skill as part of a generic OpenClaw deployment.** It may contain engagement-specific defaults (emails, chat IDs, team names, VC-flavored workflows), Claude Opus routing (we default to Codex per `audit/_baseline.md` §BB in the parent repo), or stale "Blocked Commands" claims from 2026.2.26 that were resolved in 2026.4.15+. Use as inspiration when building custom-office skills, not as a default.

All `deploy/` skills for this client follow this protocol.

## Connection

{{AGENT_NAME}}'s Mac Mini is accessed via OpenAgent Connect device exec API.
Credentials are in `../../credentials.md`.

### How It Works

OAC has **two connection modes**. {{AGENT_NAME}} uses the **enrolled device** mode:

| Mode | When | Transport | Persistence |
|------|------|-----------|-------------|
| **Session/Token** (one-liner) | First-time setup, unenrolled devices | HTTP polling | Temporary — dies when process exits |
| **Enrolled Device** (WebSocket) | After enrollment, production use | Persistent WebSocket | LaunchAgent keeps it alive across reboots |

**{{AGENT_NAME}} is enrolled.** Commands go through the device exec API, which pushes them over a persistent WebSocket. The agent auto-reconnects with exponential backoff (1s → 60s max).

### Prerequisites for Device Exec

The OAC agent must be running on {{AGENT_NAME}} in enrolled mode. Verify:
1. LaunchAgent is loaded: `launchctl list | grep com.openagent.agent`
2. Agent is running: `ps aux | grep agent.js`
3. Config exists: `cat /Users/edge/.openclaw/config.json` (must have `server`, `deviceId`, `enrollmentKey`, `clientId`)

If the agent is NOT running after a reboot, check:
- Is the LaunchAgent plist at `~/Library/LaunchAgents/com.openagent.agent.plist`?
- Is the agent binary at `~/.openagent/agent/`?
- Check logs: `cat ~/.openagent/agent.stderr.log`

**DO NOT use the one-liner (`curl .../connect/...`) to reconnect enrolled devices.** That creates a session-based connection that bypasses the device exec API. Instead, restart the LaunchAgent: `launchctl kickstart -k gui/$(id -u)/com.openagent.agent`

### Sending Commands

```bash
# Login (bearer token expires after ~2 hours)
BEARER=$(curl -sS -X POST "https://api.openclawinstall.net/api/operator/login" \
  -H "content-type: application/json" \
  -d '{"operatorId":"owner-1","accessCode":"<from credentials.md>"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['accessToken'])")

# Send command (returns 409 DEVICE_OFFLINE if agent isn't connected)
curl -sS -X POST "https://api.openclawinstall.net/api/devices/0e586989-6bf4-44ba-a8c8-99a9666ada62/exec" \
  -H "content-type: application/json" \
  -H "authorization: Bearer $BEARER" \
  -d '{"command":"<command>"}'

# Poll results (use seq from exec response)
curl -sS "https://api.openclawinstall.net/api/devices/0e586989-6bf4-44ba-a8c8-99a9666ada62/results?afterSeq=N" \
  -H "authorization: Bearer $BEARER"
```

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `DEVICE_OFFLINE` on exec | Agent not running or WebSocket disconnected | Check `ps aux \| grep agent.js`. Restart LaunchAgent. |
| Exec returns 200 "queued" but no results | Agent connected but not processing (stale WS) | Kill agent, restart LaunchAgent. |
| `INVALID_ENROLLMENT_KEY` on check-in | Enrollment key changed or config.json is stale | Re-read key from server, update config.json. |
| One-liner works but device exec doesn't | One-liner uses session mode, not device WebSocket | Kill session agent. Start agent via LaunchAgent (enrolled mode). |

## {{AGENT_NAME}} System Reference

- **Machine:** Mac mini M4 Pro, 24 GB RAM, macOS 15.6
- **User:** edge
- **OpenClaw:** 2026.3.22 at `/opt/homebrew/bin/openclaw`
- **Workspace:** `/Users/edge/.openclaw/workspace/`
- **Config:** `/Users/edge/.openclaw/openclaw.json`
- **Cron jobs:** `/Users/edge/.openclaw/cron/jobs.json`
- **Primary model:** Claude Opus 4.6 (1M context)
- **Fallback:** GPT-5.4 Pro
- **Channels:** Telegram (bot 8784533237, user {{PRINCIPAL_CHAT_ID}})
- **Notion API:** ntn_596950270374... (in .zshrc)
- **Tavily API:** configured

Full system profile: `rel-hq/repos/rel-agent/clients/{{CLIENT_SLUG}}.md`

## Skill Format

Every deploy skill follows this structure:

```markdown
# Deploy [Skill Name]

## What This Does
One paragraph — what the user gets.

## Prerequisites
What must be deployed/working before this skill.

## Pre-Flight Checks
Commands to verify prerequisites are met. Run these first.

## Deployment Steps
Numbered steps. Each step has:
- What to do (intent)
- The command (exact syntax where it matters)
- What success looks like

## Verification
Checklist to prove it works end-to-end.

## Troubleshooting
Common failures and fixes.

## Rollback
How to undo if something goes wrong.
```

## Rules

1. **Research first** — Don't write a deploy skill until the research doc is done.
2. **Test on {{AGENT_NAME}}** — Every command is tested live before it goes in the skill.
3. **Verify every step** — If a step can fail silently, add a verification command.
4. **Document failures** — When something doesn't work, document why in the research doc.
5. **Write the SOP after** — The SOP is written after successful deployment, not before.
6. **Update the wiki** — After deployment, add relevant info to the wiki.
7. **Update the client profile** — After deployment, update `rel-hq/repos/rel-agent/clients/{{CLIENT_SLUG}}.md`.
