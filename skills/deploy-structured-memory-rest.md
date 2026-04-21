# Deploy Structured Memory — REST + curl pattern (the working one)

## Compatibility

- **OpenClaw Version:** 2026.4.15+
- **Status:** WORKING — validated end-to-end on Mac Mini 4C 2026-04-21
- **Platforms:** macOS (Apple Silicon preferred); adaptable to Linux with minor launchd → systemd swap
- **Relationship to other skills:** complements `memory-options/lossless-claw.md` (conversation fidelity) and `memory-options/graphiti-neo4j.md` (backend choice); supersedes the MCP-registration approach described in earlier drafts of `deploy-memory-advanced.md`.

## Purpose

Give a long-running OpenClaw agent access to a typed, temporal knowledge graph — without relying on OpenClaw's MCP-to-agent-turn integration, which has a known surfacing gap on 2026.4.15 that causes agents to hallucinate tool calls. The working pattern: wrap the structured-memory backend in a local Flask REST service and have the agent reach it via `curl` from the built-in exec shell tool, which OpenClaw surfaces correctly.

## Architecture

```
  ┌─────────────────────────────────────┐
  │  Agent turn  (Codex or Claude)      │
  │  emits exec tool_use (natively      │
  │  supported — NOT MCP)               │
  └────────┬────────────────────────────┘
           │ bash: curl -X POST http://localhost:8100/api/episode -d '{...}'
           ▼
  ┌─────────────────────────────────────┐
  │  Flask REST wrapper  (localhost:8100)│
  │  ~250 lines Python; 5 endpoints      │
  │  launchd agent, auto-start, restart  │
  └────────┬────────────────────────────┘
           │ graphiti-core async API
           ▼
  ┌─────────────────────────────────────┐
  │  Graphiti + Neo4j backend (Docker)   │
  │  entity extraction: local Ollama     │
  │  embeddings: Gemini / local fallback │
  └─────────────────────────────────────┘
```

## Why this pattern and not MCP

We tried six MCP registration paths on OpenClaw 2026.4.15:

1. `openclaw mcp set <name> <json>` (global `mcp.servers.*`)
2. `plugins.entries.acpx.config.mcpServers.*` + ACPX enabled
3. `agents.defaults.embeddedHarness.runtime = codex`
4. `openclaw agent --local` (embedded harness)
5. Primary model swap Codex → Claude Sonnet 4.6
6. Community plugin `@aiwerk/openclaw-mcp-bridge` v0.13.5

All six resulted in the agent hallucinating the MCP tool call — confident "Added" text with no MCP trace in the gateway log and zero state change in the backend. The exec / shell tool, by contrast, surfaces reliably and emits real tool_use blocks the gateway routes correctly. `curl` is just a shell command.

The production reference deployment (the "Edge" agent in the Crusoe-Ventures consulting engagement, on the same OpenClaw version) uses the same REST + curl pattern. That convergence is the strongest signal that MCP-for-structured-memory is not the right abstraction at this OpenClaw version.

## Reference implementation

See `flyn-agent` (RShuken/flyn-agent, private) for a complete working deployment:

- `deploy/kg/flyn-graphiti-api.py` — the Flask wrapper
- `deploy/launchd/ai.flyn.graphiti-api.plist.template` — launchd auto-start
- `deploy/install-flyn.sh` — idempotent install script
- `workspace/AGENTS.md` — how the agent is instructed to use the REST API via curl
- `KNOWLEDGE/08-graphiti-neo4j-recipe.md` — full install recipe with gotchas
- `KNOWLEDGE/09-mcp-agent-turn-gap-investigation.md` — the MCP investigation
- `KNOWLEDGE/10-rest-pattern-the-working-fix.md` — the pattern summarized
- `POSTMORTEM-2026-04-21.md` — full narrative

## Deploy steps (summary; see `memory-options/graphiti-neo4j.md` for full commands)

