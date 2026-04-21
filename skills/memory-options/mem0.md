---
name: "mem0 (not recommended — see caveats below)"
category: "structured-memory"
repo: "https://github.com/mem0ai/mem0"
license: "Apache-2.0"
version_last_verified: "v1.0.x (stable); v2.0.0 breaking release April 2026"
last_checked: "2026-04-21"
maintenance_status: "active but flagged — see caveats"
openclaw_integration: "HTTP-bridge via official mem0 OpenClaw plugin (untested by us post-v2.0 breaking change)"
cost_tier: "extra-llm-calls"
privacy_tier: "hybrid"
requires:
  - "Python 3.11+ or Node 22+"
  - "Vector store (Qdrant/Chroma/pgvector/LanceDB)"
docs_urls:
  - "https://github.com/mem0ai/mem0"
  - "https://docs.mem0.ai/integrations/openclaw"
  - "https://github.com/mem0ai/mem0/tree/main/openclaw"
  - "https://arxiv.org/abs/2504.19413"
---

# mem0 — NOT RECOMMENDED as of 2026-04-21

> **Use [`graphiti-neo4j.md`](./graphiti-neo4j.md) instead for structured temporal memory.** mem0 has three material issues that came to light in April 2026 research (see caveats at the top of this file). This entry is kept for completeness and to inform migrations off mem0, not as a recommendation.

## Why not mem0 (be honest)

1. **v2.0.0 (April 16, 2026) went ADD-only.** No UPDATE or DELETE operations on extracted facts. Superseded facts accumulate indefinitely. For evolving client configs ("client X switched from weekly to biweekly syncs"), this is a staleness bomb.
2. **Open CVE GHSA-5gv3-2fv6-jvhx (Apr 17, 2026, CVSS 8.1)** — SQL/Cypher injection in PGVector/Neptune/Azure MySQL backends. Lower risk on default SQLite backend but signals project security hygiene.
3. **Weak at temporal reasoning.** mem0's own arxiv paper admits this. On LongMemEval temporal subtask: mem0 49.0% vs Zep/Graphiti 63.8%. "What did I configure for client X last month" is exactly the query mem0 loses.

If you're already committed to mem0 or have a specific use case (simple user-preference injection, not temporal reasoning), the install notes below remain accurate. Otherwise route to `graphiti-neo4j.md`.

---

## Legacy content (for reference / existing mem0 deployments)

- **Stats:** 53.6k stars. Apache 2.0.
- **Official OpenClaw integration:** [docs.mem0.ai/integrations/openclaw](https://docs.mem0.ai/integrations/openclaw) + [mem0ai/mem0/openclaw](https://github.com/mem0ai/mem0/tree/main/openclaw)

## Purpose

Universal memory layer for AI agents. Extracts entities and salient facts from conversation, stores in a multi-level hierarchy (user / session / agent), injects relevant memories back into prompts. Hybrid retrieval fuses semantic vector + BM25 keyword + entity-linking graph.

## When to pick mem0

- Need **structured semantic memory** (facts + entities + relationships) not opaque vector chunks
- Need **cross-session scoping** (user memory that persists vs. session memory that doesn't)
- Want **auto-recall** (inject before each run) and **auto-capture** (summarize after each run)

## When NOT to pick mem0

- One agent, one user, simple recall — built-in memory is fine
- Running another HTTP service (mem0 REST + vector DB) is operational overhead you don't need
- You need typed temporal edges ("X worked at Y from 2023-2025") → pick `graphiti-neo4j.md` instead

## Architecture

- Default embedding: OpenAI `text-embedding-3-small`
- Vector store: pluggable (Qdrant, Chroma, pgvector, LanceDB)
- Retrieval: hybrid (semantic + BM25 + entity-linking graph)
- Multi-level hierarchy: user / session / agent scoping via `run_id`

## Install (self-hosted path)

```bash
# Python
pip install mem0ai
# Or with NLP extras for entity extraction
pip install mem0ai[nlp]
python -m spacy download en_core_web_sm

# Node
npm install mem0ai

# OpenClaw plugin (community pattern — verify path on target)
mkdir -p ~/.openclaw/extensions/memory-mem0
# Copy the plugin content from mem0ai/mem0/openclaw
cd ~/.openclaw/extensions/memory-mem0 && npm install
```

⚠️ **Path uncertainty:** verify the `~/.openclaw/extensions/` directory convention against the target OpenClaw version. Official mem0 docs are authoritative.

## Wire into OpenClaw

`openclaw.json`:

```json
{
  "extensions": {
    "memory-mem0": {
      "baseUrl": "http://localhost:PORT",
      "userId": "<your-user-id>",
      "autoCapture": true,
      "autoRecall": true
    }
  }
}
```

## Tools the agent gains

- `memory_recall` — semantic + entity search
- `memory_store` — save new memory with metadata
- `memory_forget` — soft-delete or redact

Plus auto-recall (before each run) and auto-capture (after each run).

## Trade-offs

- ✅ Structured semantic memory vs opaque vector chunks
- ✅ Built-in cross-session scoping
- ✅ Official OpenClaw integration maintained by mem0 team
- ❌ Extra REST service + vector DB to run/monitor
- ❌ Fact extraction = extra LLM calls per conversation turn (cost)
- ❌ Community forks exist (`serenichron/openclaw-memory-mem0`, `tensakulabs/openclaw-mem0`, etc.) — pick the official path unless you know why

## Benchmark caveat

Mem0's README claims 91.6 on LoCoMo, 93.4 on LongMemEval, 64.1 on BEAM-1M ([arxiv 2504.19413](https://arxiv.org/abs/2504.19413)). Vendor-reported — verify independently before citing externally.

## Citations

- https://github.com/mem0ai/mem0
- https://docs.mem0.ai/integrations/openclaw
- https://mem0.ai/blog/add-persistent-memory-openclaw
- https://arxiv.org/abs/2504.19413
