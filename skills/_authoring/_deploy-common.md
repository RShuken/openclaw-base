# Standard Deployment Protocol

The rules every `deploy-*.md` skill follows. This is the constitution for installing OpenClaw systems on a fresh machine. Not optional.

> **Rewritten 2026-04-19** against OpenClaw 2026.4.15 (live-verified on Mac Mini 4C). Replaces the pre-2026.4 protocol that was OAC-operator-centric and carried stale "Blocked Commands" claims. See `audit/skill-audit-2026-04-19.md` row 57 for what changed.

## Philosophy

Skills are **runbooks for an intelligent operator**, not shell scripts.

The operator (whichever AI model `openclaw.json` routes to — default stack is OpenAI Codex GPT-5.4 per `audit/_baseline.md` §BB) doesn't need exact commands copy-pasted. It needs to understand **what** to accomplish, **why** it matters, and **where exact syntax is critical**.

Skills operate at three levels of abstraction:

| Level | When to use | Example |
|-------|-------------|---------|
| **Level 1: Exact syntax** | Daemon registration, service configs, JSON schemas, security settings | `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.openclaw.plist` |
| **Level 2: Intent + reference command** | Most steps — show what AND why | "Create the workspace directory. Reference: `mkdir -p ${WORKSPACE}`" |
| **Level 3: Intent + knowledge** | High-level orchestration, decision points | "Schedule nightly sync configured for the client's timezone" |

**Default to Level 2–3.** Drop to Level 1 only where exact syntax matters for correctness.

## Phase 0.5: OpenClaw Compatibility Check (MANDATORY)

Every deploy skill must verify, at the start of deployment, that the OpenClaw CLI on the target machine actually supports the commands the skill will call. Skip this and you get silent failures or cascading improvisations.

### 1. Record the OpenClaw version

**Remote:**
```
openclaw --version
```

Expected: a version string (e.g., `OpenClaw 2026.4.15 (041266a)`). Skills in this repo target 2026.4.15+. If the client is on an older version, stop and ask the operator whether to upgrade or adapt.

### 2. List bundled skills

**Remote:**
```
openclaw skills list
```

Expected: a table showing bundled skills and their `ready` vs `needs setup` status. If the skill you're about to deploy duplicates a bundled skill (e.g., `channels`, `memory`, `coding-agent`), prefer the bundled one.

### 3. Inspect the config

**Remote:**
```
cat ~/.openclaw/openclaw.json
```

Do NOT use `openclaw config` with no args — that launches an interactive wizard that hangs over remote PTY. Use `openclaw config get <dotpath>` / `openclaw config set <dotpath> <value>` for scalar reads/writes, or edit the JSON directly with Python for structured changes.

### 4. Verify each openclaw subcommand the skill uses

**Remote:**
```
openclaw <subcommand> --help 2>&1 | head -3
```

Run this for every unique `openclaw <subcommand>` the skill calls. If any returns "unknown command," stop. Do not improvise a workaround — report the gap to the operator.

### 5. Available subcommands as of 2026.4.15

Reference list (live-verified on 4C, see `audit/_baseline.md` §4C.7):

`acp, agent, agents, approvals, backup, capability, channels, clawbot (legacy), completion, config, configure, cron, daemon (legacy), dashboard, devices, directory, dns, docs, doctor, exec-policy, gateway, health, hooks, infer, logs, mcp, memory, message, models, node, skills`

**Do NOT assume `openclaw status` exists** — it doesn't. Use `openclaw health` or `openclaw gateway status`.

**Do NOT assume `openclaw telegram` exists** — it's been absorbed into `openclaw channels` (and always-reliable raw Bot API calls work regardless).

## Model Defaults and Per-Skill Overrides

Every skill that routes through an LLM follows the model stack in `~/.openclaw/openclaw.json`:

```
agents.defaults.model.primary    → openai-codex/gpt-5.4
agents.defaults.model.fallbacks  → ordered Codex + OpenAI ladder
```

