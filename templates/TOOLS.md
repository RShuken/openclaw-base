# TOOLS (template)

What tools the agent has available and how to use them. Loaded every turn as system context.

Replace every `{{...}}` before deploying. Target: 50-150 lines depending on toolset. If this grows past ~200 lines, split specialized tooling into `workspace/tools-<area>.md` files that the agent loads on demand.

---

## Built-in OpenClaw capabilities

Available by default on any 2026.4.15+ install. Verified by `openclaw skills list`.

- **Memory** — `openclaw memory search`, `openclaw memory index`. See `skills/deploy-memory.md` + `skills/deploy-memory-advanced.md`.
- **Cron** — `openclaw cron add/list/edit/rm/runs/status`. Native scheduling; don't use platform crontab as a first choice.
- **Channels** — `openclaw channels send/list`. See `skills/channels/` for per-channel detail.
- **Agent delegation** — `openclaw agent --agent <id> -m "..."` for spawning focused sub-agent runs.
- **Models** — `openclaw models status/list/auth`. See `skills/model-providers/` (future) or `templates/openclaw.json`.

## Platform integrations (installed in this workspace)

List only what's actually installed. Missing sections = that integration isn't deployed yet.

### Email

- **Provider:** {{e.g. "Gmail via @steipete/gog" / "Himalaya via App Password"}}
- **CLI:** {{e.g. "gog gmail search ..."}}
- **Send rule:** **requires explicit approval per AGENTS.md approval gates**

### Calendar

- **Provider:** {{e.g. "Google Calendar via gog"}}
- **CLI:** {{e.g. "gog calendar list --today"}}

### Messaging

- **Primary channel:** {{e.g. "Telegram bot @FlynBot"}}
- **Group ID:** {{CHAT_ID_PLACEHOLDER}}
- **Topic routing:** see `workspace/tools/telegram-topics.md`
- See `skills/channels/telegram.md` for channel config

### File storage / documents

- **Provider:** {{e.g. "Google Drive via gog / local filesystem / 1Password"}}
- **CLI:** {{e.g. "gog drive files list"}}

### Structured memory (optional)

- **Backend:** {{none | mem0 | graphiti+neo4j | zep}}
- **How to query:** {{e.g. "memory_recall tool / Cypher via neo4j-driver"}}

### Knowledge base (optional)

- **Installed:** {{yes | no}}
- **Ingest:** {{e.g. "drop URL into Telegram KB topic"}}
- **Query:** {{e.g. "memory search or kb search"}}

## Skills registered in this workspace

List only skills actually installed + configured. Don't copy the whole library list.

| Skill | Purpose | Trigger |
|-------|---------|---------|
| {{deploy-identity}} | Agent's persona | N/A (always on) |
| {{deploy-memory}} | Cross-session memory | N/A (always on) |
| {{add more as installed}} | | |

## Per-skill model overrides

If any skills in this workspace route their LLM calls through a specific model via env var (see `skills/_authoring/_deploy-common.md` "Per-skill env-var overrides"), document them here.

| Skill | Env var | Current value |
|-------|---------|---------------|
| {{e.g. deploy-urgent-email}} | `URGENT_EMAIL_MODEL` | `openai-codex/gpt-5.4-nano` |
| {{e.g. deploy-advisory-council}} | `COUNCIL_PERSONA_MODEL` | `ollama/gemma4:e4b` |

## How to pick the right tool

- **Quick lookup of known information** → memory search first (cheap, fast)
- **Web-fresh information** → search skill (Tavily / brave) — flag to operator that web query is happening
- **External action (email, post, deal modification)** → AGENTS.md approval gate
- **Scheduled recurring work** → `openclaw cron add`, not inline
- **Background fact-extraction** → heartbeat auto-save (`skills/memory-options/community-patterns.md`)

## Anti-patterns

- Don't use channels to send to the operator what the operator can see in-session
- Don't spawn sub-agents for work the main session can do quickly
- Don't call LLMs in a loop for things a structured-memory query can answer
- Don't bypass approval gates even when the owner "probably" wants the action — ask
