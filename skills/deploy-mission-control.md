# Deploy Mission Control Dashboard

Install and configure [openclaw-mission-control](https://github.com/robsannaa/openclaw-mission-control) — a self-hosted Next.js GUI for managing OpenClaw without touching the CLI.

## What It Does

Mission Control is a browser-based dashboard that talks directly to the local OpenClaw gateway. It provides:

- **Dashboard** — live overview of agents, gateway health, cron jobs, system resources
- **Chat** — talk to agents directly in the browser with file attachments and model selection
- **Cron Jobs** — create, edit, pause, test recurring tasks with full run history
- **Usage** — token and cost tracking across all models and agents
- **Agents** — interactive org chart of agent hierarchy, start/stop subagents
- **Memory** — view/edit agent long-term memory and daily journals, vector search
- **Models** — manage providers, credentials, fallback chains per agent
- **Doctor** — diagnostics with one-click fixes
- **Terminal** — built-in command line with multiple tabs
- **Channels** — configure Telegram, Discord, WhatsApp, Signal, Slack connections
- **Documents** — file browser across agent workspaces
- **Security** — audit reports and permission controls
- **Tasks** — Kanban board (Backlog, In Progress, Review, Done)
- **Tailscale** — remote access tunnel controls

## Philosophy

Mission Control is a **thin layer** — it reads/writes directly to OpenClaw, stores nothing itself. If it goes down, agents keep running. No database, no sync, no migrations.

## Prerequisites

- OpenClaw installed and gateway running
- Node.js 22+
- npm
- Git (to clone the repo)

## Adaptation Points

| Setting | Default | Alternatives |
|---------|---------|-------------|
| Port | 3333 | Any available port via `PORT=8080 ./setup.sh` |
| Host | 127.0.0.1 (loopback) | 0.0.0.0 for LAN access (not recommended without auth) |
| Gateway URL | http://127.0.0.1:18789 | Custom via `OPENCLAW_GATEWAY_URL` env var |
| Gateway token | Auto-detected | Set via `OPENCLAW_GATEWAY_TOKEN` env var |
| Install location | ~/.openclaw/openclaw-mission-control | Any path, but ~/.openclaw/ is conventional |
| Service manager | launchd (macOS) / systemd (Linux) | `--no-service` for manual start, nohup fallback |

## Deployment Steps

### Phase 1: Pre-flight

1. Verify OpenClaw is running: `openclaw --version && openclaw gateway call health`
2. Check if an existing dashboard is running on the target port: `lsof -i :3333`
3. If an old dashboard exists, stop it first:
   - macOS: `launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.dashboard.plist`
   - Linux: `systemctl --user stop openclaw-dashboard`

### Phase 2: Install

```bash
cd ~/.openclaw
git clone https://github.com/robsannaa/openclaw-mission-control.git
cd openclaw-mission-control
./setup.sh
```

The setup script will:
1. Install npm dependencies (including lightningcss native module)
2. Build the production bundle
3. Install and start a launchd service (macOS) or systemd service (Linux)
4. Print the URL (default: http://127.0.0.1:3333)

### Phase 3: Configure Gateway Token

If the gateway uses token auth (check `openclaw config get gateway.auth`):

```bash
# Get the gateway token
GATEWAY_TOKEN=$(openclaw config get gateway.auth.token 2>/dev/null | tr -d '"')

# Option A: Set in the launchd plist (macOS)
# Add to the EnvironmentVariables dict in com.openclaw.dashboard.plist:
#   OPENCLAW_GATEWAY_TOKEN = <token>

# Option B: Set in the shell environment
# Add to ~/.zshrc or ~/.bashrc:
export OPENCLAW_GATEWAY_TOKEN="<token>"
```

Then restart the dashboard service.

### Phase 4: Verification

1. Open `http://localhost:3333` in a browser
2. Verify the dashboard loads and shows:
   - Gateway status: connected
   - Agent list populated
   - Cron jobs visible
   - System resources (CPU, memory, disk) reporting
3. Test chat: send a message to the default agent
4. Check logs if issues: `tail -50 ~/.openclaw/openclaw-mission-control/.dashboard.log`

### Phase 5: Old Dashboard Cleanup (if applicable)

If replacing an older dashboard installation:
1. Stop old service: `launchctl bootout gui/$(id -u) ~/Library/LaunchAgents/com.openclaw.dashboard.plist`
2. Remove old plist if it points to a different path
3. The new setup.sh writes its own plist with the same label (`com.openclaw.dashboard`)

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| "OpenClaw not found" | Ensure `openclaw` is in PATH. Try: `OPENCLAW_BIN=$(which openclaw) npm run dev` |
| Port in use | `PORT=8080 ./setup.sh` or kill the existing process |
| WebSocket errors | Set `OPENCLAW_GATEWAY_TOKEN` to match `gateway.auth.token` in openclaw.json |
| lightningcss missing | `rm -rf node_modules package-lock.json && npm install --include=optional` then rerun setup |
| Blank page | Check `.dashboard.err.log` in the install dir |

## Environment Variables (all optional)

| Variable | Default | Purpose |
|----------|---------|---------|
| `OPENCLAW_HOME` | `~/.openclaw` | OpenClaw data directory |
| `OPENCLAW_BIN` | Auto-detected | Path to `openclaw` binary |
| `OPENCLAW_WORKSPACE` | Auto-detected | Default workspace folder |
| `OPENCLAW_TRANSPORT` | `auto` | Gateway transport: `auto`, `http`, or `cli` |
| `OPENCLAW_GATEWAY_URL` | `http://127.0.0.1:18789` | Gateway address |
| `OPENCLAW_GATEWAY_TOKEN` | _(empty)_ | Bearer token for gateway auth |

## Post-Install

After installation, the dashboard is a launchd/systemd service that:
- Starts automatically on boot
- Restarts on crash
- Binds to loopback only (127.0.0.1)
- Logs to `.dashboard.log` and `.dashboard.err.log` in the install directory

For remote access, use SSH tunneling:
```bash
ssh -N -L 3333:127.0.0.1:3333 user@host
```