Template shipped with openclaw-base: `templates/openclaw.json`. Every documented model ID has been primary-source-verified against OpenAI's `/codex/models` and `/api/docs/models/all` (see `audit/_baseline.md` §Y).

**Do NOT hardcode model IDs in skill files.** Defer to the agent default. Every hardcoded `claude-opus-4-6`, `claude-sonnet-4-20250514`, `claude-3-haiku-20240307` in a skill is a ticking bomb — model deprecations happen monthly.

### Per-skill env-var overrides (the routing convention)

Skills that make LLM calls each expose **one env var** the operator can set to route that skill's calls through a different model than the agent default. Naming convention: `<SKILL_SHORT_NAME>_MODEL`. Examples:

| Skill | Env var | When to set |
|-------|---------|-------------|
| `deploy-urgent-email.md` | `URGENT_EMAIL_MODEL` | Route per-email classification to a cheap model (e.g., `openai-codex/gpt-5.4-nano`) |
| `deploy-advisory-council.md` | `COUNCIL_PERSONA_MODEL`, `COUNCIL_SYNTHESIS_MODEL` | Route the 8 personas cheap, synthesis to full model (skill has two env vars because it has two distinct call tiers) |
| `deploy-daily-briefing.md` | `BRIEFING_MODEL` | Route the morning synthesis to a different model than default |
| `deploy-platform-health.md` | `HEALTH_ANALYSIS_MODEL` | Route 9-area analysis cheaper |
| `deploy-food-journal.md` | `CORRELATION_MODEL` | Weekly correlation pass |
| `deploy-earnings.md` | `EARNINGS_NARRATIVE_MODEL` | Narrative generation |
| `deploy-knowledge-base.md` | `KB_QUERY_MODEL` | RAG query synthesis |
| `deploy-personal-crm.md` | `CRM_INTENT_MODEL` | Natural-language intent detection |
| `deploy-humanizer.md` | `HUMANIZER_MODEL` | Pattern-stripping pass |
| `deploy-fathom-pipeline.md` | `FATHOM_EXTRACTION_MODEL` | Action-item extraction |
| `deploy-video-analysis.md` | `VIDEO_ANALYSIS_MODEL` | Gemini video analysis |
| `deploy-video-research.md` | `VIDEO_RESEARCH_MODEL` | Research pass |
| `deploy-video-pipeline.md` | `VIDEO_PIPELINE_MODEL` | Pipeline synthesis |
| `deploy-content-pipeline.md` | `CONTENT_PIPELINE_MODEL` | Content generation |
| `deploy-security-safety.md` | `SECURITY_SCAN_MODEL` | Security scan AI pass |
| `deploy-health-monitoring.md` | `HEARTBEAT_TRIAGE_MODEL` | Alert-triage pass (only fires when a check fails) |

### The resolution pattern

Every LLM call-site in a skill uses this bash pattern:

```bash
MODEL_FLAG=""
if [ -n "${SKILL_MODEL:-}" ]; then
  MODEL_FLAG="--model ${SKILL_MODEL}"
fi
openclaw agent --agent main $MODEL_FLAG -m "<the prompt>"
```

- If the env var is set, the skill routes through that specific model
- If not, `--model` is omitted and `openclaw agent` uses `agents.defaults.model`
- Variant: for `node` scripts, read `process.env.SKILL_MODEL` and pass it through in the `spawn` call

### Where to set these env vars

When scheduling the skill via `openclaw cron add`, pass the env-var with `--env`:

```
openclaw cron add --name urgent-email --cron "*/5 * * * *" \
  --env URGENT_EMAIL_MODEL=openai-codex/gpt-5.4-nano \
  --command "${WORKSPACE}/scripts/urgent-email-scan.sh"
```

For manual invocations, set the env var in the shell first:

```
URGENT_EMAIL_MODEL=ollama/llama3.1:8b ${WORKSPACE}/scripts/urgent-email-scan.sh
```

### Why one env var per skill (not role-based abstractions)

Considered a role-based router (`classification`, `heartbeat`, `synthesis` → map to models centrally). Rejected: it's one more config file to learn + requires a shell helper + makes skill bodies less self-explanatory. Per-skill env vars are explicit, grep-able, and each skill stays portable without depending on external mapping.

