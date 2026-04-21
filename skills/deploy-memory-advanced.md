# Deploy Advanced Memory — Router + Recommendations

## Compatibility

- **OpenClaw Version**: 2026.4.15+
- **Builds on**: [`deploy-memory.md`](./deploy-memory.md) (the built-in subsystem must be deployed first — this skill extends or replaces it)
- **Status**: WORKING — validated end-to-end 2026-04-21 on Apple Silicon, 16 GB RAM
- **Structure**: this file is a lean routing document. Per-option details live in [`memory-options/`](./memory-options/). Integration pattern for structured memory lives in [`deploy-structured-memory-rest.md`](./deploy-structured-memory-rest.md).

## Recommended stack (as of 2026-04-21)

For a solo operator agent on Apple Silicon, the **validated working stack** is:

| Layer | Component | Where detailed |
|-------|-----------|----------------|
| Context engine (conversation fidelity) | **Lossless Claw plugin** | [`memory-options/lossless-claw.md`](./memory-options/lossless-claw.md) |
| Semantic recall over markdown | Built-in `sqlite-vec` + Gemini embeddings | [`deploy-memory.md`](./deploy-memory.md) + [`memory-options/gemini-embeddings.md`](./memory-options/gemini-embeddings.md) |
| Local embedding fallback | Built-in `embeddinggemma` GGUF | [`memory-options/embeddinggemma.md`](./memory-options/embeddinggemma.md) |
| Background chat / heartbeat model | **Gemma 4 `gemma4:e4b`** via Ollama | [`memory-options/gemma4-heartbeat.md`](./memory-options/gemma4-heartbeat.md) |
| Structured temporal KG | **Graphiti + Neo4j via Flask REST** | [`memory-options/graphiti-neo4j.md`](./memory-options/graphiti-neo4j.md) + [`deploy-structured-memory-rest.md`](./deploy-structured-memory-rest.md) |
| Agent access pattern | `curl` from exec shell tool (NOT MCP) | [`deploy-structured-memory-rest.md`](./deploy-structured-memory-rest.md) |
| Tiering | Hot / Warm / Cold markdown lifecycle | [`memory-options/community-patterns.md`](./memory-options/community-patterns.md) |

**NOT recommended for new deploys (despite still being in the options matrix):** `mem0` — see [`memory-options/mem0.md`](./memory-options/mem0.md) for the three April-2026 reasons (v2.0 breaking, open CVE, weak temporal).

## Purpose

Document the memory architecture options beyond OpenClaw's built-in subsystem. When the default sqlite-vec + cloud-embedding setup hits a limit — compaction hallucinations, quota exhaustion, privacy/offline needs, or you want structured entity-aware recall — this router maps the problem to the right option file.

**When to use:** diagnose a memory pain point here, jump to the specific option file for install detail. Don't read every option.

## How to read this

1. Identify your problem in the **Known failure modes** table below
2. Find the right option using the **Options matrix** or **Decision tree**
3. Jump to the linked file in [`memory-options/`](./memory-options/) for install/verification/tradeoffs
4. For full pre-composed stacks, use a template from [`memory-options/`](./memory-options/) (currently `enterprise-v2-stack-template.md` for the reference pattern)

## Known OpenClaw memory failure modes (2026)

