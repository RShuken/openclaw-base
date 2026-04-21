---
name: "Graphiti + Neo4j via local Flask REST (the validated pattern)"
category: "structured-memory"
repo: "https://github.com/getzep/graphiti"
license: "MIT (Graphiti); GPLv3 (Neo4j Community)"
version_last_verified: "graphiti-core 0.28.2, Neo4j 5.26, OpenClaw 2026.4.15"
last_checked: "2026-04-21"
maintenance_status: "active — validated end-to-end 2026-04-21"
openclaw_integration: "Flask REST on localhost:8100 + curl from exec tool (NOT MCP — see §integration)"
cost_tier: "local-only (optionally cloud embeddings for quality)"
privacy_tier: "fully-local-compute (only embeddings may egress, configurable)"
requires:
  - "Docker"
  - "Python 3.10+"
  - "16GB+ RAM"
  - "OpenClaw 2026.4.15+"
docs_urls:
  - "https://github.com/getzep/graphiti"
  - "https://neo4j.com/download/neo4j-community-edition/"
---

# Graphiti + Neo4j — the validated pattern (REST + curl, NOT MCP)

Self-hosted temporal knowledge graph on Apple Silicon. **Validated end-to-end on Mac Mini 4C on 2026-04-21.** This file reflects the pattern that actually works; earlier versions recommending MCP registration have been retired (see `09-mcp-agent-turn-gap-investigation.md` in any deployment's KNOWLEDGE/ directory for why).

## When to pick this

- Your agent needs **temporal queries** — "what did I decide about client X last month"
- You need **predicate-typed relationships** — "list all people who work at company Y"
- You're tracking **50+ evolving entities** (people, projects, decisions, configs)
- You want **sub-200ms structured lookups** that markdown traversal can't deliver
- You want the facts queryable as data, not just prose

## When NOT to pick this (or what to pick instead)

- Solo agent tracking <50 entities → semantic search over markdown (`openclaw memory search`) is probably enough
- You value "zero new processes" over structured recall → `lossless-claw.md` alone covers conversation fidelity
- You need a human-inspectable wiki more than machine-queryable facts → Obsidian vault over `workspace/memory/` may be a better fit

## Architecture

```
  Agent turn
     │ model emits exec tool_use (natively supported)
     ▼
  exec / shell → curl -X POST http://localhost:8100/api/episode -d '{...}'
     ▼
  Flask REST (graphiti-core wrapper, ~250 lines Python)
     │ asyncio + neo4j-driver
     ▼
  Neo4j 5.26 Docker container (1 GB heap cap)
```

Three code paths, one local service. The agent never sees MCP; it sees shell commands, which OpenClaw's exec surface routes correctly.

## Install recipe (validated)

### 1. Neo4j in Docker

```bash
docker pull neo4j:5.26
mkdir -p ~/.openclaw/workspace/memory/structured/neo4j/{data,logs}
NEO4J_PASS="$(openssl rand -base64 24 | tr -d '/+=')"
docker run -d \
  --name flyn-neo4j \
  --restart unless-stopped \
  -p 127.0.0.1:7474:7474 -p 127.0.0.1:7687:7687 \
  -v ~/.openclaw/workspace/memory/structured/neo4j/data:/data \
  -v ~/.openclaw/workspace/memory/structured/neo4j/logs:/logs \
  -e NEO4J_AUTH="neo4j/${NEO4J_PASS}" \
  -e NEO4J_server_memory_heap_initial__size=512m \
  -e NEO4J_server_memory_heap_max__size=1G \
  -e NEO4J_server_memory_pagecache_size=256m \
  neo4j:5.26
```

Ports bound to `127.0.0.1` only — loopback trust boundary. Steady-state footprint: ~830 MiB.

### 2. Python venv + graphiti-core

```bash
python3 -m venv ~/.openclaw/workspace/memory/structured/graphiti-venv
~/.openclaw/workspace/memory/structured/graphiti-venv/bin/pip install \
  "graphiti-core[google-genai]>=0.28.2" flask
```

The `[google-genai]` extra is REQUIRED for Gemini embedder. Default install does not include it.

### 3. Auth profile entries

Add to `~/.openclaw/agents/<agent-id>/agent/auth-profiles.json`:

```json
{
  "profiles": {
    "neo4j:default": {
      "type": "token",
      "provider": "neo4j",
      "user": "neo4j",
      "token": "<NEO4J_PASS from step 1>",
      "uri": "bolt://localhost:7687"
    },
    "gemini:default": { "type": "token", "provider": "gemini", "token": "<gemini_api_key>" },
    "google:default": { "type": "token", "provider": "google", "token": "<gemini_api_key>" },
    "ollama:default": { "type": "token", "provider": "ollama", "token": "local" }
  }
}
```

**Both `gemini:default` AND `google:default` are required** — OpenClaw's embedding provider ID is `gemini` but the runtime auth lookup hits provider ID `google`. Same key, two profile entries.

### 4. Flask REST wrapper

A ~250-line Python file exposes these endpoints, calling Graphiti's async API:

- `GET /api/health` — liveness + Neo4j connectivity
- `POST /api/episode` body `{body, name?, source?, valid_at?}` — ingest prose; Graphiti extracts typed edges
- `GET /api/search?q=...` — semantic + graph search
- `GET /api/temporal?q=...&from=ISO&to=ISO` — temporal-filtered facts
- `GET /api/episodes?limit=N` — raw episode list

Reference implementation: `flyn-agent/deploy/kg/flyn-graphiti-api.py` on the deployment host. Key implementation notes:
- **LLM client:** `OpenAIGenericClient` pointing at `http://localhost:11434/v1` with model `gemma4:e4b` (local)
- **Embedder:** `GeminiEmbedder` with `embedding_model="gemini-embedding-001"` (stable)
- **Reranker:** `OpenAIRerankerClient` with the same Ollama config (else default tries OpenAI API)
- **Python timeout:** `run_async(coro, timeout=600)` — local entity extraction takes 30-120s per episode
- **JSON serialization:** Neo4j `DateTime` needs a `_coerce()` helper using `.isoformat()` — Flask's default serializer rejects it

### 5. launchd plist for auto-start

```xml
<!-- ~/Library/LaunchAgents/ai.flyn.graphiti-api.plist -->
<plist version="1.0"><dict>
  <key>Label</key><string>ai.flyn.graphiti-api</string>
  <key>ProgramArguments</key><array>
    <string>/Users/<user>/.openclaw/workspace/memory/structured/graphiti-venv/bin/python</string>
    <string>/Users/<user>/.openclaw/workspace/kg/flyn-graphiti-api.py</string>
  </array>
  <key>EnvironmentVariables</key><dict>
    <key>HOME</key><string>/Users/<user></string>
    <key>PATH</key><string>/Users/<user>/.openclaw/workspace/memory/structured/graphiti-venv/bin:/opt/homebrew/bin:/usr/bin:/bin</string>
  </dict>
  <key>RunAtLoad</key><true/>
  <key>KeepAlive</key><dict><key>Crashed</key><true/></dict>
  <key>ThrottleInterval</key><integer>30</integer>
  <key>StandardOutPath</key><string>/tmp/flyn-graphiti-api.log</string>
  <key>StandardErrorPath</key><string>/tmp/flyn-graphiti-api.err</string>
</dict></plist>
```

Load: `launchctl bootstrap gui/$(id -u) ~/Library/LaunchAgents/ai.flyn.graphiti-api.plist`.

## How the agent calls it (THIS IS THE PATTERN)

Agent's system prompt / AGENTS.md includes curl patterns. The LLM emits exec tool_use; OpenClaw runs the bash; REST returns JSON.

```bash
# Ingest — prose; Graphiti auto-extracts typed entity edges
curl -sS -X POST http://localhost:8100/api/episode \
  -H 'Content-Type: application/json' \
  -d '{"body": "Ryan approved the Railway cost cap increase on 2026-04-21", "name": "railway-cap-bump"}'

# Semantic + graph search — returns edges with valid_at/invalid_at
curl -sS 'http://localhost:8100/api/search?q=Railway+cost'

# Temporal filter
curl -sS 'http://localhost:8100/api/temporal?q=cora&from=2026-04-01&to=2026-04-30'
```

**Why NOT OpenClaw MCP registration** — we spent hours trying `mcp.servers.*`, `plugins.entries.acpx.config.mcpServers`, and `@aiwerk/openclaw-mcp-bridge`. All six paths resulted in the agent hallucinating tool calls (confident "Added" text, zero actual invocation, Neo4j untouched). `openclaw agent` on 2026.4.15 does not surface MCP tools to the model's per-turn tool list. Edge (the production reference deployment) uses the same REST + curl pattern for this exact reason. Do not chase MCP for this integration on 2026.4.15; revisit only if upstream fixes it.

## Trade-offs

- ✅ Validated end-to-end 2026-04-21. Proven working pattern.
- ✅ Sub-200ms structured queries via Neo4j indices.
- ✅ Temporal queries out of the box (`valid_at` / `invalid_at` on every edge).
- ✅ Schema evolution via Pydantic ontology — add a relationship type, reindex, queryable.
- ✅ Entity extraction routes to local gemma4:e4b → $0 per ingest.
- ❌ Entity extraction blocks 30-120s per episode (not a problem for heartbeat/cron; noticeable in agent turns).
- ❌ Neo4j in Docker adds ~830 MiB RAM + 1 GB heap cap.
- ❌ Two DBs to back up (Neo4j volumes + SQLite for Lossless Claw).
- ❌ Embedding calls hit Gemini cloud by default (can swap to local EmbeddingGemma with quality tradeoff).

## Graph schema (starting template — adapt to use case)

**Graphiti auto-infers entity types during ingestion**, so you don't need to define them up front. But you CAN provide typed entity hints via the MCP server's `entity_types` config or by passing classes to the Python API. Recommended starter types for a solo-operator agent:

- Preference (user preferences / opinions)
- Requirement (must-have functionality)
- Procedure (SOPs, sequences)
- Location (physical or virtual)
- Event (time-bound)
- Organization (companies, institutions)
- Document (books, articles, reports)
- Topic (subject domain — use sparingly)
- Object (items, tools — use sparingly)

Plus domain-specific types you add as ontology grows: `Project`, `Deployment`, `Decision`, `Incident`, etc.

## Citations

- https://github.com/getzep/graphiti
- https://github.com/Martian-Engineering/lossless-claw (context-engine companion)
- https://neo4j.com/download/neo4j-community-edition/
- Validated deployment: github.com/RShuken/flyn-agent (private) — see `deploy/install-flyn.sh` and `deploy/kg/flyn-graphiti-api.py` for reference implementation
- Investigation of the MCP path that doesn't work: flyn-agent `KNOWLEDGE/09-mcp-agent-turn-gap-investigation.md` and `POSTMORTEM-2026-04-21.md`
