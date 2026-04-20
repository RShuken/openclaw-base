# Install OpenClaw

Install OpenClaw from scratch on a single machine. This is the first skill that runs on any new box — everything else depends on it.

> **Rewritten 2026-04-20.** Previous version had a 270-line Phase 1 for OpenAgent Connect (OAC) device enrollment before the actual OpenClaw install. OAC is a separate system and lives in `RShuken/openagent-connect` — it is intentionally NOT part of openclaw-base. This file covers **only** the OpenClaw install itself.

## Purpose

Install OpenClaw using the official installer, run onboarding, verify everything is running, and leave the machine ready for skill deployment.

**When to use:** First-time setup on a fresh macOS/Linux/Windows machine, or reinstalling after a clean OS install.

**What this skill does:**
1. Detects OS + architecture + existing OpenClaw install (if any)
2. Installs OpenClaw via the official one-line installer (or npm fallback)
3. Runs `openclaw onboard --install-daemon --non-interactive` to bootstrap the workspace, gateway daemon, and default model config
4. Verifies with `openclaw --version`, `openclaw health`, `openclaw doctor`

## Variables

| Variable | Source | Example |
|----------|--------|---------|
| `${OPENCLAW_DIR}` | Installer default | `~/.openclaw` |
| `${WORKSPACE}` | `agents.defaults.workspace` | `~/.openclaw/workspace` |

That's it. There are no bearer tokens, device IDs, session IDs, or enrollment keys at install time — those belong to OAC, not OpenClaw. If you need persistent remote access, install that separately after OpenClaw is working.

## Before You Start

Detect whether OpenClaw is already installed. **Do NOT install twice** — an older version is still OpenClaw and should be upgraded, not duplicated.

```
# macOS / Linux
which openclaw && openclaw --version 2>/dev/null || echo 'NOT_INSTALLED'
ls -d ~/.openclaw 2>/dev/null || echo 'NO_STATE_DIR'
```

```
# Windows (PowerShell)
Get-Command openclaw -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source
Test-Path "$env:USERPROFILE\.openclaw"
```

**Decision:**
- Installed + recent version (2026.4.15+): skip install, jump to onboarding if `~/.openclaw/openclaw.json` is missing
- Installed + older version: run the installer (it upgrades in place)
- Not installed: proceed with install

Also check Node availability (OpenClaw needs Node 22.14+; installer provides it if missing):

```
node --version 2>/dev/null || echo 'NO_NODE'
```

## Prerequisites

- Internet access to `https://openclaw.ai` (for the installer) and the chosen model provider
- macOS / Linux / WSL2 / Windows 10+/11
- Admin/sudo NOT required on macOS/Windows (user-level install). Linux systemd unit install may need sudo.

## What Gets Installed

| Component | Location | Purpose |
|-----------|----------|---------|
| OpenClaw CLI | npm global bin (or `~/.openclaw/bin` for local-prefix variant) | `openclaw` command |
| Config file | `~/.openclaw/openclaw.json` | Model stack, channel config, gateway settings |
| Workspace | `~/.openclaw/workspace/` | `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `HEARTBEAT.md`, `TOOLS.md`, `USER.md`, `BOOTSTRAP.md`, `memory/` |
| Gateway daemon | launchd (macOS) / systemd user unit (Linux) / Scheduled Task (Windows) | Keeps the gateway running on `ws://127.0.0.1:18789` |
| Auth store | `~/.openclaw/agents/main/agent/auth-profiles.json` | API credentials (never in openclaw.json) |

## Steps

### 1. Install OpenClaw

**macOS / Linux / WSL2:**
```
curl -fsSL https://openclaw.ai/install.sh | bash
```

**Windows (PowerShell):**
```
iwr -useb https://openclaw.ai/install.ps1 | iex
```