If the operator wants a consistent "cheap-everywhere" strategy: set the relevant env vars on every cron job that needs the cheap path. If they want "agent default everywhere": set nothing.

Exception: if a skill genuinely needs a specific model that doesn't have a sensible fallback (e.g., `gpt-image-1` for image generation — text models won't substitute), pin it explicitly AND document why in the adaptation points table.

## Authentication

API credentials live in one place: `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`. This is the ONLY file the built-in subsystems (memory, models, channels) check.

| Key prefix | Provider | Type |
|-----------|----------|------|
| `sk-ant-api03-` | Anthropic | API key |
| `sk-ant-oat01-` | Anthropic | OAuth token (needs SDK refresh) |
| `AIza…` | Google (Gemini) | API key |
| `sk-…` (not `sk-ant-`) | OpenAI | API key |

OpenAI-Codex OAuth (ChatGPT subscription) uses a different flow: `openclaw models auth login --provider openai-codex --method oauth`. See skill `oauth-reauth.md`.

**NEVER use `openclaw auth paste-token` over a remote PTY** — the TUI hangs non-interactively. Write `auth-profiles.json` directly with Python:

```python
python3 -c "
import json, os
p = os.path.expanduser('~/.openclaw/agents/main/agent/auth-profiles.json')
with open(p) as f: auth = json.load(f)
auth['profiles']['default'] = {'type': 'api_key', 'provider': 'openai', 'key': '${API_KEY}'}
with open(p, 'w') as f: json.dump(auth, f, indent=2)
"
```

Validate key format BEFORE writing. A malformed key silently breaks every LLM-dependent skill in the workspace.

## File Transfer Standard

All file transfers to a remote machine MUST use base64. Heredocs through remote-command APIs get mangled — quotes, backslashes, dollar signs, backticks all get interpreted or escaped at multiple layers.

### macOS / Linux

```
echo '<base64-encoded-content>' | base64 -d > /path/to/file
```

### Windows (PowerShell)

```
[System.IO.File]::WriteAllBytes('C:\path\to\file', [Convert]::FromBase64String('<base64>'))
```

Do NOT use PowerShell `[System.Text.Encoding]::UTF8` — it adds a BOM that breaks `JSON.parse()`. Always go through `[Convert]::FromBase64String()` for clean bytes.

## Scheduling

Use `openclaw cron` (available on 2026.4.15, verified live §4C.3). Subcommands: `add | list | edit | enable | disable | rm | run | runs | status`.

Example:
```
openclaw cron add --name daily-briefing --cron "0 7 * * *" --tz America/Los_Angeles --message "Run morning briefing"
```

### Heartbeat vs cron

| Use | When |
|-----|------|
| **Heartbeat** (default every 30 min) | Periodic awareness sweeps, batchable checks, anything that can share context with prior check |
| **Cron** (precise timing) | Exact-time events (7:00 briefing), isolated-session tasks, one-shot scheduled jobs |

Per community research (`audit/_baseline.md` §L), collapsing 5 periodic-check cron jobs into ONE heartbeat turn is ~5x cheaper. Fan out only when timing truly matters.