| Issue | GitHub | Symptom | Fix |
|-------|--------|---------|-----|
| `heartbeat.model` / `compaction.model` override ignored | [#58137](https://github.com/openclaw/openclaw/issues/58137) | Forces background calls back to primary model | [`memory-options/community-patterns.md`](./memory-options/community-patterns.md) — call local model explicitly via `--model` flag |
| Compaction-reserve precheck forces 16384-token reserve | [#65218](https://github.com/openclaw/openclaw/issues/65218) | Compaction fires too early | [`memory-options/lossless-claw.md`](./memory-options/lossless-claw.md) (sidesteps compaction issues via context engine slot) |
| Dreaming pollutes daily files | [#66947](https://github.com/openclaw/openclaw/issues/66947) | Heartbeat skips logging because dreaming wrote file first | Disable dreaming OR hardcode heartbeat to always append |
| Pre-compaction flush overwrites | [#44787](https://github.com/openclaw/openclaw/issues/44787) | Daily memory files overwritten instead of appended | [`memory-options/lossless-claw.md`](./memory-options/lossless-claw.md) |
| Compaction → silent hallucinations | baseline §K.3 | Procedures flattened into lossy summaries that agent treats as ground truth | [`memory-options/lossless-claw.md`](./memory-options/lossless-claw.md) — **single biggest fix** |
| Memory folder ≥400 files → quality collapse | baseline §K.352 | Retrieval quality degrades | [`memory-options/community-patterns.md`](./memory-options/community-patterns.md) — weekly rollup + Hot/Warm/Cold tiering |
| Heartbeat prompt-injection persistence | [arxiv 2603.23064v2](https://arxiv.org/html/2603.23064v2) | Polluted heartbeat content survives compaction (75.6% attack success) | Isolate heartbeat sources from untrusted web content |

## Options matrix

| Option | File | Category | Cost | Privacy | Adds to / Replaces |
|--------|------|----------|------|---------|---------------------|
| Built-in (default) | [`deploy-memory.md`](./deploy-memory.md) | baseline | $ cloud | cloud | — |
| Lossless Claw | [`memory-options/lossless-claw.md`](./memory-options/lossless-claw.md) | context-engine | extra LLM calls | local storage | **Adds** — contextEngine slot |
| ~~mem0~~ | [`memory-options/mem0.md`](./memory-options/mem0.md) | structured-memory | extra LLM + vector DB | hybrid | ⚠️ **NOT recommended** for new deploys (see file for v2.0 breaking + CVE + temporal weakness) |
| Ollama embeddings | [`memory-options/ollama-embeddings.md`](./memory-options/ollama-embeddings.md) | embedding-local | $0 | local | Swaps provider |
| oMLX (Apple Silicon) | [`memory-options/omlx-apple-silicon.md`](./memory-options/omlx-apple-silicon.md) | embedding-local + chat | $0 | local | Swaps provider (AS only) |
| EmbeddingGemma | [`memory-options/embeddinggemma.md`](./memory-options/embeddinggemma.md) | embedding-local | $0 | local | Swaps provider |
| Gemini cloud (v1 + v2 preview) | [`memory-options/gemini-embeddings.md`](./memory-options/gemini-embeddings.md) | embedding-cloud | per-token | cloud | Swaps provider |
| OpenAI cloud | [`memory-options/openai-embeddings.md`](./memory-options/openai-embeddings.md) | embedding-cloud | per-token | cloud | Swaps provider |
| **Graphiti + Neo4j (REST)** | [`memory-options/graphiti-neo4j.md`](./memory-options/graphiti-neo4j.md) + [`deploy-structured-memory-rest.md`](./deploy-structured-memory-rest.md) | structured-memory | Docker + Python | local | Adds structured memory (**recommended**) |
| Zep (hosted) | [`memory-options/zep-hosted.md`](./memory-options/zep-hosted.md) | structured-memory | $ service | cloud | Managed-service alt to self-hosted Graphiti |
| Letta (MemGPT) | [`memory-options/letta-memgpt.md`](./memory-options/letta-memgpt.md) | structured-memory | LLM + vector | local | Replaces agent harness (wrong shape for augmenting existing OpenClaw agent) |
| LanceDB / Chroma / Qdrant / Weaviate | [`memory-options/vector-dbs-alternatives.md`](./memory-options/vector-dbs-alternatives.md) | vector-db | self-hosted | local | Replaces sqlite-vec |
| **Gemma 4 (`gemma4:e4b` heartbeat)** | [`memory-options/gemma4-heartbeat.md`](./memory-options/gemma4-heartbeat.md) | chat-heartbeat | $0 | local | Adds local inference (**recommended**; gemma3:4b fails tool calls) |
| Community patterns | [`memory-options/community-patterns.md`](./memory-options/community-patterns.md) | pattern | — | — | Additive |
| Enterprise Reference Stack template | [`memory-options/enterprise-v2-stack-template.md`](./memory-options/enterprise-v2-stack-template.md) | template | see constituents | local | Pre-composed stack |

## Decision tree

From the [`deploy-memory.md`](./deploy-memory.md) baseline (built-in memory + sqlite-vec + Gemini `gemini-embedding-001`):

- **Agent hallucinating compacted facts as real** → [`lossless-claw.md`](./memory-options/lossless-claw.md). Single biggest leverage point. Install first.
- **Zero outbound network required** → swap embedding provider to [`embeddinggemma.md`](./memory-options/embeddinggemma.md) or [`ollama-embeddings.md`](./memory-options/ollama-embeddings.md)'s `nomic-embed-text`. Keep sqlite-vec. Apple Silicon? Use [`omlx-apple-silicon.md`](./memory-options/omlx-apple-silicon.md) instead of Ollama.
- **Cost-sensitive but cloud OK** → [`openai-embeddings.md`](./memory-options/openai-embeddings.md) `text-embedding-3-small` at $0.02/M tokens.
- **Best raw retrieval quality regardless of cost** → [`gemini-embeddings.md`](./memory-options/gemini-embeddings.md) `gemini-embedding-2-preview` (multimodal, #1 MTEB, preview status) OR `text-embedding-3-large` (stable).
- **Need to embed images / video / audio / PDFs in same space as text** → `gemini-embedding-2-preview` — currently the only mainstream multimodal embedding option.
- **Hit Gemini free quota on a 24/7 agent** → [`openai-embeddings.md`](./memory-options/openai-embeddings.md) `text-embedding-3-small` (pennies/month) OR local (`omlx-apple-silicon.md`/`ollama-embeddings.md`).
- **Privacy / air-gap hard requirement** → local embedding + local vector DB ([`vector-dbs-alternatives.md`](./memory-options/vector-dbs-alternatives.md) LanceDB or Chroma). No third-party network.
- **24/7 always-on, minimize $** → local everything ([`omlx-apple-silicon.md`](./memory-options/omlx-apple-silicon.md) or [`ollama-embeddings.md`](./memory-options/ollama-embeddings.md)) for background; cloud frontier model only for user-chat turns. Caveat: blocked on #58137 / #65218 — verify.
- **Want structured semantic memory (entities, relationships, provenance)** → pick ONE of:
  - [`mem0.md`](./memory-options/mem0.md) — simplest, largest community, official OpenClaw integration
  - [`graphiti-neo4j.md`](./memory-options/graphiti-neo4j.md) — self-hosted temporal graph, when relationships have time validity (the reference snapshot's choice)
  - [`letta-memgpt.md`](./memory-options/letta-memgpt.md) — self-editing memory blocks as primary pattern (replaces OpenClaw harness)
  - [`zep-hosted.md`](./memory-options/zep-hosted.md) — managed version of temporal graph
- **Index larger than RAM** → [`vector-dbs-alternatives.md`](./memory-options/vector-dbs-alternatives.md) → LanceDB
- **Need rich filter predicates** → [`vector-dbs-alternatives.md`](./memory-options/vector-dbs-alternatives.md) → Qdrant
- **Heartbeats need a local chat/tool model** → [`gemma4-heartbeat.md`](./memory-options/gemma4-heartbeat.md) — recommended default; `qwen3:8b` as alternative

## Stack templates

### Default stack (most personal deployments)

For personal single-user agents and first-time installs:

1. [`lossless-claw.md`](./memory-options/lossless-claw.md) in the `contextEngine` slot
2. Built-in sqlite-vec for workspace memory
3. Embedding provider — pick one:
   - [`gemini-embeddings.md`](./memory-options/gemini-embeddings.md) `gemini-embedding-2-preview` (multimodal, top quality, preview caveat)
   - [`openai-embeddings.md`](./memory-options/openai-embeddings.md) `text-embedding-3-small` (cheapest stable cloud)
   - [`embeddinggemma.md`](./memory-options/embeddinggemma.md) (fully local)
4. Heartbeat model — [`gemma4-heartbeat.md`](./memory-options/gemma4-heartbeat.md)
5. [`community-patterns.md`](./memory-options/community-patterns.md) (weekly rollup + Hot/Warm/Cold + MEMORY.md security gate + heartbeat auto-save)
6. Add [`mem0.md`](./memory-options/mem0.md) ONLY if structured entity memory is a real requirement

Good fit for: Flyn on 4C, most personal deployments, anyone without relationship-history-heavy use cases.

### Enterprise Reference Stack (Apple Silicon, privacy-first, relationship-heavy)

Full pre-composed stack: [`memory-options/enterprise-v2-stack-template.md`](./memory-options/enterprise-v2-stack-template.md)

Layer summary:
1. oMLX substrate (Apple Silicon)
2. Gemma 4 (or Qwen 3 8B on pre-Gemma-4 deployments) as background chat
3. `mxbai-embed-large` via oMLX for embeddings
4. sqlite-vec vector storage
5. Lossless Claw as context engine
6. Graphiti + Neo4j for temporal relationship memory
7. Hot/Warm/Cold memory tiering
8. Heartbeat memory auto-save via local model

Full install walkthrough in the template file.

Good fit for: Apple Silicon hosts; privacy-first deployments; relationship-heavy organizations (VC firms, executive offices); 24/7 agents where $0 background-inference cost matters.

## For audit / update-checker agents

See [`memory-options/README.md`](./memory-options/README.md) "For audit / update-checker agents" section. Every option file in that directory has YAML frontmatter with `repo`, `license`, `version_last_verified`, `last_checked`, `maintenance_status`. An audit agent parses those to check staleness without loading the full option body.

## Dependencies

- **Depends on**: [`deploy-memory.md`](./deploy-memory.md)
- **Enhanced by**: [`deploy-knowledge-base.md`](./deploy-knowledge-base.md) (KB is separate RAG but shares embedding infra)
- **Required by**: None directly — optional extension skill
