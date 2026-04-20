---
name: "Lossless Claw"
category: "context-engine"
repo: "https://github.com/Martian-Engineering/lossless-claw"
license: "MIT"
version_last_verified: "v0.9.2"
last_checked: "2026-04-20"
maintenance_status: "active"
openclaw_integration: "contextEngine-slot-plugin"
cost_tier: "extra-llm-calls"
privacy_tier: "local-storage"
requires:
  - "OpenClaw 2026.4.x+ (contextEngine slot)"
docs_urls:
  - "https://github.com/Martian-Engineering/lossless-claw"
  - "https://x.com/jlehman_/status/2033319118343184725"
---

# Lossless Claw

**This is the highest-leverage fix for most OpenClaw memory pain.** If the agent is inventing facts it claims to "remember," install this before anything else.

- **Version (2026-04-20):** v0.9.2 (released 2026-04-19). 4.4k stars.

## Purpose

Replace OpenClaw's default "chop and forget" context compaction (sliding-window truncation + lossy summarization) with lossless context management (LCM). Every message is kept in a SQLite store; older chunks are summarized into a DAG of summary nodes; agents retrieve detail on demand via dedicated tools.

## When to pick this

- Agent is hallucinating facts from compacted sessions
- Multi-step procedures get flattened into lossy summaries
- You want to be able to retrieve the exact text of something said 500 turns ago
- Known compaction bugs ([#58137](https://github.com/openclaw/openclaw/issues/58137), [#65218](https://github.com/openclaw/openclaw/issues/65218)) are biting you

## Architecture

- SQLite for immutable message store (per-workspace `.lcm/*.sqlite`)
- DAG of LLM-generated summary nodes rebuilt during compaction
- Three tools exposed to the agent:
  - `lcm_grep` — keyword search across full history
  - `lcm_describe` — get a summary of a time range or topic
  - `lcm_expand` — recover the original verbatim message text behind a summary node

## Install

⚠️ **Command uncertainty:** README says `openclaw plugins install @martian-engineering/lossless-claw`. On OpenClaw 2026.4.15 (live 4C), the top-level `--help` does NOT list `plugins`. Verify on the target: `openclaw --help | grep -iE 'plugin|extension'`. If `plugins` isn't present, try `openclaw skills install` or manual install under `~/.openclaw/extensions/`.

```bash
# Production
openclaw plugins install @martian-engineering/lossless-claw

# Local dev
cd /path/to/lossless-claw
pnpm build
openclaw plugins install --link /path/to/lossless-claw
```

## Wire into OpenClaw

Configure the contextEngine slot in `~/.openclaw/openclaw.json`:

```json
{
  "plugins": {
    "slots": {
      "contextEngine": "lossless-claw"
    }
  }
}
```

Restart the gateway after.

## Verification

```bash
openclaw config get plugins.slots.contextEngine
# → "lossless-claw"

ls ~/.openclaw/workspace/.lcm/*.sqlite

# Ask the agent to lcm_grep for a phrase that happened many turns ago
```

## Trade-offs

- ✅ Agent "never forgets" — compacted history recoverable
- ✅ In-context tools (`lcm_grep`, `lcm_describe`, `lcm_expand`) are grep-able, testable
- ⚠️ Added latency: every compaction round costs LLM calls to build summary nodes (use local model per `community-patterns.md` heartbeat routing)
- ⚠️ Storage grows monotonically — plan retention on the SQLite db
- ⚠️ **Per-session context engine**, NOT global cross-session recall. For cross-session + cross-agent, still need OpenClaw's memory folder (or `mem0.md`/`graphiti-neo4j.md`)

## Known issues / gotchas

- Install path uncertainty (see Install section)
- Summary model defaults to primary — configure it to point at local oMLX/Ollama for $0 compaction cost

## Citations

- https://github.com/Martian-Engineering/lossless-claw
- https://x.com/jlehman_/status/2033319118343184725