**Known issue:** `agents.defaults.heartbeat.model` override is broken — heartbeat always uses primary model despite config (GitHub openclaw/openclaw#9556, baseline §M). To route heartbeats to a cheap model, set `agents.defaults.model.primary` to a local Ollama model and escalate on demand, not vice versa.

## Variables

Skills use these placeholders. Resolve them from the client's config before running the skill:

| Variable | Default | Source |
|----------|---------|--------|
| `${WORKSPACE}` | `~/.openclaw/workspace` | `agents.defaults.workspace` in openclaw.json |
| `${OPENCLAW_DIR}` | `~/.openclaw` | `OPENCLAW_STATE_DIR` env var |
| `${AGENT_USER}` | (from `whoami`) | Pre-flight discovery |
| `${AGENT_HOME}` | (from `echo $HOME`) | Pre-flight discovery |
| `${TIMEZONE}` | `America/Los_Angeles` | `IANA` tz string; ask the client |
| `${TELEGRAM_GROUP}` | (none) | Client messaging config in openclaw.json `channels.telegram` |
| `${EMAIL_PROVIDER}` | Gmail via `@steipete/gog` | Client preference |
| `${CALENDAR_PROVIDER}` | Google Calendar via `@steipete/gog` | Client preference |

Variables expand **on the operator side** before the command is sent. Do NOT send `${VARIABLE}` strings expecting the remote shell to expand them — the remote-command APIs ship the string literally.

## Platform Adaptation

Skills describe **intent** in reference bash. The operator adapts to the target platform.

| Intent | macOS/Linux | Windows (PowerShell) |
|--------|-------------|----------------------|
| Create dir tree | `mkdir -p path/to/dir` | `New-Item -ItemType Directory -Force -Path 'path\to\dir'` |
| Read file | `cat file.txt` | `Get-Content file.txt` |
| Write file (base64) | `echo '<b64>' \| base64 -d > file` | `[System.IO.File]::WriteAllBytes('file', [Convert]::FromBase64String('<b64>'))` |
| Check file exists | `test -f file && echo exists` | `Test-Path file` |
| Delete recursively | `rm -rf path` | `Remove-Item -Recurse -Force path` |
| Env var | `echo $VAR` | `$env:VAR` |
| Home dir | `$HOME` | `$env:USERPROFILE` |
| Current user | `whoami` | `$env:USERNAME` |
| Schedule | `openclaw cron add ...` (preferred), `launchd` / `crontab` (fallback) | `openclaw cron add ...`, `schtasks` (fallback) |

## Platform Gotchas

Hard-won lessons. Read the relevant section before deploying.

### macOS

| Gotcha | Workaround |
|--------|-----------|
| Fresh Macs show Xcode CLT GUI dialog on first `git`/`gcc` call (blocks remote PTY) | Run `xcode-select --install` first; if dialog appears, client must click "Install" |
| `launchd` does NOT expand `~` or `${HOME}` in plist paths | Use absolute paths: `/Users/<user>/...` |
| `crontab -e` and `crontab /file` hang on macOS 15.x via remote PTY | Use `openclaw cron add` (native) or `launchd` (fallback), never `crontab` over remote |
| `sudo` prompts for password → hangs PTY | Never use `sudo` over remote session. Use `~/.openclaw-*` user-level paths. |
| Machine sleep drops WebSocket sessions | Use `openclaw devices` exec API (survives sleep/wake via Hibernation API) |

### Linux

| Gotcha | Workaround |
|--------|-----------|
| `curl` may not be pre-installed on minimal images | `which curl` first; install with `apt install -y curl` |
| `systemctl --user` may fail in SSH sessions without linger | `loginctl enable-linger ${USER}` |
| Gateway reads env from `~/.openclaw/.env`, NOT the workspace `.env` | Write gateway-relevant env vars to `~/.openclaw/.env` |

### Windows

| Gotcha | Workaround |
|--------|-----------|
| PowerShell `[System.Text.Encoding]::UTF8` adds BOM, breaks `JSON.parse()` | Use `[System.IO.File]::WriteAllBytes()` + `[Convert]::FromBase64String()` |
| `schtasks /TR` fails with spaces in paths | Wrap target in a `.bat` file |
| Emoji in `Set-Content` returns exit=1 | Use base64 path above |
| Windows commands return `\r\n` line endings | `.replace(/\r\n/g, "\n")` before string compare |
| ~1000-char command length limit (PTY MAX_CANON) | Split into chunks or base64-wrap the payload |

### OpenClaw General

| Gotcha | Workaround |
|--------|-----------|
| `openclaw config set` can't set arrays | Edit JSON directly with Python; `openclaw doctor --fix` normalizes after |
| Unknown keys in openclaw.json make every CLI command fail schema validation | Only use `openclaw config set` for known keys; if editing JSON directly, modify only existing paths |
| `models.providers.<id>` overrides need full schema (`baseUrl`, `name`, etc.) — minimal partial overrides are rejected | Known limitation as of 2026.4.15; see GitHub openclaw/openclaw#8506 |
| `agents.defaults.heartbeat.model` override is ignored (silently falls through to primary) | Set `primary` to cheap model, escalate on demand (GitHub #9556) |

### Secret storage — **NEVER migrate OpenClaw .env to macOS Login Keychain**

Hard-won lesson from a real 64-hour outage (2026-03-08). If an AI-generated security audit recommends "move API keys from `.env` to macOS Keychain for better security," **the recommendation is wrong for any LaunchAgent-based OpenClaw deployment.**

**Root cause:** macOS Login Keychain requires an active, unlocked user session to access. LaunchAgents run at boot as background processes before any user logs in and cannot access the Login Keychain. Migrating secrets to Keychain silently breaks every background service that depends on API keys (memory, models, channels, all cron/heartbeat skills).

**Correct secret storage for OpenClaw LaunchAgent deployments:**
- `~/.openclaw/.env` with file permissions `600` (owner-only read/write)
- Protected by FileVault full-disk encryption (whole-disk encryption makes plaintext-at-rest a non-issue on a stolen/recovered disk)
- This is the OpenClaw community standard and is SECURE when both conditions are met

**Rule for security-audit automation:** before flagging `.env`/plaintext/keychain as a finding, check `fdesetup status` (FileVault ON?) and `stat -f "%OLp" ~/.openclaw/.env` (owner-only perms?). If both conditions are met, the finding is a false positive — downgrade to INFO severity or suppress. Do NOT recommend moving secrets to Keychain for OpenClaw deployments.

## Autonomy Modes

Skills support three operator autonomy levels. Declare your mode at the start of deployment.

| Mode | Behavior | When |
|------|----------|------|
| **Auto** | Proceed silently except `[HUMAN_INPUT]` steps | Trusted operator, familiar system, routine redeploy |
| **Guided** | Confirm before each phase | First deployment, unfamiliar system |
| **Manual** | Confirm before each command | Training, debugging, client watching |

Step markers:
- `[AUTO]` — proceed in all modes
- `[GUIDED]` — confirm in guided/manual, silent in auto
- `[HUMAN_INPUT]` — always needs human input (API keys, verification codes)

## Error Handling

Every significant step has:
- **Expected:** what success looks like
- **If this fails:** what to check

Common patterns:
- **Command not found:** check PATH, verify prerequisite installed
- **Permission denied:** check file ownership. Never `sudo` over remote macOS session.
- **File already exists:** usually fine (idempotency). Check content before overwriting.
- **Connection refused:** verify target service running, check port
- **Config validation error:** the key doesn't exist in this OpenClaw version. STOP. Do not edit openclaw.json directly as a workaround.

## Idempotency

Skills must be safe to re-run. The pattern:

```
Check if [thing] already exists →
  Yes + correct → Skip, note "already configured"
  Yes + wrong → Backup existing, apply new
  No → Create/install
```

- **Files:** `diff` before write, backup on mismatch
- **Directories:** `mkdir -p` already idempotent
- **Cron jobs:** `openclaw cron list` → check for duplicate names before adding
- **Database tables:** `CREATE TABLE IF NOT EXISTS`
- **Service registration:** check if loaded, update only if config changed
- **npm/pip installs:** safe to re-run; use `--upgrade` for specific versions

## After Deployment

1. **Verify** — every skill's Verification section confirms end-to-end flow works
2. **Document** — update the client's profile or agent notes with what was installed
3. **Follow-up** — note any skills that SHOULD be deployed next for full functionality

---

## What this file does NOT cover

- OAC-operator-specific flows (session commands, device exec, bearer tokens, client-profile management) — those live in the OpenAgent Connect repo at `RShuken/openagent-connect`
- Telegram operator-interaction patterns (polling for client replies, webhook trap/restore) — same
- Per-client deployment state resumption — same

openclaw-base is about installing OpenClaw on a fresh box and deploying skills into its workspace. Nothing more.