**Local-prefix variant** (Node + OpenClaw both under `~/.openclaw/`, useful when the client can't install Node globally):
```
curl -fsSL https://openclaw.ai/install-cli.sh | bash
```

**npm fallback** (if the installer is blocked by a corporate proxy):
```
npm install -g openclaw@latest
```

Expected: installer prints a success message and the `openclaw` binary is on PATH.

**If this fails:**
- `openclaw: command not found` after install — run `source ~/.bashrc` or `source ~/.zshrc`, or restart the terminal. On the local-prefix variant, `~/.openclaw/bin` needs to be on PATH.
- "Node.js not found" — installer handles Node itself, but if something goes wrong install manually: `curl -fsSL https://fnm.vercel.app/install | bash && fnm install 22`.
- Corporate proxy blocks `openclaw.ai` — use the npm fallback.
- Windows execution policy blocks the script — `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned`.

### 2. Verify install

```
openclaw --version
```

Expected: `OpenClaw 2026.4.15+ (...)` (or later).

### 3. Run onboarding

```
openclaw onboard --install-daemon --non-interactive
```

This:
- Creates `~/.openclaw/openclaw.json` with default model config
- Initializes `~/.openclaw/workspace/` with bootstrap files
- Installs the gateway daemon (launchd / systemd / schtasks depending on OS)
- Starts the gateway on `ws://127.0.0.1:18789`

**Interactive variant** (if the client wants to configure channels, model auth, etc. immediately):
```
openclaw onboard --install-daemon
```

The non-interactive variant is recommended — model auth and channels come later via `deploy-messaging-setup.md` / the agent's preferred identity skills.

### 4. Overwrite the default `openclaw.json` with our Codex template

The default config ships with a minimal model stack. Replace it with the openclaw-base template (Codex-first, primary-source verified — see `audit/_baseline.md` §BB and §Y):

```
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.default-backup
```

Then write `templates/openclaw.json` from this repo to `~/.openclaw/openclaw.json` using the base64 pattern from `_authoring/_deploy-common.md`.

Validate:
```
openclaw config validate
```

Expected: `Config valid: ~/.openclaw/openclaw.json`.

### 5. Verify gateway + health

```
openclaw health
```

Expected: gateway reachable, all probes pass. (Warnings about missing auth profiles are fine at this stage — those come from model-auth skills.)

```
openclaw doctor
```

Expected: health checks pass. Warnings about missing channel configs are fine.

```
openclaw gateway status
```

Expected: running.

> **Do NOT use `openclaw status`** — that command doesn't exist on 2026.4.15. Use `openclaw health` or `openclaw gateway status`.

### 6. Note what comes next

After install, the typical next deployments are:
- `deploy-identity.md` — give the agent its name, soul, heartbeat
- `deploy-messaging-setup.md` — connect Telegram / Discord / Slack / WhatsApp via `openclaw channels`
- `deploy-security-safety.md` — prompt-injection defenses, approval gates, nightly review
- `deploy-memory.md` — embedding provider + vector search across the workspace

Deploy in that order for the foundation.

## Verification

```
openclaw --version && openclaw health && openclaw gateway status && ls ~/.openclaw/workspace/ && openclaw config validate
```

Expected:
- Version string printed
- Health all green
- Gateway running
- Workspace shows `AGENTS.md`, `SOUL.md`, `IDENTITY.md`, `HEARTBEAT.md`, `TOOLS.md`, `USER.md`, `BOOTSTRAP.md`
- Config valid

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| `openclaw: command not found` after install | PATH not refreshed in current shell | `source ~/.bashrc` / `source ~/.zshrc`, or restart terminal |
| Installer fails with "Node.js not found" | Installer's Node detection failed | Install Node 22.14+ manually via `fnm` or nodesource |
| `openclaw onboard` hangs | Interactive prompts can't complete | Always use `--non-interactive` on first install |
| Gateway won't start (port 18789 in use) | Another process on the port | macOS/Linux: `lsof -i :18789`; Windows: `netstat -ano \| findstr 18789` — find + kill |
| `openclaw doctor` reports "auth profile missing" | Model credentials not configured yet | Fine at install time — deploy `deploy-messaging-setup.md` + model-auth skills next |
| macOS: "node" blocked by Gatekeeper | Quarantine attribute on bundled binary | `xattr -d com.apple.quarantine ~/.openclaw/bin/node` |
| Windows: execution policy blocks install | PowerShell default is Restricted | `Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned` |
| Corporate proxy blocks `openclaw.ai` | Firewall / SSL inspection | Use npm fallback: `npm install -g openclaw@latest` |
| `openclaw config validate` fails after copying our template | Unknown key rejected | Run `openclaw doctor --fix`. If keys still rejected, the template is ahead of the installed version — either upgrade OpenClaw or trim the template to match installed schema |

## Dependencies

- **Depends on:** nothing. This is the foundation.
- **Required by:** every other `deploy-*.md` skill.
- **Enhanced by:** `deploy-identity.md`, `deploy-messaging-setup.md`.
- **Intentionally NOT part of this skill:** OpenAgent Connect (OAC) remote access. That's a separate system — see `RShuken/openagent-connect` if remote operator access is needed after OpenClaw is installed.