1. **Neo4j in Docker** with 1 GB heap cap (`-e NEO4J_server_memory_heap_max__size=1G`), loopback-only port bindings (`127.0.0.1:7474:7474`), persistent volumes under workspace.
2. **Python venv + `graphiti-core[google-genai]>=0.28.2` + `flask`** — the `[google-genai]` extra is required for the Gemini embedder.
3. **Auth profiles** in `auth-profiles.json`: `neo4j:default`, `gemini:default` + `google:default` (same key, both profile IDs — runtime auth lookup quirk), `ollama:default` with `token: "local"`.
4. **Flask REST wrapper** with 5 endpoints (`/api/health`, `POST /api/episode`, `/api/search`, `/api/temporal`, `/api/episodes`). Uses `OpenAIGenericClient` pointing at `http://localhost:11434/v1` for LLM entity extraction, `GeminiEmbedder` for embeddings, `OpenAIRerankerClient` with same Ollama config (else it defaults to OpenAI API — a common boot-time trap).
5. **launchd plist** at `~/Library/LaunchAgents/ai.flyn.graphiti-api.plist` with explicit `HOME` + `PATH` env vars (inherited shell env isn't enough).
6. **Agent instructions** in `workspace/AGENTS.md` show concrete curl patterns the agent should generate via exec.

## Operational characteristics

| Dimension | Value |
|-----------|-------|
| Memory footprint | Neo4j ~830 MiB steady, Flask ~100 MiB, Gemma4 transient ~11 GB (auto-unload) |
| Ingest latency | 30-120 s per episode (local gemma4 entity extraction pipeline) |
| Query latency | <100 ms (Neo4j indexed lookups) |
| Cost per ingest | $0 (local Gemma) + ~$0.0001 per Gemini embedding call |
| Cost per query | ~$0.0001 (one embedding) |
| Availability | Auto-start at boot via launchd; auto-restart on crash with 30s throttle |

## Agent-side instructions (AGENTS.md snippet)

```markdown
## Structured memory — local Graphiti REST at localhost:8100

Reach via curl from the exec shell tool (NOT via MCP).

### Write
curl -sS -X POST http://localhost:8100/api/episode \
  -H 'Content-Type: application/json' \
  -d '{"body": "<prose fact>", "name": "<short-slug>"}'

### Search (typed facts with valid_at/invalid_at)
curl -sS 'http://localhost:8100/api/search?q=<query>'

### Temporal filter
curl -sS 'http://localhost:8100/api/temporal?q=<query>&from=YYYY-MM-DD&to=YYYY-MM-DD'

### When to write vs rely on Hot-tier MEMORY.md
Write to Graphiti for: decisions, config changes, client/project updates,
learned facts that should survive across sessions. Use MEMORY.md only for
facts that ALSO need to be in-context every turn (pinned operator preferences,
standing rules).
```

## Install order that works (enforced by reference script)

1. OpenClaw 2026.4.15 + tarball Node 22 LTS (NOT Homebrew Node 25 — see `postmortem_ian_ferguson`).
2. Ollama + `gemma4:e4b` pull.
3. Auth profiles: `ollama:default`, `gemini:default` + `google:default`, (later) `neo4j:default`.
4. Neo4j Docker container.
5. Python venv + graphiti-core install.
6. Flask wrapper + launchd plist + service start.
7. Lossless Claw plugin (context engine — separate from structured memory).
8. OpenClaw config: `agents.defaults.heartbeat.model`, `agents.defaults.models.<ollama/gemma4:e4b>`, `agents.defaults.memorySearch.{provider,fallback,model}`.
9. Workspace `*.md` files → `~/.openclaw/workspace/`.
10. Gateway restart.

## Hygiene

- `group_id` scoped per-agent. Don't ingest into another agent's group.
- One episode = one coherent fact/event. Whole conversation dumps dilute retrieval quality.
- Include explicit dates in the prose when you know them — Graphiti auto-infers `valid_at` from date strings.
- Never embed secrets in openclaw.json config. Launcher wrapper reads from `auth-profiles.json` at runtime.

## Anti-patterns

- Don't register the Graphiti backend as an MCP server and expect agent turns to use it (see six-path investigation).
- Don't replace openclaw.json wholesale — other tools' auth lives there. Use `openclaw config set` for additive changes.
- Don't use `openclaw capability model run --model X` as a routing probe — the `--model` flag doesn't reliably bind; it's advisory, not authoritative. Check actual routing via the backend service logs (Ollama log / gateway `model_fallback_decision` entries).
- Don't swap primary to Claude to "fix" MCP tool surfacing — confirmed 2026-04-21 that primary model is not the cause. MCP surfacing is upstream of the model.

## Known gotchas (full list in `KNOWLEDGE/` directory of reference deploy)

- `gemma3:4b` rejects tool-carrying inference schemas with HTTP 400 → falls back to cloud. Use `gemma4:e4b` instead.
- Ollama needs an `ollama:default` auth profile even though it's local — OpenClaw errors 401 otherwise.
- Neo4j DateTime isn't JSON-serializable by Flask default — wrapper needs `.isoformat()` coercion.
- `graphiti-core`'s default cross-encoder uses OpenAI API — override with `OpenAIRerankerClient(config=ollama_llm_config)` to keep it local.
- Python `run_async` timeout of 120s is too short for local entity extraction — bump to 600s.

## Dependencies

- Referenced by: `deploy-memory-advanced.md` (router), `memory-options/graphiti-neo4j.md` (backend detail)
- Complements: `memory-options/lossless-claw.md` (contextEngine slot — conversation fidelity layer; different concern)
- Supersedes (within this library): the MCP-registration pattern that earlier drafts of `deploy-memory-advanced.md` recommended
