# Templates

Scaffolds for every file in OpenClaw's canonical workspace. Copy these into a new agent's workspace, fill in the `{{...}}` placeholders, and deploy.

## The canonical workspace files

Per `audit/_baseline.md` §5, OpenClaw's workspace loads these files on session start. `openclaw onboard` creates scaffolds but they're minimal — these templates are more complete starting points.

| File | Template | What it does | Where it goes after fill-in |
|------|----------|--------------|------------------------------|
| Model config | [`openclaw.json`](./openclaw.json) | Codex-default model stack + fallback ladder | `~/.openclaw/openclaw.json` |
| [`IDENTITY.md`](./IDENTITY.md) | ✅ | Name, owner, boundaries, approval gates | `~/.openclaw/workspace/IDENTITY.md` |
| [`SOUL.md`](./SOUL.md) | ✅ | Voice, personality, humor, anti-patterns | `~/.openclaw/workspace/SOUL.md` |
| [`HEARTBEAT.md`](./HEARTBEAT.md) | ✅ | Recurring pulses (cron-style) | `~/.openclaw/workspace/HEARTBEAT.md` |
| [`MEMORY.md`](./MEMORY.md) | ✅ | Curated long-term memory index (Hot tier, <200 lines) | `~/.openclaw/workspace/MEMORY.md` |
| [`AGENTS.md`](./AGENTS.md) | ✅ | Boot sequence, rules, approval gates, session-type routing | `~/.openclaw/workspace/AGENTS.md` |
| [`USER.md`](./USER.md) | ✅ | Who the operator is (person profile, preferences, hard nos) | `~/.openclaw/workspace/USER.md` |
| [`TOOLS.md`](./TOOLS.md) | ✅ | Available tools + how to pick the right one | `~/.openclaw/workspace/TOOLS.md` |
| [`BOOTSTRAP.md`](./BOOTSTRAP.md) | ✅ | First-session setup ritual (delete after first run) | `~/.openclaw/workspace/BOOTSTRAP.md` |

## Defaults

- **Model stack:** OpenAI Codex GPT-5.x primary with Codex fallback ladder. Claude is intentionally NOT in the default stack (cost). Add per-deployment only if a skill specifically needs it.
- **Identity philosophy:** under 50 lines. Long IDENTITY files crowd the context window and dilute the agent's character.
- **MEMORY.md philosophy:** Hot tier < 200 lines. See `skills/memory-options/community-patterns.md` for the Hot/Warm/Cold tiering pattern.
- **Heartbeats:** run as `openclaw cron add` jobs (native, 2026.4.15+), NOT inside long-lived openclaw sessions.

## Security rules (apply across all templates)

- **MEMORY.md never loads in group chats or sub-agent sessions.** AGENTS.md boot sequence enforces this. Baseline §§K.153, K.405, K.430.
- **Secrets never live in workspace files.** API keys belong in `~/.openclaw/agents/<agent>/agent/auth-profiles.json`. See `skills/_authoring/_deploy-common.md` "Secret storage" for the Keychain-migration-is-wrong lesson.
- **Approval gates for external actions.** Email, public posts, deletions, spending, production writes — all require explicit operator approval per AGENTS.md.

## Deployment order

When setting up a fresh agent:

1. Copy `openclaw.json` into `~/.openclaw/openclaw.json`
2. Copy `IDENTITY.md`, `SOUL.md`, `USER.md`, `AGENTS.md`, `TOOLS.md` into the workspace and fill in `{{...}}` placeholders
3. Copy `HEARTBEAT.md` and `MEMORY.md` — these can start mostly empty and grow
4. Copy `BOOTSTRAP.md` — the agent processes it on its first session, then it's renamed out
5. Verify: `openclaw health && openclaw doctor && openclaw models auth list`

See `skills/deploy-identity.md` for the full deploy workflow that consumes IDENTITY.md + SOUL.md + HEARTBEAT.md. MEMORY.md / AGENTS.md / USER.md / TOOLS.md / BOOTSTRAP.md don't have dedicated deploy skills — they're just filled in and dropped into the workspace.

## Which files load when

| File | Every turn? | Group chat? | Sub-agent? |
|------|-------------|-------------|------------|
| IDENTITY.md | ✅ | ✅ | ✅ |
| SOUL.md | ✅ | ✅ | ✅ |
| USER.md | ✅ | ✅ | ✅ |
| AGENTS.md | ✅ | ✅ | ✅ |
| TOOLS.md | ✅ | ✅ | ✅ |
| HEARTBEAT.md | per heartbeat | N/A | N/A |
| **MEMORY.md** | ✅ main session only | **❌ NEVER** | **❌ NEVER** |
| BOOTSTRAP.md | first session only | N/A | N/A |

AGENTS.md's session-type routing enforces the MEMORY.md gate.
